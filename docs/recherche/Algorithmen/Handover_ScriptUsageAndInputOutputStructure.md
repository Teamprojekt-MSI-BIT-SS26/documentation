# local_run/

**Deutsch:** [local_run/ (Handover)](Handover_SkriptbedienungUndEinAusgabestruktur.md)

Runs the team algorithm locally on your machine – without Canvas, without a server, without a database.

The input is the same JSON file that Canvas otherwise sends to the server. The script validates
the data exactly like the real server and then starts the algorithm.

## Setup (first time only)

Run all commands in the terminal from the project's root folder (`algorithms/`).

### Step 1: Create a virtual environment

```bash
python3 -m venv .venv
```

**Why:** This creates the folder `.venv/`, a separate Python installation just for this project.
That way the libraries do not interfere with other projects on your machine.

### Step 2: Activate the virtual environment

```bash
source .venv/bin/activate
```

**Why:** This makes the terminal use the Python from `.venv/` instead of the system Python.
The terminal prompt then starts with `(.venv)`. You have to repeat this step in every new terminal.

### Step 3: Install libraries

```bash
pip install -r requirements.txt
```

**Why:** The algorithm needs third-party libraries (e.g. `numpy`, `schema`).
Without them the program aborts with `No module named ...`. `requirements.txt` lists all of them.

### Step 4: Point VS Code to the environment

`Cmd+Shift+P` → "Python: Select Interpreter" → select `.venv`.

**Why:** Otherwise VS Code does not know the installed libraries, marks imports as errors
and the "Run" button uses the wrong Python.

## Running

### Step 5: Start with the sample data

```bash
python -m local_run.run
```

**Why:** This quickly shows you whether everything works. The script automatically uses
`sample_payload.json` (12 students) and shows the teams in the terminal.

### Step 6 (optional): Use your own data and save the result

```bash
python -m local_run.run my_data.json -o teams.json
```

**Why:** With your own data you test real cases. `-o` saves the complete result
as a JSON file so it can be processed further.

## Input structure

```json
{
  "students": [
    {"id": 1, "name": "Anna", "attributes": {"1": [1]}, "relationships": {"2": "friend"}}
  ],
  "algorithm_options": {"algorithm_type": "weight", "max_project_preferences": 0},
  "team_generation_options": {
    "total_teams": 4, "max_team_size": 3, "min_team_size": 3,
    "initial_teams": [{"id": 1}, {"id": 2}, {"id": 3}, {"id": 4}]
  }
}
```

- `students`: all students. `id` must be unique.
- `attributes`: properties of a person, e.g. skill level. Key = attribute number as text, value = list of numbers.
- `relationships`: relationship to other students (via their `id`): `friend`, `enemy` or `default`.
- `algorithm_type`: which algorithm – `random`, `weight`, `social` or `priority`.
- `team_generation_options`: number of teams and allowed team size. `initial_teams` needs an `id` per team.

A complete example is in `sample_payload.json`.

## Output structure

The terminal shows one line per team with the names:

```
Team 1: Anna, Emma, Jonas
Team 2: Ben, Felix, Lena
```

With `-o`, a JSON file is written as well. It contains, per team, the list of
students with all their data, in the same format as the server's response to Canvas.
