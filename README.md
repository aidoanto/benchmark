HELMBench benchmark content lives under `benchmark/scenarios/`.

## Scenario Tree

Benchmark runtime files are stored in a nested tree:

```text
benchmark/scenarios/
  P1/
    README.md
    B1/
      README.md
      S1/
        S1.md
        S1.json
        conditions.md
        user-contexts.md
        perturbations.md
```

- `P#/README.md`: pillar-level documentation.
- `P#/B#/README.md`: behaviour-level documentation and scenario notes.
- `P#/B#/S*/`: runnable scenario files.

## Scenario Notation

Canonical scenario/test notation remains:

- Scenario code: `P#-B#-S#` (or draft variant `S#A`, `S#B`)
- Test code: `P#-B#-S#-C#(-U#)-PT#`

Where:
- `P#` = Pillar
- `B#` = Behaviour
- `S#` = Scenario
- `C#` = Condition
- `U#` = User Context (optional)
- `PT#` = Perturbation

## Readability Suffixes

Scenario folders and code-named files may include an optional readability suffix after `--` without changing canonical IDs.

Examples:
- Folder: `S1--basic-threat-recognition/`
- File: `S1--basic-threat-recognition.md`

The runner still uses canonical codes such as `P1-B1-S1` and `P1-B1-S1-C1-PT1`.
