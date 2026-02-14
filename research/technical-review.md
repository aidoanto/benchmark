# Technical Review: HELMbench MVP

## Scope and framing
This review is from the perspective of a Python engineer working on AI benchmark infrastructure. It is tailored for an MVP, a solo/new developer workflow, and practical research use.

## Top findings (ordered by severity)

### 1) Data integrity risk: condition/user-context identity is globally ambiguous
Severity: Critical

`Condition.code` and `UserContext.code` are globally unique, but the benchmark reuses `C1`, `U1`, etc. across many scenarios. Seeding and lookup resolve by code only, so content can be overwritten and FKs can point to semantically wrong records.

Evidence:
- `helmbench/models.py:79`
- `helmbench/models.py:92`
- `helmbench/seed_database.py:186`
- `helmbench/seed_database.py:211`
- `helmbench/runner.py:390`
- `helmbench/runner.py:398`
- Local seeded DB has 11 scenarios but only 5 `condition` rows and 8 `usercontext` rows.

Why it matters:
- Prompt composition is file-based (correct), but relational analytics via foreign keys can be wrong.
- Benchmark auditability is weakened.

Practical example (current vs future):
- Current state:
  Scenario `P1-B1-S1` and `P4-B2-S1` both have `C1`, but DB stores one global `Condition(code="C1")`.
  The later seed can overwrite `C1.content`, so old runs linked to `C1` may now appear to have the wrong condition text.
- Improved future state:
  DB stores condition identity as scenario-specific (for example `full_code="P1-B1-S1-C1"`), so each run always links to the exact authored condition variant.

MVP fix direction:
- Make condition/user-context scenario-scoped in DB, or
- switch to globally unique canonical codes like `P1-B1-S1-C1` / `P1-B1-S1-U1`.

### 2) Evaluation run counters are inconsistent for MFMCQ/MCSQE
Severity: High

`EvaluationRun.total_tests` is stored as number of test codes, but MFMCQ/MCSQE produce multiple evaluations per test code. `passed_tests` is computed from evaluation outcomes.

Evidence:
- `helmbench/runner.py:907`
- `helmbench/runner.py:1109`
- `helmbench/runner.py:1117`

Why it matters:
- Stored run metadata becomes mathematically inconsistent.
- Cross-run comparison quality drops.

Practical example (current vs future):
- Current state:
  For an MFMCQ scenario with 4 test codes and 8 evaluations per code, DB may save `total_tests=4` but `passed_tests=24`.
  This is confusing because pass count is larger than total tests.
- Improved future state:
  Store `total_test_codes=4` and `total_evaluations=32` (or rename fields clearly).
  Then metrics and dashboard percentages are unambiguous.

MVP fix direction:
- Store both `total_test_codes` and `total_evaluations`, or define run metrics strictly at evaluation granularity.

### 3) Single-test runs are not finalized in `EvaluationRun`
Severity: High

`run_single_test_case` creates an evaluation run but does not set completion metadata/counters.

Evidence:
- `helmbench/runner.py:719`
- No completion update path in `run_single_test_case` (unlike batch at `helmbench/runner.py:1103`)

Why it matters:
- Run history is incomplete and harder to trust in notebooks/reporting.

Practical example (current vs future):
- Current state:
  You run one single test from CLI. `evaluationrun.started_at` is set, but `completed_at` stays `NULL`.
  Querying “finished runs” misses this run, even though a result row exists.
- Improved future state:
  Single-run path sets `completed_at`, `total_tests` (or `total_evaluations`), and `passed_tests` right after saving result(s).

MVP fix direction:
- Finalize `EvaluationRun` in single-run path the same way as batch path.

### 4) SQE asks for chain-of-thought-style reasoning and stores it
Severity: Medium

Current SQE schema/prompt explicitly requests reasoning text and stores it as evaluation detail.

Evidence:
- `helmbench/sqe.py:43`
- `helmbench/sqe.py:93`
- `helmbench/sqe.py:149`

Why it matters:
- For production-style eval pipelines, concise rubric-grounded rationale is safer than requesting full reasoning traces.
- It can increase noise, tokens, and policy risk.

Practical example (current vs future):
- Current state:
  Marking model returns long freeform reasoning text for every evaluation, increasing token cost and variability.
- Improved future state:
  Marking model returns compact fields like:
  `justification="Missed direct handoff language in final response."`
  `evidence=["No referral instruction", "No crisis resource"]`

MVP fix direction:
- Replace `reasoning` with short `justification` (1-3 sentences, evidence-linked).

### 5) Reliability controls for API calls are minimal
Severity: Medium

No retry/backoff/circuit-breaking exists around OpenRouter calls.

Evidence:
- `helmbench/openrouter_client.py:58`
- `helmbench/openrouter_client.py:62`

Why it matters:
- Batch runs will be brittle under transient provider/network errors.

Practical example (current vs future):
- Current state:
  One temporary HTTP 502 immediately fails that test case.
- Improved future state:
  Retry up to 3 times on retryable errors (timeout/429/5xx) with backoff (for example 1s, 2s, 4s), then mark as error only if all retries fail.

MVP fix direction:
- Add simple bounded retries with exponential backoff + jitter for 429/5xx/timeouts.

### 6) Cost tracking currently misses your configured model
Severity: Medium

`models.yml` defaults to `google/gemini-2.5-flash`, but pricing table only includes `google/gemini-2.5-flash-lite`.

Evidence:
- `benchmark/models.yml:1`
- `benchmark/models.yml:6`
- `helmbench/pricing.py:10`

Why it matters:
- Cost summaries undercount or drop rows (`cost_usd=None`).

Practical example (current vs future):
- Current state:
  You run with `google/gemini-2.5-flash`; results save token counts but cost is `None`, so total run cost looks artificially low.
- Improved future state:
  Pricing map includes every model in `models.yml`, so every run has comparable cost metrics by model and scenario.

MVP fix direction:
- Add pricing for configured models or load pricing externally from a maintained source.

### 7) Documentation drift
Severity: Low

README says only two evaluation strategies exist and lists outdated roadmap items.

Evidence:
- `README.md:55`
- `README.md:111`

Why it matters:
- New contributors (including future you) will make wrong assumptions.

MVP fix direction:
- Keep README aligned with implemented eval modes (`tag_threshold`, `sqe`, `mfmcq`, `mcsqe`) and current architecture state.

## What is already strong
- Clean modular split across loader/runner/discovery/database.
- Good dry-run ergonomics for safe iteration.
- Flexible scenario authoring design (nested directories + readability suffix support).
- Useful evaluation abstraction in frontmatter.

## Practical MVP roadmap (recommended order)
1. Fix identity model for condition/user-context and migrate existing DB.
2. Normalize evaluation run accounting (single + batch) and backfill counters.
3. Add minimal reliability middleware for API calls (retry/backoff/timeouts).
4. Add a small test suite for core invariants.
5. Align README/docs with current code and data model.

## Minimal test plan to add now
- Unit test: scenario code parsing and directory/file resolution.
- Unit test: evaluation strategies (`category_match`, `list_includes`, `tag_threshold`, `sqe` wiring).
- Integration test (dry-run): one regular scenario, one MFMCQ, one MCSQE.
- DB test: seeding should not conflate same short codes across scenarios.

## Guidance for your learning path
- Keep MVP scope tight: prioritize data correctness and reproducibility over feature breadth.
- Prefer explicit invariants in code/comments over implicit assumptions (especially IDs and counters).
- Treat benchmark metadata as first-class product surface: if run metadata is wrong, conclusions are wrong.

## Suggested next implementation slice
If you want the highest ROI next, implement the schema/lookup fix for condition+user-context identity first, then patch run accounting in the same PR. That will immediately improve trustworthiness of your research outputs.
