# Code vs. Paper: Does the Implementation Match?

**Deutsch:** [Code vs. Paper: Stimmt die Implementierung?](PAPER_VERGLEICH.md)

The code in this repository was compared with two papers:

- **Paper 1:** Hui et al. (2022), *Design Guidelines for a Team Formation and Analytics Software*
- **Paper 2:** Hui et al. (2025), *Diversity Considerations in Team Formation Design, Algorithm, and Measurement*

> This assessment is based on reading the code. It was not run because dependencies are missing from the local environment.

## Summary

The **Priority algorithm from Paper 2 is correctly implemented in its core elements.** There are issues in three areas:

1. an API bug,
2. a setting from Paper 1 that has no effect,
3. reproducibility of the paper results.

## What is implemented correctly ✅

| What the paper describes | Implementation in the code |
|---|---|
| **Hill climbing:** Improve many team sets in parallel and keep the best K | `priority_algorithm.py`: parameters `MAX_ITERATE`, `MAX_SPREAD`, `MAX_KEEP` |
| **Weighted priorities:** More important criteria count exponentially more | `scoring.py` with 25 scoring levels (“buckets”) |
| **Improvement step:** Swap two randomly selected students between teams | Default mutation `RandomSwapMutation` |
| **Avoiding tokenism:** For example, never leave exactly one woman alone in a team | `TokenizationPriority`; formula and θ = 0.2 exactly match the paper |
| **Diversity:** Gini–Simpson index per team | `DiversityPriority` |
| **Initial solution:** Random or generated with the Weight algorithm (= GRR) | `START_TYPE` setting |
| **Metrics:** Intra-heterogeneity, inter-homogeneity, solo status, project coverage, common times | `benchmarking/evaluations/metrics/` |

## Issues ⚠️

### 1. API bug: Only one type of priority works *(most important)*

Through the API, **every** priority is parsed as a tokenism priority (`algorithm_options.py`, `parse_json`). As a result, diversity, project, and social priorities are either not passed through or cause an error. The paper supports all of these types.

**Solution:** Select the appropriate class in `parse_json` using `get_priority_from_type()`.

### 2. “Include/Invert” for friends and enemies has no effect

Paper 1 shows switches for friends and enemies in the dashboard. The `friend_behaviour` and `enemy_behaviour` options are passed through in the code, but **never evaluated**.

### 3. The paper results cannot be reproduced

- The comparison algorithms **DRR** and **Group Matcher** are missing from the repository.
- The benchmark scripts that use them import modules that no longer exist (`algorithms.models`, `old.team_formation`).
- The tables and figures from Paper 2 therefore cannot be reproduced.

### 4. The default settings differ from the paper

The paper used `MAXITER = 250`, `SPREAD = 100`, and `K = 30`. The code defaults are much smaller, and the search **stops after 30 seconds** (`MAX_TIME`). Using the algorithm with its default settings therefore does not produce the quality reported in the paper.

### 5. Social Satisfaction: the paper text and code do not match

- **Paper text:** proportion of *satisfied members*
- **Code:** proportion of teams in which *all* members are satisfied

The numbers in the paper (e.g. 44.4% = 4 of 9 teams) match the code. The imprecise description is therefore in the paper, not the code.

## Minor observations

- **Diversity can never be “fully satisfied”:** For an attribute with two values, the Gini–Simpson index is at most 0.5. Diversity priorities therefore never reach the full score.
- **Robinhood-Holistic mutation:** If it finds no improvement, it returns an arbitrary combination instead of the best one. This does not harm the result because the best result is retained anyway, but it wastes computation time. This mutation is not in the paper.
- **Missing responses:** Students without a response slightly distort the diversity calculation.

## Recommended next steps

1. Fix the API bug (item 1).
2. Either implement the friend/enemy switches or remove them from the options (item 2).
3. Document the default parameters or align them with the paper (item 4).
4. If the paper results need to be reproduced, reintroduce DRR and Group Matcher (item 3).
