---
name: classify-pr
description: Classify a pull request as Low/Medium/High criticality per this repo's PR-classification decision tree (README.md) and domain glossary (CONTEXT.md), then report reasoning, post a PR comment, apply a criticality label, and exit with a matching status code.
disable-model-invocation: true
---

# Classify PR

Classifies a single PR against the decision tree in [`README.md`](../../../README.md), using the canonical terms defined in [`CONTEXT.md`](../../../CONTEXT.md) and the rationale in [`docs/adr/0001-agent-judgment-for-quality-criteria.md`](../../../docs/adr/0001-agent-judgment-for-quality-criteria.md). Read those three files first; this skill only describes the procedure, not the definitions (which may evolve independently).

## Invocation

`/classify-pr [<PR number or URL>]`

If no argument is given, resolve the PR from the current branch (`gh pr view --json number,url`). If none is found, ask the user which PR to classify.

## Procedure

### 1. Gather the PR

- `gh pr view <n> --json number,title,body,url,files,additions,deletions,baseRefName,headRefName`
- `gh pr diff <n>` for the full diff (per-file hunks, not just a stat summary — categorization and Regression detection both require line-level detail)
- Read the title/body/linked issues for narrative context only (do **not** use stated intent to decide Regression — see step 3).

### 2. Categorize every changed file

For each file in the diff, assign exactly one category, per `CONTEXT.md`:

- **Documentation**: the file is a prose file (`*.md`, `docs/**`), OR every changed line in the file is a docstring line (JSDoc `/** */` in `.ts`, XML `///` doc comments in `.cs`, treating this as a seed list — extend to other doc-comment styles if this repo's stack grows). Any non-docstring line changed in the file disqualifies it from Documentation.
- **Test Code**: the file's path matches a test convention (`*.test.ts`, `*.spec.ts`, `*Tests.cs`, under `__tests__/` or `test/`) — seed list, extend as the repo's conventions grow.
- **Global Config Change**: the file's effect alters behavior across multiple services/environments without a code change of its own — shared env defaults, feature-flag defaults, CI workflow files, infra-as-code, dependency/lockfile bumps. Per-module local config does not count.
- **Critical Path**: the file matches a human-maintained sensitive-paths list (look for `config/sensitive-paths.yml` or similar at the repo root; if absent, treat the list as empty and rely on heuristics only) OR the diff matches heuristics: destructive SQL keywords (`DROP`, `DELETE`, `TRUNCATE`), common PII field names (e.g. `ssn`, `email`, `dob`, `address`, `personal_data`), or secrets-like patterns (API keys, tokens, credentials). This also covers secrets-leakage and GDPR concerns — there is no separate check for those.
- **Production Code**: anything left over that ships to and executes in the deployed runtime artifact.

A file can only be Global Config Change or Critical Path in addition to its base category (Production/Test); Documentation is mutually exclusive with everything else.

### 3. Walk the decision tree

Using the categorization, follow `README.md`'s tree to a structural outcome of **Merge** or **Need review**:

- All changed files are Documentation → **Merge**.
- Otherwise, a Global Config Change anywhere in the diff → **Need review**.
- Otherwise, determine **Regression**: does the diff modify or delete at least one line in a file that existed before this PR? (Any such line counts, regardless of test coverage — do not use PR title/intent for this call.) Purely new files / purely additive changes → **Non-Regression**.
- If Regression: does it contain functional changes (changes to business logic/behavior), or is it only non-functional (e.g. formatting, comments, perf tuning that doesn't alter business rules)? Functional → **Need review**. Only-non-functional → continue to the criteria check.
- If Non-Regression, or Regression-only-non-functional: continue to the criteria check (step 4). This applies identically whether the changed files are Production Code or Test Code.

### 4. Score Quality Criteria

Always compute this step, even when step 3 already produced a structural **Need review** — the score decides Medium vs. High. Score each criterion 0 (no breach) to 3 (severe breach), by your own judgment reading the diff and any tests (do not re-derive from CI tool output — CI already surfaces tool-detectable errors elsewhere; see ADR 0001):

1. Reasonable performance impact
2. Maintainability and readability (qualitative judgment; no complexity-linter dependency)
3. Business rules documented in spec/docs
4. Business rules supported by automated tests
5. No Critical Path present (score 3 if step 2 found one)
6. No problematic Global Config Change (score 3 if step 2 found one with real cross-environment blast radius; lower if the config change is trivial, e.g. a comment-only CI tweak)

All six are equally weighted — no criterion is inherently more severe than another; only the degree of breach matters.

### 5. Determine Criticality

- Step 3 said **Merge** → **Low**.
- Step 3 said **Need review** → **High** if any criterion scored 3 in step 4, otherwise **Medium**.

Low is never assigned when step 3 said Need review, regardless of scores: the tree's structural verdict is the authority on mergeability; scoring only grades severity among Need-review outcomes.

### 6. Report the result

- Print to stdout: the tree path taken, each file's category, the per-criterion scores with one-line justifications, and the final Criticality.
- Post the same reasoning as a PR comment: `gh pr comment <n> --body "..."`, prefixed with `> *This classification was generated by AI.*`
- Apply the label: remove any existing `criticality:low` / `criticality:medium` / `criticality:high` label first (`gh pr edit <n> --remove-label`), then add the new one (`gh pr edit <n> --add-label "criticality:<low|medium|high>"`). Create the label first with `gh label create` if it doesn't exist yet in the repo.
- Exit with a status code matching the Criticality: `0` for Low, `1` for Medium, `2` for High.
