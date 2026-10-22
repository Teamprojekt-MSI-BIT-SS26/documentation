# Open Bugs: What Needs to Be Fixed for Everything from the Papers to Work

!!! note "Code references"
    Paths such as `algorithms/ai/priority_algorithm/scoring.py:38-50` refer to the **Teamable-Analytics/algorithms** repository (snapshot: commit `e731bd7`).

Found by reading the code, not by running it. As of: 2026-10-20.
Which paper requirement is affected in each case is listed in [PAPER_UMSETZUNG.md](PAPER_UMSETZUNG.md).

## Overview

| # | Bug | Consequence | Severity |
|---|---|---|---|
| 1 | API and CLI parse every priority as a tokenism priority | Diversity, project and social priorities cause an error | 🔴 high |
| 2 | Validation removes `priority_type` from the request data | Prevents Bug 1 from being fixed on its own; the type is also missing from the API logs | 🔴 high |
| 3 | Schema and class of `RequirementPriority` do not match; `criteria` cannot be set anywhere via the API | Project requirement priorities cannot be created; via the API, "one person is enough" always applies | 🔴 high |
| 4 | The schema of `DiversityPriority` does not allow `max_num_choices` | Concentrating attributes (e.g. time slots) is impossible via the API | 🔴 high |
| 5 | Enum values are not converted during parsing | Project preferences behave incorrectly | 🟠 medium |
| 6 | Friend/enemy switches are ignored | The dashboard option from Paper 1 has no effect | 🟠 medium |
| 7 | Two methods in the validator are defined twice | Two checks never run | 🟠 medium |
| 8 | In the priority algorithm, the weights only affect the initial solution | Without a matching priority, fulfilled project preferences, for example, are lost again during hill climbing | 🟠 medium |
| 9 | Missing answers count as a separate category | Diversity values are slightly distorted | 🟡 low |

Bugs 1 to 5 concern the same topic (parsing priorities) and should be fixed together.

**Only `algorithm_type: "priority"` is affected.** With `weight`, `social` and `random`, the code paths of Bugs 1 to 5 are not executed.

---

## 1. API and CLI parse every priority as a tokenism priority 🔴

**Where:** `algorithms/ai/interfaces/algorithm_options.py:166-175`

`PriorityAlgorithmOptions.parse_json` creates a `TokenizationPriority` object for every priority, regardless of which `priority_type` was sent.

| Type sent | Result |
|---|---|
| `tokenization` | ✅ works |
| `diversity` | `direction` missing → `TokenizationConstraintDirection(None)` → `ValueError` |
| `project_requirement`, `project_preference`, `social_preference` | `strategy` missing → `DiversifyType(None)` → `ValueError` |

Via the API, this results in status 500. The CLI (`runs/cli.py:52`) is affected as well, because it calls the same method.

## 2. Validation removes `priority_type` from the request data 🔴

**Where:** `algorithms/ai/interfaces/algorithm_options.py:224`

`validate_priority` calls `priority_data.pop("priority_type")` and thereby modifies the original dict. The data loader only makes a shallow copy of the data (`algorithms/api/utils/generate_teams_data_loader.py:42`), so the priority dicts are the same objects.

**Relation to Bug 1:** At the moment this is barely noticeable, because `parse_json` ignores the type anyway. But if only Bug 1 is fixed, `parse_json` no longer finds a `priority_type` and fails for **every** priority, including `tokenization`. Both bugs therefore have to be fixed together.

**Consequence already visible today:** The API only logs the request after parsing (`algorithms/api/views/generate_teams.py:30-37`), using the same dict objects. As a result, `priority_type` is missing from every priority in the stored logs.

The CLI is not affected, because it does not call the validator.

## 3. Schema and class of `RequirementPriority` do not match 🔴

**Where:** `algorithms/ai/priority_algorithm/priority/priority.py:197-222`

The schema requires a **mandatory field** `criteria`, but the class has no fields and only the empty constructor `RequirementPriority()`:

| Request | Result |
|---|---|
| without `criteria` | ❌ The validator rejects it (status 400) |
| with `criteria` | ✅ The validator accepts it, then `TypeError: unexpected keyword argument 'criteria'` |

No request satisfies both places at the same time. At the moment, Bug 1 hides this problem.

**In addition, `criteria` cannot be set per requirement via the API either:** `ProjectRequirement` does have a `criteria` field (`algorithms/dataclasses/project/dataclass.py:19`). However, the requirement schema in the validator only knows `attribute`, `operator` and `value` (`algorithms/api/validators/generate_teams_validator.py:281-289`), and the serializer also only reads these three fields (`algorithms/dataclasses/project/serializer.py:22-26`).

Via the API, `someone_meets_requirement` (one person in the team is enough) therefore always applies. This is sufficient for the paper results, because *Project Coverage* in Paper 2 uses exactly this rule. The variants "everyone" (`everyone_meets_requirement`) and "n members" (`some_number_of_members_meet_requirement`), however, cannot be reached.

## 4. The schema of `DiversityPriority` does not allow `max_num_choices` 🔴

**Where:** schema in `algorithms/ai/priority_algorithm/priority/priority.py:186-194`, check in `algorithms/ai/priority_algorithm/priority/priority.py:147-152`

For `strategy: "concentrate"`, the class requires the field `max_num_choices` and fails with a `ValueError` without it. The schema, however, only knows `attribute_id` and `strategy` and rejects all other fields.

| Request | Result |
|---|---|
| `concentrate` without `max_num_choices` | Validator ✅, then `ValueError` in the class |
| `concentrate` with `max_num_choices` | ❌ The validator rejects it (status 400) |

Concentrating attributes, e.g. grouping students with shared time slots, is therefore not possible via the API. Paper 2 uses exactly this in Scenarios 2 and 3 and in both real courses. `diversify` works as soon as Bug 1 is fixed.

## 5. Enum values are not converted during parsing 🟠

**Where:** `algorithms/ai/priority_algorithm/priority/priority.py:141-145` and `algorithms/ai/priority_algorithm/priority/priority.py:226-245`

The JSON data contains strings (`"diversify"`, `"exclude"`). The classes, however, compare against enums. For example, `"exclude" == PreferenceDirection.EXCLUDE` evaluates to `False` (`algorithms/ai/priority_algorithm/priority/priority.py:241`). An "exclude" would therefore silently be treated like an "include".

## 6. Friend/enemy switches are ignored 🟠

**Where:** `algorithms/ai/interfaces/algorithm_options.py:69-70`, passed on in `algorithms/ai/priority_algorithm/priority_algorithm.py:149-150` and `algorithms/ai/social_algorithm/social_algorithm.py:272-273`

`friend_behaviour` and `enemy_behaviour` (`enforce`, `invert`, `ignore`) are parsed and passed on, but no algorithm reads them. The dashboard switches from Paper 1 therefore have no effect.

## 7. Two methods in the validator are defined twice 🟠

**Where:** `algorithms/api/validators/generate_teams_validator.py`

In Python, a second method with the same name replaces the first one. The first definition is never executed.

| Method | First definition (never executed) | Second definition (in effect) | Missing check |
|---|---|---|---|
| `_validate_student_attributes` | `algorithms/api/validators/generate_teams_validator.py:100-130` | `algorithms/api/validators/generate_teams_validator.py:193-210` | Whether the attribute keys are numeric and unique, and whether the attributes exist |
| `_validate_student_relationships` | `algorithms/api/validators/generate_teams_validator.py:132-152` | `algorithms/api/validators/generate_teams_validator.py:212-228` | Whether the relationship keys are unique |

`_validate_student_ids_unique`, `_validate_student_project_preferences_exist` and `_validate_student_project_preferences` are also defined twice, but both versions are identical.

Whether the "attribute exists" check is intended at all is an open question. It requires every attribute of a person to also appear in a project requirement. Gender without a matching requirement would then not be allowed. This is probably why the method was replaced.

## 8. In the priority algorithm, the weights only affect the initial solution 🟠

**Where:** `algorithms/ai/priority_algorithm/priority_algorithm.py:50-56` (initial solution), `algorithms/ai/priority_algorithm/priority_algorithm.py:85-93` (selection), `algorithms/ai/priority_algorithm/custom_dataclasses.py:40-59` (score)

`PriorityAlgorithmOptions` inherits all options from `WeightAlgorithmOptions`, e.g. `preference_weight`. These weights, however, are only used once, to create the initial solution with the weight algorithm. During hill climbing, **only** the `priorities` count.

- **Without priorities** (`priorities: []`), every variant has a score of 0. When sorting, new variants with the same score are placed before the old ones. For up to 1500 rounds, students are therefore swapped randomly, and the good initial solution is lost.
- **With other priorities**, the same happens for all swaps that do not change these priorities. Project preferences can get worse unnoticed in the process.

This is derived from the code and has not yet been confirmed by a test run.

## 9. Missing answers count as a separate category 🟡

**Where:** `algorithms/ai/weight_algorithm/utility/diversity_utility.py:45` (intended according to a comment in the code)

Students without an answer (`-1`) are counted like a separate attribute value. This makes teams appear more diverse than they are.

---

## Not a Bug, but Noteworthy

- **Robinhood mutation** (`algorithms/ai/priority_algorithm/mutations/robinhood.py:117-151`): If it finds no improvement, it returns the last state it tried instead of the original state. This does not make the result worse, because the best team set is kept anyway. It does cost computing time, though. This mutation is not in the paper and is not active by default.
- **Score cache** (`algorithms/ai/priority_algorithm/custom_dataclasses.py:43`): `if self.score:` treats a computed score of 0 as "not yet computed". A score of 0 is therefore recomputed every time. The result stays correct; it only costs computing time.
- **No tests:** There are no tests for `parse_json` and `validate_priority`.
