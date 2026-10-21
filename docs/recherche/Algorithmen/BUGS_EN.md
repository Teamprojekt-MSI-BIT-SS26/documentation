# Open Bugs: What Needs to Be Fixed for Everything from the Papers to Work

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

**Where:** [algorithm_options.py:166-175](algorithms/ai/interfaces/algorithm_options.py#L166-L175)

`PriorityAlgorithmOptions.parse_json` creates a `TokenizationPriority` object for every priority, regardless of which `priority_type` was sent.

| Type sent | Result |
|---|---|
| `tokenization` | ✅ works |
| `diversity` | `direction` missing → `TokenizationConstraintDirection(None)` → `ValueError` |
| `project_requirement`, `project_preference`, `social_preference` | `strategy` missing → `DiversifyType(None)` → `ValueError` |

Via the API, this results in status 500. The CLI ([runs/cli.py:52](runs/cli.py#L52)) is affected as well, because it calls the same method.

## 2. Validation removes `priority_type` from the request data 🔴

**Where:** [algorithm_options.py:224](algorithms/ai/interfaces/algorithm_options.py#L224)

`validate_priority` calls `priority_data.pop("priority_type")` and thereby modifies the original dict. The data loader only makes a shallow copy of the data ([generate_teams_data_loader.py:42](algorithms/api/utils/generate_teams_data_loader.py#L42)), so the priority dicts are the same objects.

**Relation to Bug 1:** At the moment this is barely noticeable, because `parse_json` ignores the type anyway. But if only Bug 1 is fixed, `parse_json` no longer finds a `priority_type` and fails for **every** priority, including `tokenization`. Both bugs therefore have to be fixed together.

**Consequence already visible today:** The API only logs the request after parsing ([generate_teams.py:30-37](algorithms/api/views/generate_teams.py#L30-L37)), using the same dict objects. As a result, `priority_type` is missing from every priority in the stored logs.

The CLI is not affected, because it does not call the validator.

## 3. Schema and class of `RequirementPriority` do not match 🔴

**Where:** [priority.py:197-222](algorithms/ai/priority_algorithm/priority/priority.py#L197-L222)

The schema requires a **mandatory field** `criteria`, but the class has no fields and only the empty constructor `RequirementPriority()`:

| Request | Result |
|---|---|
| without `criteria` | ❌ The validator rejects it (status 400) |
| with `criteria` | ✅ The validator accepts it, then `TypeError: unexpected keyword argument 'criteria'` |

No request satisfies both places at the same time. At the moment, Bug 1 hides this problem.

**In addition, `criteria` cannot be set per requirement via the API either:** `ProjectRequirement` does have a `criteria` field ([project/dataclass.py:19](algorithms/dataclasses/project/dataclass.py#L19)). However, the requirement schema in the validator only knows `attribute`, `operator` and `value` ([generate_teams_validator.py:281-289](algorithms/api/validators/generate_teams_validator.py#L281-L289)), and the serializer also only reads these three fields ([project/serializer.py:22-26](algorithms/dataclasses/project/serializer.py#L22-L26)).

Via the API, `someone_meets_requirement` (one person in the team is enough) therefore always applies. This is sufficient for the paper results, because *Project Coverage* in Paper 2 uses exactly this rule. The variants "everyone" (`everyone_meets_requirement`) and "n members" (`some_number_of_members_meet_requirement`), however, cannot be reached.

## 4. The schema of `DiversityPriority` does not allow `max_num_choices` 🔴

**Where:** schema in [priority.py:186-194](algorithms/ai/priority_algorithm/priority/priority.py#L186-L194), check in [priority.py:147-152](algorithms/ai/priority_algorithm/priority/priority.py#L147-L152)

For `strategy: "concentrate"`, the class requires the field `max_num_choices` and fails with a `ValueError` without it. The schema, however, only knows `attribute_id` and `strategy` and rejects all other fields.

| Request | Result |
|---|---|
| `concentrate` without `max_num_choices` | Validator ✅, then `ValueError` in the class |
| `concentrate` with `max_num_choices` | ❌ The validator rejects it (status 400) |

Concentrating attributes, e.g. grouping students with shared time slots, is therefore not possible via the API. Paper 2 uses exactly this in Scenarios 2 and 3 and in both real courses. `diversify` works as soon as Bug 1 is fixed.

## 5. Enum values are not converted during parsing 🟠

**Where:** [priority.py:141-145](algorithms/ai/priority_algorithm/priority/priority.py#L141-L145) and [priority.py:226-245](algorithms/ai/priority_algorithm/priority/priority.py#L226-L245)

The JSON data contains strings (`"diversify"`, `"exclude"`). The classes, however, compare against enums. For example, `"exclude" == PreferenceDirection.EXCLUDE` evaluates to `False` ([priority.py:241](algorithms/ai/priority_algorithm/priority/priority.py#L241)). An "exclude" would therefore silently be treated like an "include".

## 6. Friend/enemy switches are ignored 🟠

**Where:** [algorithm_options.py:69-70](algorithms/ai/interfaces/algorithm_options.py#L69-L70), passed on in [priority_algorithm.py:149-150](algorithms/ai/priority_algorithm/priority_algorithm.py#L149-L150) and [social_algorithm.py:272-273](algorithms/ai/social_algorithm/social_algorithm.py#L272-L273)

`friend_behaviour` and `enemy_behaviour` (`enforce`, `invert`, `ignore`) are parsed and passed on, but no algorithm reads them. The dashboard switches from Paper 1 therefore have no effect.

## 7. Two methods in the validator are defined twice 🟠

**Where:** [generate_teams_validator.py](algorithms/api/validators/generate_teams_validator.py)

In Python, a second method with the same name replaces the first one. The first definition is never executed.

| Method | First definition (never executed) | Second definition (in effect) | Missing check |
|---|---|---|---|
| `_validate_student_attributes` | [l. 100-130](algorithms/api/validators/generate_teams_validator.py#L100-L130) | [l. 193-210](algorithms/api/validators/generate_teams_validator.py#L193-L210) | Whether the attribute keys are numeric and unique, and whether the attributes exist |
| `_validate_student_relationships` | [l. 132-152](algorithms/api/validators/generate_teams_validator.py#L132-L152) | [l. 212-228](algorithms/api/validators/generate_teams_validator.py#L212-L228) | Whether the relationship keys are unique |

`_validate_student_ids_unique`, `_validate_student_project_preferences_exist` and `_validate_student_project_preferences` are also defined twice, but both versions are identical.

Whether the "attribute exists" check is intended at all is an open question. It requires every attribute of a person to also appear in a project requirement. Gender without a matching requirement would then not be allowed. This is probably why the method was replaced.

## 8. In the priority algorithm, the weights only affect the initial solution 🟠

**Where:** [priority_algorithm.py:50-56](algorithms/ai/priority_algorithm/priority_algorithm.py#L50-L56) (initial solution), [priority_algorithm.py:85-93](algorithms/ai/priority_algorithm/priority_algorithm.py#L85-L93) (selection), [custom_dataclasses.py:40-59](algorithms/ai/priority_algorithm/custom_dataclasses.py#L40-L59) (score)

`PriorityAlgorithmOptions` inherits all options from `WeightAlgorithmOptions`, e.g. `preference_weight`. These weights, however, are only used once, to create the initial solution with the weight algorithm. During hill climbing, **only** the `priorities` count.

- **Without priorities** (`priorities: []`), every variant has a score of 0. When sorting, new variants with the same score are placed before the old ones. For up to 1500 rounds, students are therefore swapped randomly, and the good initial solution is lost.
- **With other priorities**, the same happens for all swaps that do not change these priorities. Project preferences can get worse unnoticed in the process.

This is derived from the code and has not yet been confirmed by a test run.

## 9. Missing answers count as a separate category 🟡

**Where:** [diversity_utility.py:45](algorithms/ai/weight_algorithm/utility/diversity_utility.py#L45) (intended according to a comment in the code)

Students without an answer (`-1`) are counted like a separate attribute value. This makes teams appear more diverse than they are.

---

## Not a Bug, but Noteworthy

- **Robinhood mutation** ([robinhood.py:117-151](algorithms/ai/priority_algorithm/mutations/robinhood.py#L117-L151)): If it finds no improvement, it returns the last state it tried instead of the original state. This does not make the result worse, because the best team set is kept anyway. It does cost computing time, though. This mutation is not in the paper and is not active by default.
- **Score cache** ([custom_dataclasses.py:43](algorithms/ai/priority_algorithm/custom_dataclasses.py#L43)): `if self.score:` treats a computed score of 0 as "not yet computed". A score of 0 is therefore recomputed every time. The result stays correct; it only costs computing time.
- **No tests:** There are no tests for `parse_json` and `validate_priority`.
