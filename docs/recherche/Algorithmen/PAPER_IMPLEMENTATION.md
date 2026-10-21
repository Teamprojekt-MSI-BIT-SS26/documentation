# Paper Requirements and Their Implementation in the Code

**Deutsch:** [Paper-Vorgaben und ihre Umsetzung im Code](PAPER_UMSETZUNG.md)

The code was compared with two papers:

- **Paper 1:** Hui et al. (2022), *Design Guidelines for a Team Formation and Analytics Software*
- **Paper 2:** Hui et al. (2025), *Diversity Considerations in Team Formation Design, Algorithm, and Measurement*

The findings are based on reading the code; it was not run. Code snapshot: 20 October 2026.

**The Priority algorithm from Paper 2 is implemented correctly in its core elements:** hill climbing, priority-based scoring, the tokenism function, and diversity metrics match the paper. However, at present only the tokenism priority works through the API and CLI. Project requirements, project preferences, social preferences, and diversity cannot be used as priorities there. In addition, the paper results cannot be reproduced because the comparison algorithms are missing and the benchmark scripts no longer run.

**Legend:** ✅ implemented · ⚠️ partial or deviates · ❌ missing or does not work

For all entries marked ⚠️ or ❌, [BUGS.md](BUGS.md) describes what needs to be done.

**Sections:**
1. [Project requirements](#1-project-requirements)
2. [Priority scoring](#2-priority-scoring)
3. [Tokenism function](#3-tokenism-function)
4. [Diversity metrics](#4-diversity-metrics)
5. [General](#5-general)

## Overview: How the algorithms work together

The overall logic can be understood as follows:

- The **Priority algorithm** is the actual optimization algorithm. It generates team assignments, evaluates them using priorities, and improves them iteratively through small swaps (`hill climbing`). Individual priorities such as skills, project preferences, diversity, or tokenism are implemented as separate rules in the Priority logic.
- The **Weight algorithm** is used mainly to create an **initial solution** or as an alternative scoring basis. In practice, it is used as GRR (Greedy Round Robin) to produce a reasonably good initial assignment before the Priority algorithm continues optimizing it.
- The **Random algorithm** provides a baseline for comparison: it produces a simple, non-optimized assignment and shows how much the targeted priorities improve the result.
- The **metrics** and evaluation functions then assess the quality of the assignment, for example project coverage, social satisfaction, diversity, common times, and runtime.

In short: **Weight or Random produces an initial solution, the Priority algorithm improves it, and the metrics then assess how good the result actually is.**

This makes clear that the individual components do not operate in isolation; they work together as a process:

1. Inputs define the criteria.
2. An initial solution is generated.
3. The Priority algorithm evaluates and improves it.
4. The metrics measure the result and show whether the requirements have been met.

---

## 1. Project requirements

| # | Paper requirement | Source | Status | Algorithm | Location in code |
|---|---|---|---|---|---|
| 1.1 | **Match skills to project requirements:** Each project has requirements (e.g. “knows Python”). A team meets a requirement if its members have the necessary skills. | P1 3.1.2, P2 3.1 | ✅ | Priority | [priority.py:197-212](algorithms/ai/priority_algorithm/priority/priority.py#L197-L212) (`RequirementPriority`); the *anyone / everyone / n members* rules are in [project/dataclass.py:19-80](algorithms/dataclasses/project/dataclass.py#L19-L80) |
| 1.2 | **Project preferences:** Students rank their preferred projects. The higher the assigned project is on their list, the better. | P1 3.1.2 | ✅ in the algorithm · ❌ through the API | Priority | [priority.py:226-245](algorithms/ai/priority_algorithm/priority/priority.py#L226-L245) (`ProjectPreferencePriority`). This does not work through the API (bugs 1, 2, and 4). |
| 1.3 | **Project requirements as an API priority:** Requirements should be configurable as a priority, just like any other criterion. | P2 3.1 | ❌ | Priority | [algorithm_options.py:166-175](algorithms/ai/interfaces/algorithm_options.py#L166-L175), [priority.py:214-222](algorithms/ai/priority_algorithm/priority/priority.py#L214-L222) (bugs 1–3) |
| 1.4 | ** *Project Coverage* metric:** The proportion of requirements met by at least one team member, averaged across all projects. | P2 4.4 | ✅ | Metric / Benchmark | [average_project_requirements_coverage.py:16-40](benchmarking/evaluations/metrics/average_project_requirements_coverage.py#L16-L40) |

## 2. Priority scoring

| # | Paper requirement | Source | Status | Algorithm | Location in code |
|---|---|---|---|---|---|
| 2.1 | **Ordered list of criteria:** The instructor specifies which criteria matter and in what order. A more important criterion always outweighs all less important criteria combined. | P2 3, 3.1 | ✅ | Priority | [priority_algorithm.py:86-92](algorithms/ai/priority_algorithm/priority_algorithm.py#L86-L92) sorts by score |
| 2.2 | **`satisfy()`:** A satisfaction level between 0 and 1 is calculated for each criterion and averaged across all teams. | P2 3.1 | ✅ | Priority | [scoring.py:38-50](algorithms/ai/priority_algorithm/scoring.py#L38-L50) |
| 2.3 | **`normalize()` with B levels:** The satisfaction level is assigned to one of B integer levels. Tiny differences (e.g. 0.451 and 0.4512) therefore do not count. | P2 3.1 | ✅ | Priority | [scoring.py:10](algorithms/ai/priority_algorithm/scoring.py#L10) (B = 25), [scoring.py:24-35](algorithms/ai/priority_algorithm/scoring.py#L24-L35) |
| 2.4 | **`score()`:** Total score = sum of level × B^(j−1). The powers of B ensure that the most important criterion dominates. | P2 3.1 | ✅ | Priority | [scoring.py:53-56](algorithms/ai/priority_algorithm/scoring.py#L53-L56) (multipliers) |
| 2.5 | **Priority types:** Project requirements, social preferences, concentrating attributes, diversifying attributes, and avoiding tokenism. | P2 3.1 | ✅ in the algorithm · ❌ through the API | Priority | Classes: [priority.py:27-299](algorithms/ai/priority_algorithm/priority/priority.py#L27-L299). Only the tokenism type comes through the API (bugs 1–4). |
| 2.6 | ** *Priority Satisfaction* metric:** Weighted sum of satisfaction levels with B = 2. Used only internally to test the parameters. | P2 3.3 | ✅ | Metric / Benchmark | [priority_satisfaction.py:39-69](benchmarking/evaluations/metrics/priority_satisfaction.py#L39-L69) (`exponent_base = 2`) |

## 3. Tokenism function

| # | Paper requirement | Source | Status | Algorithm | Location in code |
|---|---|---|---|---|---|
| 3.1 | **Avoid tokenism:** A minority group (e.g. women) is distributed so that each team either has none or at least m members of that group. This ensures that nobody is the only one in their team. | P2 3.1.3 | ✅ | Priority | [priority.py:27-48](algorithms/ai/priority_algorithm/priority/priority.py#L27-L48) (`TokenizationPriority`, m = `threshold`) |
| 3.2 | **Piecewise scoring function f(x):** x = number of minority members in a team, n = team size. It holds that f(0) = θ, f(0 < x < m) = 0, and f(m) = 1. After that, the function decreases linearly from 1 − θ to f(n) = θ, so that not everyone is placed on one team. | P2 3.1.3 | ✅ | Priority | [priority.py:79-93](algorithms/ai/priority_algorithm/priority/priority.py#L79-L93). The formula exactly matches the paper. |
| 3.3 | **θ = 0.2:** The minimum score given to a team with no minority members or too many of them. θ must be less than 0.5; otherwise the function increases instead of decreasing. | P2 3.1.3 | ✅ | Priority | [priority.py:73](algorithms/ai/priority_algorithm/priority/priority.py#L73) |
| 3.4 | **Alternative: concentrate minority members** (Scenario 3): Instead of distributing them, minority members are deliberately placed together, also to avoid tokenism. | P2 4.1 | ✅ | Priority | [priority.py:95-100](algorithms/ai/priority_algorithm/priority/priority.py#L95-L100) (`CONCENTRATE` variant with `MAX_OF`) |

## 4. Diversity metrics

| # | Paper requirement | Source | Status | Algorithm | Location in code |
|---|---|---|---|---|---|
| 4.1 | **Attributes as a bit vector:** Each value of an attribute gets its own bit (is a woman, is a man, etc.). This means no value is considered “closer” to another, as can happen with numeric codes such as 1/2/3. | P2 3.1.1 | ✅ | Priority | [priority/utils.py:6-20](algorithms/ai/priority_algorithm/priority/utils.py#L6-L20); for metrics, [cosine_similarity.py:14-42](benchmarking/evaluations/metrics/cosine_similarity.py#L14-L42) |
| 4.2 | **Use the Gini–Simpson index to score diversification:** The index measures how mixed a team is with respect to an attribute, i.e. the probability that two randomly selected members differ. | P2 3.1.2 | ✅ | Priority | [priority.py:183-184](algorithms/ai/priority_algorithm/priority/priority.py#L183-L184); the index is in [diversity_utility.py:44-53](algorithms/ai/weight_algorithm/utility/diversity_utility.py#L44-L53). As in the paper, the index never reaches 1 (see improvement 8 in BUGS.md). |
| 4.3 | **Concentrate (cluster) attributes:** People with the same attribute value are placed on the same team, e.g. students with common time slots. | P1 3.1.2, P2 3.1 | ✅ | Priority | [priority.py:155-181](algorithms/ai/priority_algorithm/priority/priority.py#L155-L181) (dot product of bit vectors) |
| 4.4 | ** *Intra-Heterogeneity* metric:** How different are the members *within* a team? Calculated as the cosine distance 1 − cos(θ) between the bit vectors (0 = identical, 1 = maximally different), averaged across all teams. | P2 4.4 | ✅ | Metric / Benchmark | [cosine_similarity.py:109-124](benchmarking/evaluations/metrics/cosine_similarity.py#L109-L124) (`AverageCosineDifference.calculate`) |
| 4.5 | ** *Inter-Homogeneity* metric:** Are all teams similarly diverse? Calculated as the standard deviation of intra-heterogeneity across all teams. Lower is better. | P2 4.4 | ✅ | Metric / Benchmark | [cosine_similarity.py:126-131](benchmarking/evaluations/metrics/cosine_similarity.py#L126-L131) (`calculate_stdev`). The standard deviation of similarity is mathematically equal to the standard deviation of distance. |
| 4.6 | ** *Solo Status* metric:** The number of people who are the only member of their minority group on their team, divided by the total number of students. This measures whether tokenism has been avoided. | P2 4.4 | ✅ | Metric / Benchmark | [average_solo_status.py:26-52](benchmarking/evaluations/metrics/average_solo_status.py#L26-L52) |

## 5. General

| # | Paper requirement | Source | Status | Algorithm | Location in code |
|---|---|---|---|---|---|
| 5.1 | **Hill climbing:** A search method that starts with an initial solution and improves it step by step through small changes. The paper follows several search paths at once to avoid getting stuck in a local optimum. | P2 3, Alg. 1 | ✅ | Priority | [priority_algorithm.py:71-97](algorithms/ai/priority_algorithm/priority_algorithm.py#L71-L97) |
| 5.2 | **Parameters `MAXITER`, `SPREAD`, `K`:** `MAXITER` = number of improvement rounds (search depth). `SPREAD` = number of variants per solution and round. `K` = number of best solutions retained for the next round. | P2 3, 3.3 | ✅ | Priority | [algorithm_config.py:58-63](algorithms/ai/interfaces/algorithm_config.py#L58-L63) (`MAX_ITERATE`, `MAX_SPREAD`, `MAX_KEEP`) |
| 5.3 | **Values used in the paper:** `MAXITER = 250`, `SPREAD = 100`, `K = 30`. The results in Paper 2 were produced with these values. | P2 3.3, 5.1 | ⚠️ | Priority | Code defaults: 1500 / 3 / 3, plus a 30-second time limit (`MAX_TIME`), [algorithm_config.py:59-62](algorithms/ai/interfaces/algorithm_config.py#L59-L62) (bug 7) |
| 5.4 | **Initial solution:** The search starts with a random assignment or the result of GRR (Greedy Round Robin, the Weight algorithm). GRR provides an initial solution that is already nearly optimal. | P2 3.3 | ✅ | Weight / Random | [priority_algorithm.py:41-59](algorithms/ai/priority_algorithm/priority_algorithm.py#L41-L59) (`START_TYPE`) |
| 5.5 | **`improve()`:** Two randomly selected students from different teams are swapped. Since the best solution so far is retained, the result never gets worse. | P2 3.2 | ✅ | Priority | [random_swap.py:17](algorithms/ai/priority_algorithm/mutations/random_swap.py#L17), set by default in [algorithm_config.py:79](algorithms/ai/interfaces/algorithm_config.py#L79) |
| 5.6 | **Weight algorithm with weights:** In the dashboard, the instructor sets how much social preferences, project preferences, project requirements, and diversity count. | P1 3.1.2 | ✅ | Weight | [algorithm_options.py:62-113](algorithms/ai/interfaces/algorithm_options.py#L62-L113), [weight_algorithm/](algorithms/ai/weight_algorithm/) |
| 5.7 | **Social preferences:** Students name friends (same team) and enemies (different teams). The algorithm tries to satisfy these wishes. | P1 3.1.2, P2 3.1 | ✅ in the algorithm · ❌ through the API | Priority | [priority.py:259-290](algorithms/ai/priority_algorithm/priority/priority.py#L259-L290) (`SocialPreferencePriority`). This does not work through the API (bug 1). |
| 5.8 | **Friend/enemy switches:** These preferences can be *considered*, *inverted*, or *ignored*. | P1 3.1.2 | ❌ | Priority / API | The options are read ([algorithm_options.py:69-70](algorithms/ai/interfaces/algorithm_options.py#L69-L70)) but are never evaluated (bug 5). |
| 5.9 | ** *Social Satisfaction* metric:** The share of members whose preferences are satisfied (at least one friend and no enemies on their team), averaged across all teams. | P2 4.4 | ⚠️ | Metric / Benchmark | The paper's definition for individual members exists as `has_friend_and_no_enemies` ([student_calculations.py:46](benchmarking/evaluations/metrics/utils/student_calculations.py#L46)). However, the metric ([average_social_satisfied.py:15-19](benchmarking/evaluations/metrics/average_social_satisfied.py#L15-L19)) records only yes or no for each team, not the proportion of members. The paper's 44.4% (= 4 of 9 teams) matches this approach. |
| 5.10 | ** *Common Times* metric:** The share of time slots that *all* team members are available, averaged across all teams. | P2 4.4 | ✅ | Metric / Benchmark | [average_timeslot_coverage.py:45-57](benchmarking/evaluations/metrics/average_timeslot_coverage.py#L45-L57) |
| 5.11 | ** *Run Time* metric:** How long an algorithm takes to produce an assignment. | P2 4.4, 5.1 | ✅ | Metric / Benchmark | [simulation.py:161-163](benchmarking/simulation/simulation.py#L161-L163) |
| 5.12 | **Comparison algorithms:** Random as a lower bound, DRR (Double Round Robin, assigns skills to projects), GRR, and Group Matcher (avoids tokenism in learning groups). | P2 4.3 | ⚠️ | Random / Weight / DRR / Group Matcher | Random and GRR: [random_algorithm/](algorithms/ai/random_algorithm/), [weight_algorithm/](algorithms/ai/weight_algorithm/). DRR and Group Matcher are missing; [enums.py:32-36](algorithms/dataclasses/enums.py#L32-L36) does not include them either (bug 6). |
| 5.13 | **Open evaluation framework:** Classes are simulated or loaded as real data, all algorithms run over multiple iterations, and the results are compared using the metrics. | P2 4 | ⚠️ | Benchmarking / Evaluation | The framework exists ([benchmarking/simulation/](benchmarking/simulation/), [benchmarking/data/](benchmarking/data/)). However, the scripts for the paper scenarios in [benchmarking/runs/](benchmarking/runs/) import modules that no longer exist (bug 6). |
