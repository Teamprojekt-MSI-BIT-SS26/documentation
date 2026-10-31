# Local Run without Canvas – Documentation

**Deutsch:** [Lokaler Lauf ohne Canvas – Doku](LokalerLaufOhneCanvas.md)

## What this is about

Subtask from Epic 2: The team algorithm should run locally in VS Code, without Canvas.
Result: It runs. Test runs with 12 and 18 students formed teams with all four algorithms without
errors (details: [Test Run with Sample Data](TestRunWithSampleData.md)).

Important: The algorithm could already be run locally before (see next section). The only thing
missing on the machine were the installed libraries. What is new is a small folder `local_run/` that
uses exactly the data format of Canvas.

## What already existed

This code was not changed. These parts make sure the algorithm runs without Canvas.
None of them needs Canvas, a server or a database.

| Location | What it does | Why it matters for the local run |
| --- | --- | --- |
| `algorithms/ai/algorithm_runner.py` | `AlgorithmRunner`: selects the algorithm and starts it | The central entry point. Calling it with students and settings returns teams |
| `algorithms/ai/` | The algorithms themselves: `random`, `weight`, `social`, `priority` | Pure Python code, computes everything on your own machine |
| `algorithms/dataclasses/` | Building blocks such as `Student`, `Team`, `TeamSet` plus serializers (conversion from/to JSON) | So that data can be read from a file and teams can be output as JSON |
| `algorithms/api/validators/generate_teams_validator.py` | Checks whether the input data is structured correctly | Actually built for the server, but can also be called without a server |
| `algorithms/api/utils/generate_teams_data_loader.py` | Converts the validated JSON into objects for the algorithm | Also usable without a server |
| `runs/cli.py` | Starts the algorithm from a CSV file (a separate `transform.py` per CSV format) | Already ran without Canvas before |
| `manual_run/` | Older script for the priority algorithm with CSV | Already ran without Canvas before |
| `requirements.txt` | List of all libraries the project needs | Without these libraries nothing runs at all |

**Not needed for the local run:** the Django server (`algorithms/api/views/`, `middlewares/`,
`models/`) and the database (`docker-compose-dev.yml`). These are only needed for the path via Canvas.

The path via Canvas:
Canvas → Server → API key check → Database log → Validator → Loader → AlgorithmRunner → Teams.

## What is new

New is a shortcut: File → Validator → Loader → AlgorithmRunner → Teams. Server, database,
API key and Canvas are dropped. From the validator onwards, everything runs as in the real server.

| Location | What it does | Why |
| --- | --- | --- |
| `local_run/__init__.py` | Empty file, marks the folder as a Python package | Without it, `python -m local_run.run` does not work |
| `local_run/run.py` | Reads a JSON file, validates it, starts the algorithm, outputs the teams (with `-o` also as a file) | Does the same as the server, just without the server |
| `local_run/sample_payload.json` | Test data: 12 fictional students with skill levels and friend/enemy relationships, 4 teams of 3 | Same format that Canvas sends, so it can be swapped for real data |
| `local_run/sample_payload_18.json` | Second dataset: 18 students, 2 attributes, 4 teams of 4–5 people | Covers the upper end of the 10–20 range |
| `tests/test_local_run/` | Automated test: both datasets × all four algorithms | Makes the test run repeatable at any time (see [Test Run with Sample Data](TestRunWithSampleData.md)) |
| `.gitignore` | Added the line `.venv/` | So the virtual environment does not end up on GitHub |

The documentation for it is not in the `algorithms` repo but here in the `documentation` repo:

| Document | Contents | Why |
| --- | --- | --- |
| [Handover](Handover_ScriptUsageAndInputOutputStructure.md) | Guide: setup, running, input and output | So the team can use it without help (criterion "Handover") |
| This documentation | What already existed, what is new, open points | Records what was done and why |
| [Test Run with Sample Data](TestRunWithSampleData.md) | Sample data, automated test, results | Proves the criterion "test run with 10–20 students" |

**Not in the repo, only local: `.venv/`.** This is a separate, isolated Python installation for
this project. It contains the libraries from `requirements.txt`. Without it, no path worked
(error: `No module named 'schema'`).

The `.venv` does not belong on GitHub: It is several hundred MB in size, only works on your
own machine and can be recreated from `requirements.txt` at any time. Each person creates
it themselves (see [Handover](Handover_ScriptUsageAndInputOutputStructure.md), steps 1–3).

## Test run

Command: `python -m local_run.run` with `sample_payload.json` (weight algorithm, 12 students).
It ran through without errors.

```
Team 1: Anna, Emma, Jonas
Team 2: Ben, Felix, Lena
Team 3: Clara, Greta, Max
Team 4: David, Hannah, Sophie
```

In addition, there is a second dataset with 18 students and an automated test that checks both
datasets with all four algorithms:
`python -m unittest tests.test_local_run.test_local_run`. What the test does exactly is described in
[Test Run with Sample Data](TestRunWithSampleData.md).

## Open points

- [ ] Decide: keep `local_run` (tests with the real Canvas format incl. validator) or
      only use `runs/cli.py` (CSV data, without validator) and delete `local_run`.
- [x] Check why all 3 friend pairs were separated in the test run (Anna/Ben, Emma/Felix,
      Lena/Max). **Resolved, not a bug:** The weight algorithm always fills the smallest team. In
      the first round all teams are empty, so the first four people automatically end up in
      different teams. The social algorithm keeps Anna/Ben and Lena/Max together. Details:
      [Test Run with Sample Data](TestRunWithSampleData.md#observations).
- [ ] Relationships use different formats: `"friend"` in the input, `-1` in the output.
- [ ] In `algorithms/api/validators/generate_teams_validator.py` some methods are defined
      twice. Python only uses the second one, the first has no effect.
