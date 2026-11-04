# Task 1: Running Locally Without Canvas

**Epic:** Epic 2 – Automate Team Generation  
**Acceptance criterion:** The algorithm runs in a local environment (VS Code) without a Canvas connection.

Part of the documentation on running the algorithm locally: [1. Running locally](LOKALE_AUSFUEHRUNG_EN.md) · [2. Test run](TESTLAUF_EN.md) · [3. Handover](UEBERGABE_EN.md)  
All paths refer to the [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms) repository (branch `41-bug-1-beheben`).

## Result

The UBC algorithm runs locally without Canvas. It also needs no Django server, no PostgreSQL
database, no API key and no Docker.

This is done by the new package `local_run/`. It accepts **the same JSON payload** that the API
expects at `POST /api/generate/teams`, and it uses the same chain as the API:

```
Payload (JSON file)
  → GenerateTeamsValidator      (checks the schema, as in the API)
  → GenerateTeamsDataLoader     (JSON → student, team and option objects)
  → AlgorithmRunner.generate()  (random | weight | social | priority)
  → TeamSetSerializer           (result → JSON, same format as the API response)
```

The local run and the API therefore share validation, algorithm and output format. What works
locally also works through the API.

## Why this works without Canvas

The algorithm itself (`algorithms/ai/`, `algorithms/dataclasses/`) does not depend on Canvas. It
receives ready-made `Student` objects and returns a `TeamSet`. Canvas is only the data source for
these students. Locally, the JSON file takes over that role.

The Django parts (`APILog`, database, API key middleware) are also limited to the view
`algorithms/api/views/generate_teams.py`. The validator and the data loader do not import Django
and can be used directly.

| Component              | API mode                  | Local run                    |
|------------------------|---------------------------|------------------------------|
| Data source            | Canvas → HTTP request     | JSON file                    |
| Validation             | `GenerateTeamsValidator`  | `GenerateTeamsValidator`     |
| Algorithm              | `AlgorithmRunner`         | `AlgorithmRunner`            |
| Logging to database    | `APILog` (PostgreSQL)     | not used                     |
| Authentication         | API key (`X-Api-Key`)     | not used                     |
| Output                 | HTTP response             | stdout or JSON file          |

## Setup

Requirement: **Python 3.9 or 3.10**. This is the version the CI tests with. Newer versions such as
3.15 do not work with the pinned package versions (e.g. `numpy==1.24.4`).

The local run only needs three packages from `requirements-local.txt`: `numpy`, `python-dotenv`
and `schema`. The full `requirements.txt` (Django, Jupyter, etc.) is only needed for the API,
benchmarking and notebooks.

**Option A – with `uv`** (downloads Python 3.10 automatically if needed):

```bash
uv venv --python 3.10 .venv
uv pip install --python .venv -r requirements-local.txt
```

**Option B – with `venv`** (Python 3.10 must be installed):

```powershell
py -3.10 -m venv .venv
.venv\Scripts\python -m pip install -r requirements-local.txt
```

Then select the `.venv` in VS Code via `Ctrl+Shift+P` → *Python: Select Interpreter*.

## Running

**In the terminal** (from the repository root):

```powershell
.venv\Scripts\python -m local_run
```

Without arguments, the included sample payload with 15 students is used. With your own files:

```powershell
.venv\Scripts\python -m local_run --input path\to\payload.json --output local_run\output\result.json --seed 1
```

**In VS Code with the debugger:** `.vscode/launch.json` contains two configurations. Select them in
the *Run and Debug* view (`Ctrl+Shift+D`):

- **Local run: Teams generieren** – starts `python -m local_run` and reads `env/local_run.env`
- **Local run: Tests** – runs the tests in `tests/test_local_run`

This lets you set breakpoints directly in the algorithm, e.g. in
`algorithms/ai/priority_algorithm/priority_algorithm.py`.

## Bugs fixed during implementation

In addition to bugs 1–4 from Spike 2 ([Open bugs](../../recherche/Algorithmen/BUGS_EN.md)), the
local run revealed another schema bug, which has been fixed:

- **The optional fields `relationships` and `project_preferences` caused a crash.** According to
  the schema, both fields are optional. If they were missing from the payload, the validator failed
  with an `AttributeError` (through the API: HTTP 500). The serializer raised a `KeyError` or set
  `project_preferences=None`, which later crashes in the algorithm. Missing fields are now treated
  as "no relationships" and "no preferences".
  Changed: `algorithms/api/validators/generate_teams_validator.py`,
  `algorithms/dataclasses/student/serializer.py`.

## New and changed files

| File                                    | Purpose                                                  |
|-----------------------------------------|----------------------------------------------------------|
| `local_run/run.py`                      | Local runner (validation, run, output)                   |
| `local_run/__main__.py`                 | Entry point for `python -m local_run`                    |
| `local_run/sample_data/*.json`          | Sample payload with 15 students                          |
| `env/local_run.env.example`             | Template for configuration via environment variables     |
| `requirements-local.txt`                | Minimal dependencies                                     |
| `.vscode/launch.json`                   | Run and debug configurations for VS Code                 |
| `tests/test_local_run/test_run.py`      | Automated tests of the test run                          |
| `.gitignore`                            | `.venv/`, `env/local_run.env`, `local_run/output/`       |
