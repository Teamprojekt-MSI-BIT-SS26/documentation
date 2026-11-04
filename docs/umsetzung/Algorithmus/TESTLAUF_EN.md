# Task 2: Test Run with Sample Data

**Epic:** Epic 2 – Automate Team Generation  
**Acceptance criterion:** A test run with sample data (10–20 students) calculates teams without errors.

Part of the documentation on running the algorithm locally: [1. Running locally](LOKALE_AUSFUEHRUNG_EN.md) · [2. Test run](TESTLAUF_EN.md) · [3. Handover](UEBERGABE_EN.md)  
All paths refer to the [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms) repository (branch `41-bug-1-beheben`).

## Result

The test run with **15 students** completes without errors, with all four algorithms (`random`,
`weight`, `social`, `priority`). It is set up as an automated test and can be repeated at any time.

## Sample data

File: `local_run/sample_data/payload_priority_15_students.json`

The data is fictional. It covers the main features the algorithm processes:

| Feature               | Content of the sample data                                                           |
|-----------------------|--------------------------------------------------------------------------------------|
| Students              | 15 (IDs 1–15)                                                                        |
| Teams                 | 4 projects: Web-App, Datenanalyse, Mobile, Spiele                                    |
| Team size             | min. 3, max. 4 (→ 3 teams of 4 and 1 team of 3)                                      |
| Attribute `1`         | Gender (1 = female, 2 = male, 3 = diverse)                                           |
| Attribute `2`         | Degree programme (1 = Computer Science, 2 = Business Informatics, 3 = Media Informatics) |
| Attribute `3`         | Programming experience (1–5)                                                         |
| Friendships           | 1↔2, 5↔6↔7, 11↔12                                                                    |
| Dislikes (enemy)      | 3→4, 9→10                                                                            |
| Project preferences   | 3 choices per person (`max_project_preferences = 3`)                                 |
| Project requirement   | Team Web-App needs at least one person with programming experience > 3              |

Algorithm: `priority` with four priorities in this order:

1. `diversity` – mix gender (attribute 1) within each team
2. `social_preference` – keep friends together, separate dislikes
3. `project_preference` – take project choices into account
4. `project_requirement` – meet project requirements

## Running the test

```powershell
.venv\Scripts\python -m local_run --seed 1 --output local_run\output\result.json
```

Terminal output (stderr):

```
Generating teams from ...\local_run\sample_data\payload_priority_15_students.json
Team Web-App (4 students): Elif Yilmaz, Felix Koch, Greta Richter, Noah Lange
Team Datenanalyse (4 students): Ben Schulz, Anna Becker, Clara Wagner, Hannes Wolf
Team Mobile (4 students): Katharina Zimmer, Olivia Schmitt, Leon Hartmann, Jonas Braun
Team Spiele (3 students): Ida Neumann, Mia Krause, David Hoffmann
Teams written to ...\local_run\output\result.json
```

Runtime: about 3 seconds. The same `--seed` always gives the same result.

## Evaluation of the result (seed 1)

| Check                             | Result                                                                   |
|-----------------------------------|--------------------------------------------------------------------------|
| All students assigned             | ✅ 15 of 15, nobody twice                                                |
| Team sizes respected              | ✅ 4 / 4 / 4 / 3                                                          |
| Friendships                       | ✅ 1+2 (Datenanalyse), 5+6+7 (Web-App), 11+12 (Mobile)                    |
| Dislikes separated                | ✅ 3 (Datenanalyse) / 4 (Spiele), 9 (Spiele) / 10 (Mobile)                |
| Gender mix                        | ✅ every team has women and men                                           |
| Web-App requirement               | ✅ Elif Yilmaz has programming experience 5                               |
| Project choices                   | 8× first choice, 3× second choice, 3× third choice, 1× no choice          |

The one unmet choice (Katharina Zimmer in team Mobile) is a deliberate trade-off: the priorities
"friends together" and "gender mix" rank above project choices. Katharina was placed with her
friend Leon Hartmann.

## Automated tests

File: `tests/test_local_run/test_run.py`

```powershell
.venv\Scripts\python -m unittest tests.test_local_run.test_run -v
```

| Test                                                    | Checks                                                              |
|---------------------------------------------------------|---------------------------------------------------------------------|
| `test_sample_payload__has_10_to_20_students`            | Sample data is within the required range of 10–20 students          |
| `test_generate_teams__sample_payload_creates_valid_teams` | Everyone assigned, number of teams and team sizes are correct     |
| `test_generate_teams__works_for_every_algorithm_type`   | Same students with `random`, `weight`, `social`, `priority`         |
| `test_generate_teams__same_seed_gives_same_teams`       | Same seed → same result                                             |
| `test_generate_teams__optional_student_fields_can_be_omitted` | Payload without `relationships`/`project_preferences` works   |
| `test_generate_teams__raises_if_students_do_not_fit_into_teams` | Unsuitable team sizes are rejected with a clear message     |
| `test_main__writes_result_to_output_file`               | Full run from the command line with an output file                  |
| `test_main__returns_error_code_for_missing_input`       | Missing input file → exit code 1 instead of a crash                 |

Result: **8 of 8 tests passed** (about 11 seconds).

The existing tests in `tests/test_api` (algorithm, validators, serializers) still pass: 73 of 73.
Running `python -m unittest discover tests` with only `requirements-local.txt` shows 7 import
errors in `tests/test_benchmarking`. These tests need `gitpython` from the full `requirements.txt`
and are unrelated to the local run.
