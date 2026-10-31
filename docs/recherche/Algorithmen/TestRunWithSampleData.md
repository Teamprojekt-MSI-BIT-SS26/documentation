# Test Run with Sample Data – What the Tests Do

**Deutsch:** [Testlauf mit Beispieldaten – Was die Tests machen](TestlaufMitBeispieldaten.md)

## What this is about

Acceptance criterion from Epic 2: *"A test run with sample data (10–20 students) calculates
teams without errors."*

Result: Fulfilled. With 12 and with 18 students, all four algorithms (`random`, `weight`,
`social`, `priority`) form valid teams without errors. An automated test checks this again
every time it is run.

How to set up the local run is described in the
[Handover](Handover_ScriptUsageAndInputOutputStructure.md).

## The sample data

Both files are in the `algorithms` repo in the folder `local_run/`. All people are fictional.
The files use the same format that Canvas sends to the server.

| File | Students | Teams | Attributes | Relationships |
| --- | --- | --- | --- | --- |
| `sample_payload.json` | 12 | 4 teams of 3 | 1 attribute (skill level 1–3) | 3 friend pairs, 1 enemy pair |
| `sample_payload_18.json` | 18 | 4 teams of 4–5 people (5/5/4/4) | 2 attributes (study program 1–3, experience 1–3) | 4 friend pairs, 1 group of three, 2 enemy pairs |

**Why two files:** The 12-student file is at the lower end of the 10–20 range, the 18-student
file at the upper end. The 18-student file is also harder: the teams are not the same size, two
attributes have to be mixed at the same time, and there are more relationships.

Relationships in the 18-student file:

- Friends: Anna ↔ Ben, Emma ↔ Felix, Lena → Max, Tim ↔ Ulla, Tim → Vincent, Yusuf → Zoe
- Enemies: Greta → Hannah, Jonas ↔ Wiebke

(`↔` = both named each other, `→` = only one person named the other)

## The automated test

File: `tests/test_local_run/test_local_run.py` in the `algorithms` repo.

### Running it

In the root folder `algorithms/`, with `.venv` activated:

```bash
python -m unittest tests.test_local_run.test_local_run
```

Expected output:

```
.
----------------------------------------------------------------------
Ran 1 test in 1.5s

OK
```

If something fails, it says `FAILED` instead of `OK`, together with the file and the algorithm
where it went wrong (e.g. `payload='sample_payload_18.json', algorithm='social'`).

### What the test does step by step

1. **Read the file:** It loads `sample_payload.json` and then `sample_payload_18.json`.
2. **Check the size:** It makes sure the file really contains 10–20 students.
3. **Set the algorithm:** It plugs in each of the four algorithms in turn. For `random`, all
   other options are removed (the server rejects them there). For `priority`, an empty list of
   priorities is added (the server requires this field).
4. **Calculate teams:** Validator → Loader → AlgorithmRunner, i.e. exactly the same steps as
   `local_run/run.py` and the real server.
5. **Check the result:** the four checks from the next table.

This gives 2 files × 4 algorithms = **8 test runs**. Each one runs as its own subtest
(`subTest`). If one fails, the others still run.

### The four checks

| Check | What exactly | Why |
| --- | --- | --- |
| Number of teams | There are exactly as many teams as specified in `total_teams` | Canvas expects exactly the number it asked for |
| Everyone assigned | Each person is in exactly one team, nobody is missing, nobody is duplicated | The most important basic rule: otherwise someone would end up without a team |
| Team size | Each team has at least `min_team_size` and at most `max_team_size` members | The instructor sets the size, it has to be respected |
| JSON output | The result can be converted to JSON | This is the format that `run.py -o` writes and the server sends back to Canvas |

### What the test deliberately does not check

The test only checks whether the result is **valid**, not whether it is **good**. It does not
check whether friends end up together and enemies apart.

**Why:** No algorithm guarantees that. `random` rolls the dice, and `weight` separates friends
simply because of how it works (see below). A test that required it would fail all the time,
even though the code works correctly. Team quality is assessed instead with the metrics in
`METRICS.md` and the `benchmarking/` folder.

## Results of the test runs

`weight` and `social` produce the same teams on every run. `random` and `priority` involve
randomness, so the teams look different each time. They are always valid, though.

### 12 students

| Algorithm | Team 1 | Team 2 | Team 3 | Team 4 |
| --- | --- | --- | --- | --- |
| `weight` | Anna, Emma, Jonas | Ben, Felix, Lena | Clara, Greta, Max | David, Hannah, Sophie |
| `social` | Anna, Ben, Clara | Felix, Lena, Max | David, Hannah, Jonas | Emma, Greta, Sophie |

### 18 students

| Algorithm | Team 1 | Team 2 | Team 3 | Team 4 |
| --- | --- | --- | --- | --- |
| `weight` | Anna, Hannah, Vincent, Emma, Tim | Ben, Greta, Zoe, Felix, Wiebke | Clara, Lena, Ulla, Max | David, Jonas, Yusuf, Sophie |
| `social` | Tim, Ulla, Vincent, Hannah | Anna, Ben, Clara, David, Sophie | Lena, Max, Zoe, Wiebke | Emma, Felix, Greta, Jonas, Yusuf |

### Observations

- **Enemies are separated in all runs** (Greta/Hannah, Jonas/Wiebke).
- **`social` keeps friends together better:** Anna/Ben, Lena/Max and the group of three
  Tim/Ulla/Vincent each end up in the same team.
- **`weight` separates many friend pairs.** This is not a bug but a result of how it works:
  the algorithm always fills the currently smallest team with the person who fits best. In the
  first round all teams are empty, so every person fits equally well. Each team then simply gets
  the next person from the list. The first four people (Anna, Ben, Clara, David) are therefore
  always separated, no matter what relationships they have.
  This resolves the open point from the [Local Run without Canvas](LocalRunWithoutCanvas.md)
  documentation.

## Changed files in the `algorithms` repo

| Location | What | Why |
| --- | --- | --- |
| `local_run/sample_payload_18.json` | New: second dataset with 18 students | Covers the upper end of the 10–20 range |
| `tests/test_local_run/__init__.py` | New: empty file | Marks the folder as a Python package, otherwise `unittest` does not find the test |
| `tests/test_local_run/test_local_run.py` | New: the automated test | Makes the test run repeatable at any time |
| `local_run/run.py` | Only the comment at the top: added the 18-student file and the test command | So both can be found directly in the code |
