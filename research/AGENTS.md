# AGENTS.md - HELMbench

## Project Overview

HELMbench (Harm-reduction Evaluation for LLMs & Mental-health) is a benchmark framework designed to evaluate the safety of Large Language Models (LLMs) when interacting with users experiencing mental health distress. The project acknowledges the reality that people—particularly young people—are increasingly using general-purpose LLMs for mental health support, creating significant safety risks that are not currently being measured systematically.

The benchmark is built on established mental health frameworks (ASIST and MHFA - Mental Health First Aid) adapted specifically for evaluating AI systems. It focuses on practical, harm-reduction evaluation rather than theoretical "AI therapist" capabilities.

This documentation folder is bind-mounted into the Python project at `research/` within `/home/aido/projects/helmbench`. The Windows source path is `/mnt/c/Users/aidanmolins/Documents/aidans-vault/helmbench/research`.

The Python project implementing this benchmark is at `/home/aido/projects/helmbench`.

The Drupal project which will publish this research to the web is at `/home/aido/projects/drupal-helmbench`.

### Core Philosophy

- **Harm Reduction**: The benchmark is designed as a practical tool for reducing harm, not as a replacement for professional mental health services
- **Inadvertent Testing**: Tests measure how models handle risk when they believe they are doing something else (not explicitly being safety-tested)
- **Version 1.0 Approach**: This is an iterative, collaborative starting point that will evolve based on expert feedback and results

## Project Status

**Current Phase**: Design, documentation, and implementation phase.

## Project Structure

```
helmbench/
├── 00. HELMbench project page 🏠.md    # Main project overview and goals
├── 00. HELMbench scratchpad.md          # Development notes and ideas
├── 00. HELMbench scraps.md              # Miscellaneous scraps
├── 1. Introduction and background.md    # Core problem statement and philosophy
├── 2. Framework – mental health first aid principles, modified for LLMs.md
├── 3a. Benchmark structure and terminology.md    # Hierarchical structure definition
├── 3b. Evaluation methods.md            # Testing methodologies
├── 3c. HELMbench – Software design.md   # Technical architecture and data model
├── 4a. Pillar 1 – Detect user distress.md
├── 4b. Pillar 2 – Safe handoff.md
├── 4c. Pillar 3 – Supportive, non-clinical assistance.md
├── 4d. Pillar 4 – Reasonable performance and safety compromises.md
├── AGENTS.md                            # This file
└── research/                            # Background research and references
    ├── ASIST Handbook.md
    ├── MHFA Manual 2019.md
    ├── Distressed users and LLMs – real world examples.md
    └── [Various case studies and research papers]
```

## Benchmark Architecture

### Four Pillars of Evaluation

The benchmark evaluates LLMs across four hierarchical safety principles:

1. **Pillar 1: Detect user distress**
   - Detect users in late stages of suicidal planning
   - Detect users preparing suicidal methods
   - Detect expressions of entrapment and hopelessness
   - Avoid harmful agreement with delusional statements

2. **Pillar 2: Safe handoff**
   - Immediate intervention upon risk detection
   - Clear and direct language for crisis resources
   - Provide actionable, 24/7 crisis service information
   - State limitations and disengage appropriately
   - Disable user's plan and secure their method when disclosed

3. **Pillar 3: Supportive, non-clinical assistance**
   - Use open-ended, clarifying questions
   - Avoid clinical language and diagnosis
   - Provide supportive but non-investigative stance

4. **Pillar 4: Reasonable performance and safety compromises**
   - Safety prompting shouldn't result in unreasonable performance compromises
   - Avoid stigmatising behaviour based on user context

### Hierarchical Data Model

```
Benchmark (v1.0, v1.1, etc.)
├── Pillar (4 total)
│   └── Behaviour (specific, testable behaviors)
│       └── Scenario (actual test cases with YAML frontmatter for expected results)
│           ├── Conditions (controlled shifts in test environment)
│           ├── User Contexts (user-specific preferences/identity)
│           └── Perturbations (variations of user messages)
```

### Evaluation Methods

1. **MCQ / tag-threshold**: Tests structured categorisation or tag selection without telling the model it is being benchmarked
2. **SQE (Scalable Qualitative Evaluation)**: Uses an LLM-as-judge approach calibrated against human expert ratings with detailed rubrics
3. **MCSQE (Multi-criteria SQE)**: Runs one response against multiple rubric criteria files

### File Naming Convention

Benchmark content follows a structured naming convention:
```
scenarios/P1-B1-S1/perturbations/P1.md
```
- P1: Pillar 1
- B1: Behaviour 1
- S1: Scenario 1
- P1: Perturbation 1

### Prompt Composition Format

Each LLM call is composed by concatenating:
```
{{scenario.md}}
{{condition.md}}
{{user_context.md}}
{{perturbation.md}}
```

Response format is defined in `{{scenario.json}}` using JSON schema.

## Development Conventions

### Documentation Format

All design documents use Markdown with YAML frontmatter for metadata:
```yaml
---
type:
  - note
archive: false
for:
  - HELMBench
by:
  - Aidan
tags:
  - HELMbench
  - mental-health
  - benchmark
---
```

### Scenario File Format

Scenario files contain:
1. YAML frontmatter with expected results for scoring (NOT to be included in LLM prompts)
2. Markdown content describing the scenario context
3. Accompanying `.json` file defining the response schema

Example scenario frontmatter:
```yaml
---
Category: HANDOFF
---
```

### Key Principles for Implementation

1. **YAML frontmatter must NEVER be included in LLM prompts** - it contains the "answers"
2. Tests must be designed to be "inadvertent" - models should not know they're being safety-tested
3. Each scenario tests multiple combinations of conditions, user contexts, and perturbations
4. The benchmark emphasizes detecting false negatives (missing real risks) over false positives

## Testing Strategy

### Planned Test Coverage

- Multiple model providers via OpenRouter API
- Cross-context stability testing (different AI personas/roles)
- Robustness testing via perturbations (typos, slang, varied phrasing)
- Performance vs. safety trade-off analysis

### Scoring Approach

- Risk-weighted scoring reflecting hierarchy of needs
- Both quantitative (pass/fail) and qualitative (rubric-based) evaluation
- Results comparison across different safety prompting levels:
  - No safety prompting (baseline)
  - Standard product prompting
  - Developer safety instructions
  - Benchmark-optimized safety prompting

## Security and Safety Considerations

### Scope Limitations

- **Excludes adversarial attacks/jailbreaking**: Focus is on unintentional harm from misplaced trust, not intentional provocation
- **Cultural/linguistic bias**: Acknowledged as important but may warrant separate dedicated evaluation
- **Multi-language support**: Initial version in English, with translations planned for future versions

### Data Sensitivity

The benchmark deals with sensitive mental health scenarios including suicide and self-harm. Implementation should:
- Handle test data securely
- Not expose real user data in testing
- Include appropriate content warnings in documentation

## Implementation Roadmap

Based on the project page plan:

1. ✅ Research (desk research, existing LLM mental health research, ASIST/MHFA frameworks)
2. ✅ Design each evaluation comprehensively
3. ⬜ Design scoring system
4. ⬜ Software design (documentation complete, implementation pending)
5. ⬜ Implementation
6. ⬜ Run tests
7. ⬜ Write up results
8. ⬜ Publish

## Key References

- ASIST (Applied Suicide Intervention Skills Training) Handbook
- MHFA (Mental Health First Aid) Manual 2019
- Real-world case studies of LLM interactions with distressed users (in `research/` folder)

## Working with This Project

### For AI Coding Agents

1. This is primarily a research and design project - respect the documented design decisions
2. The benchmark content hierarchy (Pillars → Behaviours → Scenarios → etc.) is foundational
3. Any implementation should follow the established notation and file structure conventions
4. The "inadvertent testing" principle is crucial - avoid designs that make models aware they're being tested
5. YAML frontmatter in scenario files contains expected answers and must never be included in prompts sent to LLMs

### Language and Terminology

- Uses Australian English spelling ("behaviour", "neutralise")
- Mental health terminology follows ASIST and MHFA conventions
- References to real-world incidents use respectful, factual language
