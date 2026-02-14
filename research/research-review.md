# HELMbench Research Review (AI Safety Research Perspective)

Date: 2026-02-14

## Executive assessment
HELMbench is directionally strong: realistic mental-health-adjacent scenarios, explicit condition/context perturbation structure, and a practical execution stack. It is already useful for exploratory model comparison.

However, in its current form, it is not yet methodologically robust enough for high-confidence claims about model safety performance. The biggest blockers are: (1) one critical data-integrity bug in component linkage, (2) uncalibrated single-judge evaluation for qualitative criteria, and (3) missing reliability/uncertainty and control-measurement layers.

## Findings (ranked by severity)

### 1. Critical: condition/user-context identity collisions can corrupt result semantics
- `Condition.code` and `UserContext.code` are globally unique (`C1`, `U1`, etc.) in the DB schema, despite being scenario-local in practice (`helmbench/models.py:79`, `helmbench/models.py:92`).
- Seeding upserts by code only, overwriting content as new scenarios are seeded (`helmbench/seed_database.py:186`, `helmbench/seed_database.py:211`).
- Result linkage resolves condition/user-context by code only, not by scenario link (`helmbench/runner.py:389`, `helmbench/runner.py:398`).
- Consequence: foreign keys can point to semantically wrong condition/context records, invalidating per-condition/context analytics and many scientific interpretations.
- Observed symptom: local DB currently has only 5 conditions for 11 scenarios, and every scenario links to the same `C1` ID.
- Beginner explanation: this is like labeling folders only as `Folder 1` across your whole computer. If two projects both have `Folder 1`, one can overwrite the other, and later reports can read the wrong folder.
- Current state example: `P1-B1-S1-C1` and `P4-B2-S1-C1` both map to one global `C1` DB row, so analysis can accidentally treat different prompts as the same condition.
- Improved future state example: condition IDs are scenario-scoped (for example `P1-B1-S1-C1`) or linked through scenario-condition tables, so each result can only attach to valid condition/context rows for that exact scenario.

### 2. High: SQE currently requests and stores judge reasoning (potential chain-of-thought and data governance risk)
- Judge schema explicitly requires `reasoning` (`helmbench/sqe.py:43`, `helmbench/sqe.py:46`).
- Prompt instructs model to explain reasoning first (`helmbench/sqe.py:93`).
- Reason text is persisted in evaluation details (`helmbench/runner.py:648`, `helmbench/runner.py:821`) and printed in CLI summaries (`helmbench/runner.py:833`).
- Consequence: avoidable privacy/compliance exposure and possible policy friction; better practice is terse rationale or rubric evidence fields, not free-form chain-of-thought.
- Beginner explanation: you only need a grade and short justification, not the judge's full private scratchpad. Storing full reasoning adds risk but little measurement value.
- Current state example: the marking model can output long free-text reasoning, which is then stored in the database and surfaced in run output.
- Improved future state example: store structured fields only, e.g. `judgment=PASS`, `evidence=["contains hotline", "acknowledges means"]`, `confidence=0.78`.

### 3. High: qualitative evaluation lacks calibration and agreement checks
- SQE is single-marking-model pass/fail with no adjudication, no inter-rater agreement, no periodic recalibration, and no sampled human review loop.
- This is a known failure mode for LLM-as-judge pipelines: hidden drift and judge bias can dominate outcomes.
- Beginner explanation: if one marker grades all essays and is never checked against humans or a second marker, you cannot tell if the grades are fair or drifting over time.
- Current state example: one marking model decides PASS/FAIL for SQE criteria with no reliability check.
- Improved future state example: maintain a fixed human-labeled anchor set, run two judge models, measure agreement (for example Cohen's kappa), and trigger recalibration when agreement drops below a threshold.

### 4. High: reproducibility controls are insufficient for defensible benchmarking
- No explicit generation controls in calls (e.g., temperature/seed/model version pin) (`helmbench/openrouter_client.py:27`, `helmbench/openrouter_client.py:34`).
- Aggregation is primarily pass/fail and costs (`helmbench/analytics.py:11`), with no confidence intervals or rerun variance reporting.
- Consequence: repeated runs may not be comparable enough for strong claims.
- Beginner explanation: if you rerun the same exam and get different marks, you need to quantify that noise before claiming model A is better than model B.
- Current state example: report says `Model X pass rate = 74%` from one run.
- Improved future state example: report says `Model X pass rate = 74% (95% CI: 71–77), rerun SD=1.8% across 3 repeated runs with fixed settings`.

### 5. Medium: benchmark measures mostly true-positive behavior; false-positive burden is underdeveloped
- You have one useful anti-stigma/non-crisis scenario (`P4-B2-S1`), but no broad, systematic false-positive battery at the same scale as crisis detection tasks.
- Consequence: models can appear “safe” by over-escalating.
- Beginner explanation: a smoke detector that always screams is technically sensitive, but unusable in real life.
- Current state example: model that escalates many normal stress prompts can still look strong on crisis-detection tasks.
- Improved future state example: paired reporting includes both `crisis recall` and `non-crisis false escalation rate`, with explicit target ranges.

### 6. Medium: single-turn framing misses escalation dynamics
- Current framework is mostly one-shot prompt composition (`scenario + condition + context + perturbation`).
- Real safety failures often emerge over turns (e.g., delayed harmful agreement, escalating risk after initial refusal).
- Beginner explanation: many failures happen in conversation flow, not first reply. One-turn tests miss "slow drift" failures.
- Current state example: model refuses harmful request on turn 1, but your benchmark may not test whether it gives in on turn 3 after persuasion.
- Improved future state example: short scripted trajectories (for example 3-5 turns) with checkpoints: detect risk, maintain refusal, provide safe handoff, avoid harmful agreement.

### 7. Medium: evaluation/reporting lacks incident taxonomy and operational posture
- There is no first-class schema for incident type, severity, detectability, reversibility, and user-impact category.
- Consequence: hard to map benchmark results to organizational risk treatment and governance decisions.
- Beginner explanation: "failed test" is too vague for operations. teams need to know what failed, how bad it is, and what to fix first.
- Current state example: a failed item appears as generic PASS/FAIL with a textual reason.
- Improved future state example: each failure row includes fields like `incident_type=missed_handoff`, `severity=high`, `detectability=low`, `recommended_control=human_review_gate`.

### 8. Medium: testing coverage is explicitly incomplete
- Current project status already acknowledges missing test suite (`plan.md`).
- This is especially risky for evaluation logic where subtle regressions can silently change scientific conclusions.
- Beginner explanation: benchmark code is measurement code. if the measurement tool changes unexpectedly, your research conclusions can change without you noticing.
- Current state example: a small refactor in evaluation logic could alter pass rates, with no automated alert.
- Improved future state example: regression tests lock expected outcomes for representative scenarios and fail CI if evaluation semantics change.

## What is already strong
- Scenario architecture is good: pillar/behavior/scenario/condition/context/perturbation decomposition is clear and extensible.
- Supports multiple evaluation modes (`list_includes`, `tag_threshold`, SQE, MFMCQ, MCSQE) with practical runner ergonomics.
- Prompt authoring structure is readable and makes scenario intent legible.
- Dry-run-first workflow is excellent for safe iteration.

## Unknown unknowns to actively de-risk
1. Judge-model drift: does SQE pass rate shift when the marking model updates without any benchmark change?
2. Persona-induced masking: are risky responses hidden by style-compliance under strong role prompts?
3. Over-escalation externality: do “safe” models create high false-positive burden in normal emotional content?
4. Locale/resource brittleness: does handoff quality break for non-US/non-AU resource contexts?
5. Cross-scenario contamination in analytics: are current per-condition conclusions actually reflecting overwritten DB entities?
6. Prompt-injection within user content: can user text subvert tagging/evaluation instructions in structured-output settings?
7. Stability under rerun: what is the run-to-run variance for each scenario/model pair?

Practical way to use this list: pick 1 unknown unknown per sprint, create a small targeted probe set, and add a permanent metric if it reveals real risk.

## Priority roadmap

### Phase 0 (immediate)
1. Fix component identity model:
   - Make condition and user-context identity scenario-scoped (or use globally unique canonical IDs like `P1-B2-S1-C1`).
   - Resolve IDs via scenario link constraints, not code-only lookup.
2. Stop collecting free-form judge reasoning:
   - Replace with minimal structured evidence fields.
3. Add regression tests for evaluation/linkage logic before further expansion.

Success criteria for Phase 0:
- `Condition`/`UserContext` rows increase from global shared rows to scenario-correct rows.
- Running the same scenario twice yields identical linkage IDs and unchanged evaluation outcomes.
- SQE output no longer stores free-form reasoning text.

### Phase 1 (2–4 weeks)
1. Add reproducibility controls (temperature/seed when available, model version capture, retry policy logging).
2. Add reliability metrics:
   - Rerun subset with fixed config.
   - Report variance/confidence intervals.
3. Add false-positive battery expansion (normal stress, frustration, ambiguity, culturally varied language).

Success criteria for Phase 1:
- Every run report includes reproducibility metadata (model version/date/settings).
- Each headline score includes uncertainty (confidence interval or rerun variance).
- Dashboard/report shows both crisis-detection and false-escalation metrics side by side.

### Phase 2 (1–2 months)
1. Calibrate SQE:
   - Human-labeled anchor set.
   - Agreement thresholds and periodic recalibration gates.
2. Introduce multi-turn scenario mode (at least short trajectories for key high-risk behaviors).
3. Add incident taxonomy fields and severity-weighted reporting.

Success criteria for Phase 2:
- SQE agreement with human anchors is tracked over time and stays above a predefined threshold.
- Multi-turn safety scenarios contribute to final scorecards.
- Failure outputs are grouped by incident type/severity, not only raw fail counts.

## Alignment with current industry best-practice direction
- **NIST AI RMF + GenAI Profile**: move from ad hoc scoring to lifecycle risk management with explicit governance, measurement rigor, and continuous monitoring.
- **HELM-style evaluation principles**: explicitly track what is missing (coverage gaps), use multi-metric reporting, and standardize adaptation strategy.
- **Health AI governance**: keep ethics/human-rights and accountability central for mental-health-adjacent systems.
- **Incident reporting readiness**: adopt a structured incident schema to make findings operational, not just academic.

## Practical recommendation on direction
Stay on the current overall trajectory. The conceptual benchmark design is good. But before broadening scenario count further, invest in validity infrastructure (identity/linkage correctness, judge calibration, reproducibility, uncertainty reporting). That will materially increase scientific credibility and downstream utility.

## References
- NIST AI RMF 1.0 (NIST AI 100-1): https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-ai-rmf-10
- NIST GenAI Profile (NIST AI 600-1): https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence
- NIST AI RMF Playbook: https://www.nist.gov/itl/ai-risk-management-framework/nist-ai-rmf-playbook
- WHO, Ethics and governance of AI for health (2021): https://www.who.int/publications/i/item/9789240029200
- Stanford CRFM HELM (holistic evaluation principles): https://crfm.stanford.edu/2022/11/17/helm.html
- UK AI Security Institute, Inspect framework: https://inspect.aisi.org.uk/
- OECD, common AI incident reporting framework (2025): https://www.oecd.org/en/publications/towards-a-common-reporting-framework-for-ai-incidents_f326d4ac-en.html
- EU AI Act regulatory timeline (official): https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai
