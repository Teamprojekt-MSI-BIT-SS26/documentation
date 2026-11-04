# Task 3: Handover – Running, Interface and Configuration

**Epic:** Epic 2 – Automate Team Generation  
**Acceptance criterion:** Brief documentation on how to run the script and how input and output are structured.

Part of the documentation on running the algorithm locally: [1. Running locally](LOKALE_AUSFUEHRUNG_EN.md) · [2. Test run](TESTLAUF_EN.md) · [3. Handover](UEBERGABE_EN.md)  
All paths refer to the [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms) repository (branch `41-bug-1-beheben`).

## 1. Quick start

```powershell
# once: set up the environment (Python 3.10)
uv venv --python 3.10 .venv
uv pip install --python .venv -r requirements-local.txt

# optional: create the configuration
copy env\local_run.env.example env\local_run.env

# generate teams
.venv\Scripts\python -m local_run --input <payload.json> --output <result.json> --seed 1

# tests
.venv\Scripts\python -m unittest tests.test_local_run.test_run -v
```

| Argument           | Short    | Environment variable | Default                                     |
|--------------------|----------|----------------------|---------------------------------------------|
| `--input`          | `-i`     | `TEAMGEN_INPUT`      | `local_run/sample_data/payload_priority_15_students.json` |
| `--output`         | `-o`     | `TEAMGEN_OUTPUT`     | print to stdout                             |
| `--seed`           | `-s`     | `TEAMGEN_SEED`       | no seed (every run is random)               |

Relative paths are resolved from the repository root. The team overview and messages go to stderr,
the JSON result goes to stdout or to the file. You can therefore redirect the output directly,
e.g. `python -m local_run > result.json`.

**Exit codes:** `0` = teams created. `1` = error, e.g. file not found, invalid payload, team sizes
do not match the number of students, or someone was not assigned to a team. The error message is
written to stderr.

## 2. Input: payload

The payload is **identical to the request body of `POST /api/generate/teams`**. A backend that
later fetches the data from Canvas has to produce exactly this format.

```json
{
  "algorithm_options": { "algorithm_type": "priority", "...": "..." },
  "team_generation_options": {
    "max_team_size": 4,
    "min_team_size": 3,
    "total_teams": 4,
    "initial_teams": [ { "id": 1, "name": "Team Web-App", "project_id": 1, "requirements": [] } ]
  },
  "students": [
    {
      "id": 1,
      "name": "Anna Becker",
      "attributes": { "1": [1], "2": [1], "3": [2] },
      "relationships": { "2": "friend" },
      "project_preferences": [1, 4, 2]
    }
  ]
}
```

### `students[]`

| Field                 | Type                      | Required | Meaning                                                          |
|-----------------------|---------------------------|----------|------------------------------------------------------------------|
| `id`                  | int                       | yes      | Unique ID, e.g. Canvas user ID                                   |
| `attributes`          | `{ "<attr-id>": [int] }`  | yes      | Attribute values as a list, `{}` allowed. Keys as numeric strings |
| `name`                | string                    | no       | Display name                                                     |
| `relationships`       | `{ "<student-id>": str }` | no       | `"friend"`, `"enemy"` or `"default"`. IDs must exist             |
| `project_preferences` | `[int]`                   | no       | `project_id`s in order of preference, at most `max_project_preferences` |

Attributes are deliberately numbers only. What attribute `3` means and what the value `5` stands
for is defined by the caller. The backend has to document this mapping (survey → numbers).

### `team_generation_options`

| Field            | Type   | Meaning                                                                       |
|------------------|--------|-------------------------------------------------------------------------------|
| `total_teams`    | int    | Number of teams                                                               |
| `min_team_size`  | int    | Minimum size per team                                                         |
| `max_team_size`  | int    | Maximum size per team                                                         |
| `initial_teams`  | list   | Predefined teams/projects, `[]` allowed. Per team: `id` (required), `name`, `project_id`, `requirements`, `is_locked` |

The team sizes must match the number of students:
`total_teams × min_team_size ≤ number ≤ total_teams × max_team_size`. Locked teams (`is_locked`)
are not counted. Otherwise the payload is rejected.

`requirements[]` per team: `attribute` (int), `operator` (`"exactly"`, `"less than"` or
`"more than"`), `value` (int) and optionally `criteria`:

- `"someone_meets_requirement"` (default)
- `"everyone_meets_requirement"`
- `"some_number_of_members_meet_requirement"`, which also needs `num_members_required`

### `algorithm_options`

| `algorithm_type` | Further fields                                                                              |
|------------------|---------------------------------------------------------------------------------------------|
| `random`         | none                                                                                        |
| `weight`, `social` | `max_project_preferences` (required); optional `social_weight`, `diversity_weight`, `preference_weight`, `requirement_weight`, `friend_behaviour`/`enemy_behaviour` (`enforce`, `invert`, `ignore`), `attributes_to_diversify`, `attributes_to_concentrate` |
| `priority`       | like `weight`, plus `priorities` (required, order = importance)                             |

Possible entries in `priorities[]`:

| `priority_type`        | Fields                                                                                   |
|------------------------|------------------------------------------------------------------------------------------|
| `diversity`            | `attribute_id`, `strategy` (`diversify` or `concentrate`), `max_num_choices` (required with `concentrate`) |
| `tokenization`         | `attribute_id`, `strategy`, `direction` (`min_of` or `max_of`), `threshold`, `value`      |
| `social_preference`    | `max_num_friends`, `max_num_enemies`                                                     |
| `project_preference`   | `direction` (`include` or `exclude`), `max_project_preferences`                          |
| `project_requirement`  | none                                                                                     |

A complete example is in `local_run/sample_data/payload_priority_15_students.json`.

## 3. Output

The format is **identical to the API response**:

```json
{
  "teams": {
    "id": null,
    "name": null,
    "teams": [
      {
        "id": 1,
        "name": "Team Web-App",
        "project_id": 1,
        "requirements": [
          { "attribute": 3, "operator": "more than", "value": 3,
            "criteria": "someone_meets_requirement", "num_members_required": null }
        ],
        "students": [
          {
            "id": 5,
            "name": "Elif Yilmaz",
            "attributes": { "1": [1], "2": [3], "3": [5] },
            "relationships": { "6": -1 },
            "project_preferences": [1, 3, 4]
          }
        ]
      }
    ]
  },
  "metadata": { "version": "0.1.0", "timestamp": 1793791395.05 },
  "error": null
}
```

- For further processing, `teams.teams[].id` or `project_id` and `teams.teams[].students[].id`
  are usually enough.
- Note: in the output, relationships are **numbers** (`-1` = friend, `1.1` = enemy,
  `0` = default), in the input they are text. This is the existing behaviour of the API
  (`StudentSerializer`). The serializer reads both forms back in.
- Locally, the run stops with exit code 1 if someone was not assigned to a team. The API has no
  field for this case.

## 4. Configuration via environment variables

**Principle:** The **payload** contains the domain data: students, teams and the options chosen by
teaching staff. **Environment variables** contain technical runtime settings and credentials. These
belong neither in the code nor in the repository.

### Local run (`env/local_run.env`)

The template is `env/local_run.env.example`; copy it to `env/local_run.env`.
`env/local_run.env` is listed in `.gitignore`.

| Variable                       | Meaning                                                      |
|--------------------------------|--------------------------------------------------------------|
| `TEAMGEN_INPUT`                | Path to the payload                                          |
| `TEAMGEN_OUTPUT`               | Path to the output file (empty = stdout)                     |
| `TEAMGEN_SEED`                 | Random seed for reproducible results                         |
| `TEAMGEN_PRIORITY_MAX_TIME`    | Priority algorithm: maximum runtime in seconds (default 30)  |
| `TEAMGEN_PRIORITY_MAX_ITERATE` | Priority algorithm: maximum iterations (default 1500)        |
| `TEAMGEN_PRIORITY_MAX_KEEP`    | Priority algorithm: team sets kept per round (default 3)     |
| `TEAMGEN_PRIORITY_MAX_SPREAD`  | Priority algorithm: mutations per team set (default 3)       |

**Order when a value is set more than once:** command line argument before shell variable before
`env/local_run.env` before default value. Invalid numbers lead to a clear error message.

### API mode (`env/.env.prod`, see `env/sample.env`)

| Variable                                   | Meaning                                                        |
|--------------------------------------------|----------------------------------------------------------------|
| `DJANGO_SECRET_KEY`                        | Django secret (only used in `settings/prod.py`)                |
| `DJANGO_ALLOWED_HOSTS`                     | Allowed hosts, separated by spaces                             |
| `DB_ENGINE`, `DB_DATABASE`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT` | Database connection (alternatively: `RDS_*`) |
| `API_AUTH_ALLOWED_KEYS`                    | Allowed API keys as bcrypt hashes (hex), separated by spaces   |

The algorithm does not need Canvas credentials (e.g. a Canvas API token). They belong in the
backend that fetches the data from Canvas, and there too in environment variables.

### Security notes from the analysis (still open)

These points concern API mode, not the local run. They were not changed as part of this task:

1. **The API key middleware checks the key only after processing.** In
   `algorithms/api/middlewares/api_key_authentication.py`, `get_response(request)` is called
   before the key check. Requests with an invalid key therefore still run the algorithm and write
   a log entry. Only the response is discarded afterwards.
2. **The dev settings contain hard-coded values.** `algorithms/api/settings/__init__.py` sets
   `SECRET_KEY` directly in the code, `DEBUG = True` and `ALLOWED_HOSTS = ["*"]`. For production,
   `settings/prod.py` must be used.
3. **`utils/api_auth.py` looks for an `env/.env*` file on import.** If it is missing, the import
   fails with `StopIteration`.
