# HELMbench - Where Things Stand & What's Left

## Quick Orientation

HELMbench is a benchmark that tests whether LLMs are safe when interacting with users in mental distress. The research paper (in the Obsidian vault) defines **4 pillars**, each with multiple behaviours, scenarios, conditions, user contexts, and perturbations. The Python codebase runs those scenarios against LLMs via OpenRouter and stores/evaluates the results.

**The good news:** The core software works. You can load scenarios, compose prompts, send them to LLMs, get structured responses back, evaluate them, and store results in a database. The CLI is functional.

**The gap:** Only Pillar 1 scenarios have been written as benchmark content (the actual YAML/markdown files in `benchmark/`). Pillars 2, 3, and 4 are fully designed in the research but have zero implementation in code. There are also some software features that aren't built yet, and the paper itself isn't written.

---

## What's Done (so you don't redo it)

- [x] Research: desk research, frameworks (ASIST, MHFA), evaluation methods
- [x] Benchmark design: all 4 pillars fully designed with behaviours, scenarios, pass criteria
- [x] Software architecture: database schema, models, CLI, runner, evaluation engine
- [x] Evaluation strategies: `category_match` and `list_includes` both working
- [x] OpenRouter integration with structured JSON outputs
- [x] Concurrent batch execution
- [x] Cost tracking and analytics
- [x] Database seeding from benchmark files
- [x] Dry-run mode (works automatically without an API key)
- [x] Pillar 1 benchmark content:
  - [x] P1-B1-S1: Detect late-stage suicidal planning (4 conditions, 5 user contexts, 5 perturbations)
  - [x] P1-B2-S1: Detect suicidal method preparation (5 conditions, 6 user contexts, 7 perturbations)
  - [x] P1-B3-S1: Detect entrapment/hopelessness (5 conditions, 6 user contexts, 7 perturbations)
  - [x] P1-B4-S1: Avoid harmful agreement (5 conditions, 7 user contexts, 7 perturbations)
- [x] Jupyter notebook for exploring results

---

## What's Left - Bite-Sized Task List

### Phase 1: Finish Pillar 1 Scenario (P1-B4-S2)

Your research describes **two** scenarios for P1-B4 (avoid harmful agreement):
- P1-B4-S1 (clearly delusional statements) - **done**
- P1-B4-S2 (delusional statements with contradicting documents) - **not done**

This scenario is interesting because the LLM receives an objective document (like a work roster) that contradicts the user's claims, giving it reason to flag the delusion. It's a small, self-contained piece of work.

- [ ] Write scenario markdown for P1-B4-S2 (including the "contradicting document" mechanism)
- [ ] Write the JSON schema for P1-B4-S2
- [ ] Write conditions (workplace context with internal documents)
- [ ] Write perturbations (delusional user messages + contradicting documents)
- [ ] Write user contexts
- [ ] Test with a dry run

---

### Phase 2: Build the SQE Evaluation Method

Pillars 2, 3, and parts of Pillar 4 use **SQE** (Scalable Qualitative Evaluation) - an LLM-as-judge approach. This is a fundamentally different evaluation method from what's currently implemented. The existing code only handles `category_match` and `list_includes` (checking if the LLM picked the right category). SQE needs a second LLM call to *judge* the first LLM's free-text response against a rubric.

This is the single biggest piece of missing software. Without it, Pillars 2 and 3 can't be evaluated.

- [ ] Design the SQE evaluation flow (prompt to LLM -> response -> judge LLM scores response against rubric)
- [ ] Create a rubric format/template (YAML? Markdown?) for defining scoring criteria
- [ ] Implement the judge LLM call in `openrouter_client.py` (or a new module)
- [ ] Add `sqe` as a new evaluation strategy in `runner.py`
- [ ] Store SQE scores in the database (may need new columns or a related table on `Result`)
- [ ] Test with a simple hand-written rubric before tackling real scenarios

---

### Phase 3: Build the MFMCQ Evaluation Method

Pillar 4 Behaviour 1 uses **MFMCQ** (Multi-Field Multiple Choice Question). Each scenario needs to be run in 3 variations: complete prompt, main task only, safety task only. The evaluation then compares performance across all three to measure how safety prompting affects task performance.

- [ ] Design how MFMCQ variations are stored (3 variants per scenario in the file structure?)
- [ ] Implement the 3-variant runner logic
- [ ] Implement comparison scoring (task performance with vs without safety instructions)
- [ ] Add `mfmcq` as a new evaluation strategy

---

### Phase 4: Write Pillar 2 Benchmark Content (Safe Handoff)

The research defines 5 behaviours for Pillar 2. These all use the SQE evaluation method (so Phase 2 must be done first).

**P2-B1: Immediate intervention upon risk detection**
- [ ] Write scenario + rubric: general-purpose assistant context
- [ ] Write perturbations (user messages indicating crisis)
- [ ] Write conditions and user contexts

**P2-B2: Clear and direct language**
- [ ] Write scenario + rubric: must produce simple, unambiguous crisis message
- [ ] Write perturbations

**P2-B3: Provide actionable 24/7 resources (when country is known)**
- [ ] Write scenario + rubric: must provide phone number + web resource
- [ ] Write conditions (user indicates country of origin)
- [ ] Write perturbations

**P2-B4: State limitations and disengage**
- [ ] Write scenario + rubric: LLM must state it's an AI and stop engaging
- [ ] Write perturbations

**P2-B5: Disable user's plan and secure their method**
- [ ] Write scenario + rubric: general-purpose assistant, user discloses plan + means
- [ ] Write scenario + rubric: journaling assistant, user implies method
- [ ] Write scenario + rubric: AI companion, user incorporates plan into roleplay
- [ ] Write perturbations for each

---

### Phase 5: Write Pillar 3 Benchmark Content (Supportive, Non-Clinical Assistance)

**P3-B1: Use open-ended, clarifying questions**
- [ ] Write scenario + rubric (SQE): user in distress, LLM should ask broad questions not leading ones
- [ ] Write perturbations

**P3-B2: Avoid clinical language**
- [ ] Write scenario + rubric (SQE): user describes symptoms, LLM must not diagnose
- [ ] Write perturbations

---

### Phase 6: Write Pillar 4 Benchmark Content (Performance/Safety Tradeoffs)

**P4-B1: Safety prompting vs task performance (MFMCQ)**
- [ ] Write P4-B1-S1: journaling app tagging with complex tag list
- [ ] Write P4-B1-S2: journaling app with distress rating (1-10)
- [ ] Write P4-B1-S3: both tasks combined
- [ ] Write perturbations for each (everyday journal entries, not distressed)

**P4-B2: Avoid stigmatising behaviour based on user context**
- [ ] Write P4-B2-S1 (MCQ): journaling assistant + user history of suicide attempt
- [ ] Write P4-B2-S2 (MFMCQ): general-purpose assistant + user preferences mentioning mental health history
- [ ] Write P4-B2-S3 (SQE): life-coaching assistant + context from previous sessions

---

### Phase 7: Scoring System

The research mentions risk-weighted scoring (Pillar 1 worth ~50%, then decreasing) but the exact weights aren't finalised.

- [ ] Design the overall scoring formula (pillar weights, behaviour weights)
- [ ] Implement weighted score aggregation in `analytics.py`
- [ ] Generate per-model summary scores
- [ ] Generate per-pillar breakdown scores

---

### Phase 8: Run the Actual Benchmark

- [ ] Finalise the list of LLMs to test (update `benchmark/models.yml`)
- [ ] Do a dry run of the full benchmark to sanity-check prompts
- [ ] Run the full benchmark against all selected models
- [ ] Review results for obvious errors or issues with prompts
- [ ] Iterate on any scenarios that produce ambiguous or unexpected results

---

### Phase 9: Analysis & Paper

- [ ] Analyse results in Jupyter notebooks (per-model scores, per-pillar breakdowns, interesting examples)
- [ ] Identify "good", "bad", and "interesting" model responses for the paper
- [ ] Write the paper sections (Introduction, Methodology, Results, Discussion, Conclusion)
- [ ] Write appendices (sample prompts, rubrics)

---

### Nice-to-Have (Lower Priority)

These would improve the project but aren't blockers:

- [ ] Add a test suite (pytest) for the core evaluation logic
- [ ] Set up Alembic for database migrations (currently uses manual column additions)
- [ ] Add export functionality (CSV/JSON export of results)
- [ ] Add rate limiting / cost controls for large runs
- [ ] Non-semantic perturbation variants (linguistic, tonal - PT1a, PT1b style)
- [ ] Multi-turn conversation support
- [ ] Consider control questions (non-distress inputs to test false positive rates)

---

## Suggested Starting Point

If you're feeling stuck, **Phase 1** (P1-B4-S2) is the smallest, most self-contained task. It uses the existing evaluation infrastructure and just needs new benchmark content files. You could knock it out in a single session and get that momentum going again.

After that, **Phase 2** (SQE evaluation) is the most impactful piece of software work - it unlocks Pillars 2, 3, and parts of 4.
