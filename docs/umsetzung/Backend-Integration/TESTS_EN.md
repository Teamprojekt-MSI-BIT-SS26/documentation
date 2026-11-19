# Local Backend Tests

**Epic:** Epic 2 – Automate Team Generation  
**Ticket:** #51 – Technical Task 32: Backend Integration & Validation of the Team-Building Algorithm (local)

Part of the documentation on the backend integration: [1. Integration and interface](INTEGRATION_EN.md) · [2. Local backend tests](TESTS_EN.md)

All paths refer to the repository
[teamable-backend](https://github.com/Teamprojekt-MSI-BIT-SS26/teamable-backend), branch
`51-technical-task-32-backend-integration-validation-of-the-team-building-algorithm-local`.

## Overview

There are two kinds of tests in `src/app/tests`:

| File | What is tested | Needs the algorithms API |
| --- | --- | --- |
| `test_team_generation.py` | The request the backend sends to the API, the parameters, the error handling, saving and the response of the endpoints. The API is mocked. | no |
| `test_team_generation_integration.py` | The whole flow with all four algorithms: endpoint → algorithms API → saved teams → `team-sets` endpoint | yes |

The tests with the mocked API run in a few seconds and everywhere. The integration tests show that the
backend and the real algorithms API work together.

## Test dataset

The test data in `src/app/tests/data.py` is created before every test in a separate test database.
The normal database is not touched.

| Data | Content |
| --- | --- |
| Course | "Teamprojekt" with one instructor who calls the endpoints |
| Students | **15** (Anna, Ben, Clara, … Olivia, last name "Test") |
| Attribute "Programming experience" | Number attribute with the options 1, 2, 3. Every student has one value, assigned in turn. |
| Attribute "Study program" | Text attribute with the options MSI, BIT, WIN. Three students in a row have the same study program. |
| Friendships | Anna ↔ Ben, Clara → David, Emma → Felix |
| Enemies | Greta → Hannes |

This covers every kind of data the backend sends to the API: number and text attributes, friends and enemies.

## Running the tests

The backend must be set up (see the README of the backend). All commands run in the folder `src` of
the backend.

### Only the tests with the mocked API

```bash
python manage.py test app.tests.test_team_generation
```

### All tests including the integration tests

1. Start the algorithms API on port 8001, as described in
   [Local setup](INTEGRATION_EN.md#local-setup).
2. Set `TEAM_GENERATION_API_URL` and `TEAM_GENERATION_API_KEY` in the `.env` of the backend.
3. Run the tests:

```bash
python manage.py test app.tests -v 2
```

The integration tests take about 40 seconds, mostly because of the priority algorithm.

!!! note "When the integration tests are skipped"
    If no `TEAM_GENERATION_API_KEY` is set or the algorithms API is not reachable, the integration tests
    are skipped. The output then says `OK (skipped=1)`. If the API rejects the key, they fail with an
    error message, because the configuration is wrong.

## What the tests check

### Integration tests (`test_team_generation_integration.py`)

| Test | Check |
| --- | --- |
| `test_test_data_has_10_to_20_students` | The test dataset has 10 to 20 students |
| `test_default_parameters` | Without parameters: priority algorithm, 3 teams of 3 to 5 students |
| `test_every_algorithm_type` | `priority`, `weight`, `social` and `random` each return a valid team set |
| `test_custom_team_sizes` | With `total_teams=4`, `min_team_size=3`, `max_team_size=4`, every algorithm keeps to the parameters |
| `test_team_sizes_that_do_not_fit_are_rejected` | 15 students in 2 teams of at most 5: the backend answers `400` with a clear message |

After every successful run, the test fetches the result via
`GET /api/v1/courses/{id}/team-sets/{team_set_id}/` and checks:

- the structure: `id`, `name`, `teams`, `unassigned_students`, and `id`, `slug`, `name`, `members` per team,
- that `unassigned_students` is empty,
- that all 15 students appear exactly once,
- that the number and sizes of the teams match the parameters.

### Tests with the mocked API (`test_team_generation.py`)

| Area | Check |
| --- | --- |
| Request to the API | All 15 students with names, attribute values (the number attribute keeps its value, the text attribute is numbered) and relationships; default team options; derived priorities; options for every algorithm; selected attributes only; `total_teams` |
| Endpoint | Saves the team set and returns it via `team-sets`; passes the parameters on; the previous field `attribute` still works |
| Invalid parameters | `min_team_size` > `max_team_size`, unknown algorithm, `priorities` without `priority`, unknown attribute, attribute of another course: `400`, the API is not called |
| API errors | Invalid input → `400`, rejected key → `502`, server error → `502`, not reachable → `502`, timeout → `504`. No team set is saved in any case. |
| Invalid result | If a student is missing in the result, the backend answers `502` and saves nothing |
| Configuration | Without `TEAM_GENERATION_API_KEY`: `503` |

## Result of the test run

Locally on Windows 11, algorithms API with Python 3.10, backend with Python 3.11:

| Run | Result |
| --- | --- |
| `test_team_generation` (mocked API) | 15 tests, all successful, about 6 seconds |
| `test_team_generation_integration` (real API) | 5 tests, all successful, about 36 seconds |
| Algorithms repository: `tests.test_api`, `tests.test_local_run` | 78 + 9 tests, all successful |
| black and pyright (as in the backend CI) | no issues in the changed files |

## Troubleshooting

| Problem | Cause and solution |
| --- | --- |
| `OK (skipped=1)`, although the integration tests should run | The algorithms API is not running at the configured URL, or `TEAM_GENERATION_API_KEY` is missing in the `.env` of the backend |
| `The algorithms API rejected TEAM_GENERATION_API_KEY` | The hash of the key is not in `API_AUTH_ALLOWED_KEYS` of the algorithms API, or a different key was hashed there |
| Windows: "An Application Control policy has blocked this file" when starting `.venv\Scripts\python.exe` | The Windows feature "Smart App Control" blocks the Python launchers of virtual environments. A directly installed Python works, e.g. one installed with `uv python install 3.10`. |
| `400` with "students do not fit into … teams" | `total_teams`, `min_team_size` and `max_team_size` do not fit the number of students. This is intended, the check comes from the algorithms API. |
