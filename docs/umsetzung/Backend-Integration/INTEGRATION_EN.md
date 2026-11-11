# Backend Integration of the Team-Building Algorithm

**Epic:** Epic 2 – Automate Team Generation  
**Ticket:** #51 – Technical Task 32: Backend Integration & Validation of the Team-Building Algorithm (local)

Part of the documentation on the backend integration: [1. Integration and interface](INTEGRATION_EN.md) · [2. Local backend tests](TESTS_EN.md)

The code is on the branch
`51-technical-task-32-backend-integration-validation-of-the-team-building-algorithm-local` in both repositories:

- [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms): algorithms API
- [teamable-backend](https://github.com/Teamprojekt-MSI-BIT-SS26/teamable-backend): integration in the backend

## Result

| Acceptance criterion | Implementation |
| --- | --- |
| The algorithm is connected to the backend and can be invoked via endpoints | `POST /api/v1/courses/{id}/generate_teams/` calls the algorithms API and saves the teams as a `TeamSet`. They are available at `GET /api/v1/courses/{id}/team-sets/{team_set_id}/`. |
| Tests with a test dataset (10–20 students) | 15 students. 15 tests with a mocked API, 5 integration tests against the real API, all successful. See [Local backend tests](TESTS_EN.md). |
| Valid, parameter-compliant results | The backend checks every result before saving it (see [Result validation](#result-validation)). The tests also check it via the `team-sets` endpoint. |
| Documentation of the interface and the local tests | This page and [Local backend tests](TESTS_EN.md) |

## Architecture

Backend and algorithms stay two separate services. The backend calls the algorithms API over HTTP:

```
Frontend
  │  POST /api/v1/courses/{id}/generate_teams/
  ▼
Backend (teamable-backend, port 8000)
  │  1. collects the students, attribute responses and relationships of the course
  │  2. POST /api/generate/teams/  (header X-Api-Key)
  ▼
Algorithms API (algorithms, port 8001)
  │  random | weight | social | priority
  ▼
Backend
  │  3. checks the result
  │  4. saves TeamSet + teams
  ▼
{"team_set_id": 12}
```

**Why HTTP and not a library?** The algorithms API already existed, and the backend already had the call
prepared. The two projects also use different Python versions and dependencies: the algorithms need
Python 3.9 or 3.10 and Django 3.2, the backend Python 3.11 and Django 5.0. As separate services they do
not get in each other's way and can later run separately on the server.

## Changes

### Repository `algorithms`

The API could not be started before:

| Problem | Fix |
| --- | --- |
| Settings, `manage.py`, `wsgi.py`, `asgi.py` and `entrypoint.sh` referred to the module `api.api`, which no longer exists | Paths changed to `algorithms.api` |
| The API key middleware checked the key only **after** running the algorithm | The key is now checked before the view. Without configured keys, every request is rejected. |
| `utils/api_auth.py` failed with `StopIteration` without an `env/.env*` file | The settings now load `env/.env` or `env/.env.prod` explicitly. New: `python -m utils.api_auth <key>` generates the hash of a key. |
| Local development needed PostgreSQL | Without `DB_ENGINE`, a SQLite file `db.sqlite3` is used. |
| Shell scripts got CRLF line endings on Windows and did not run in the Linux container | `.gitattributes` with `*.sh text eol=lf` |
| The merge `f327ca3a` overwrote `local_run/run.py` with an older version, `tests/test_local_run/test_run.py` failed | Restored the version from `fe7fde72`. The calls from `main` (`python -m local_run.run <file>`) still work. |

New are the template `env/api.env.example`, the tests in
`tests/test_api/test_views/test_generate_teams.py` and the guide `algorithms/api/README.md`.

### Repository `teamable-backend`

| File | Change |
| --- | --- |
| `src/app/services/team_generation.py` | Rewritten: build the payload, call the API, check and save the result. The previous hard-coded pairing is removed. |
| `src/app/serializers/team_generation.py` | New: validates the request parameters |
| `src/app/views/course.py` | `generate_teams` passes the parameters and returns errors as `{"error": ...}` |
| `src/teamable/settings.py`, `sample.env` | `TEAM_GENERATION_API_URL`, `TEAM_GENERATION_API_KEY`, `TEAM_GENERATION_API_TIMEOUT` |
| `src/app/tests/` | Test data and tests, see [Local backend tests](TESTS_EN.md) |

## Local setup

Prerequisites: Python 3.10 for `algorithms`, Python 3.11 for `teamable-backend`. Docker and PostgreSQL
are not needed.

**1. Start the algorithms API** (in the repository `algorithms`):

```bash
python -m pip install -r requirements.txt
cp env/api.env.example env/.env
python -m utils.api_auth my-secret-key   # copy the output
```

Put the printed hash into `API_AUTH_ALLOWED_KEYS=` in `env/.env`. Then:

```bash
python manage.py migrate
python manage.py runserver 8001
```

**2. Configure the backend** (in the repository `teamable-backend`): put the **plain** key into the
`.env` (copy of `sample.env`):

```
TEAM_GENERATION_API_URL=http://localhost:8001/api/generate/teams/
TEAM_GENERATION_API_KEY=my-secret-key
TEAM_GENERATION_API_TIMEOUT=90
```

**3. Start the backend** as usual with `python manage.py runserver` in the folder `src`.

| Variable (backend) | Default | Meaning |
| --- | --- | --- |
| `TEAM_GENERATION_API_URL` | `http://localhost:8001/api/generate/teams/` | Endpoint of the algorithms API |
| `TEAM_GENERATION_API_KEY` | none | Plain API key. Without it, team generation answers `503`. |
| `TEAM_GENERATION_API_TIMEOUT` | `90` | Seconds to wait. The priority algorithm runs for up to 30 seconds. |

!!! info "Why two versions of the key?"
    The algorithms API only stores a bcrypt hash of the key, the backend sends the plain key in the
    header `X-Api-Key`. Someone who reads the `.env` of the algorithms API therefore does not know the key.

## Interface

`POST /api/v1/courses/{course_id}/generate_teams/`

Only instructors of the course can call the endpoint. Every field is optional:

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `algorithm_type` | `"priority"`, `"weight"`, `"social"`, `"random"` | `"priority"` | Algorithm |
| `min_team_size` | int ≥ 1 | `3` | Minimum team size |
| `max_team_size` | int ≥ 1 | `5` | Maximum team size |
| `total_teams` | int ≥ 1 | as few as possible: `ceil(students / max_team_size)` | Number of teams |
| `attributes` | list of attribute ids | all attributes of the course with options | Attributes the teams are mixed by |
| `attribute` | attribute id | | Single attribute, as sent by the frontend so far. Same as `attributes: [id]`. |
| `priorities` | list of objects | derived, see below | Only for `priority`: priorities in the format of the algorithms API, most important first |

**Priority as default:** the priority algorithm is the contribution of UBC and, according to the paper,
produces the best teams. The other algorithms can still be selected with `algorithm_type`.

Example:

```json
{
  "algorithm_type": "priority",
  "min_team_size": 3,
  "max_team_size": 4,
  "total_teams": 4
}
```

Response `200`:

```json
{"team_set_id": 12}
```

The teams are then returned by `GET /api/v1/courses/{course_id}/team-sets/{team_set_id}/`:

```json
{
  "id": 12,
  "name": "Team Set 1 (priority)",
  "teams": [
    {
      "id": 40,
      "slug": "3f0c…",
      "name": "Team 1",
      "members": [{"id": 3, "name": "Anna Test", "role": "Student", "…": "…"}]
    }
  ],
  "unassigned_students": []
}
```

### Errors

Errors are returned as `{"error": "<message>"}`, invalid fields as the usual serializer errors.
If an error occurs, **no** team set is saved.

| Status | Cause |
| --- | --- |
| `400` | Invalid fields, e.g. `min_team_size` > `max_team_size`, an attribute of another course, `priorities` without `priority` |
| `400` | The algorithms API rejects the input, e.g. because the students do not fit into `total_teams` teams of the given sizes |
| `400` | The course has no students |
| `502` | The algorithms API is not reachable, rejects the key, fails or returns an invalid team set |
| `503` | `TEAM_GENERATION_API_KEY` is not set |
| `504` | The algorithms API does not answer within `TEAM_GENERATION_API_TIMEOUT` seconds |

## Data mapping

Backend and algorithms store data differently. The backend maps it like this:

| Backend | Algorithms API |
| --- | --- |
| `CourseMember` with role `Student` | `students[].id`, `students[].name` |
| `AttributeResponse` | `students[].attributes`: `{"<attribute id>": [values]}`. Number attributes keep their value, all other attributes are numbered by option (0, 1, 2, …). |
| `Relationship` of type `Friend` / `Enemy` | `students[].relationships`: `{"<student id>": "friend" \| "enemy"}`. Relationships to people outside the course are ignored. |
| request fields | `team_generation_options` and `algorithm_options` |

For `weight`, `social` and `priority`, the selected attributes are sent as `attributes_to_diversify`.
The priority algorithm uses these options for its starting team set.

Without `priorities`, the priority algorithm gets these priorities, most important first:

1. `social_preference`: friends together, enemies apart. Only if there are relationships.
2. `diversity` with strategy `diversify`, one per attribute.

## Result validation

Before saving, the backend checks the result of the API. If a check fails, it answers `502` and saves
nothing:

- The number of teams is `total_teams`.
- Every student of the course is in exactly one team, and no unknown student is returned.
- Every team has between `min_team_size` and `max_team_size` members.

## Not implemented yet

- **Team requirements and project preferences:** `TeamRequirement`, team templates and project
  preferences are not sent yet. `initial_teams` is empty, `max_project_preferences` is 0. In the
  backend, requirements belong to team templates that are not assigned to a course yet, and the
  operators can only partly be mapped: the backend knows `GT`, `IN`, `CONTAINS` etc., the algorithms
  only `exactly`, `less than` and `more than`.
- **Frontend:** the frontend only sends `attribute` and therefore always uses the defaults for
  algorithm and team sizes.
- **Long requests on the server:** the request runs synchronously. On the server, the timeouts of
  gunicorn and nginx of both services must be longer than the runtime of the priority algorithm.
  `entrypoint.sh` of the algorithms also empties the database on every start (`flush`).
