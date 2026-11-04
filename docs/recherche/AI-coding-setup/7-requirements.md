# Requirements

The current Claude subscription is too expensive for us, so we are looking for a cheaper alternative for AI-assisted development. Before comparing options, we collect what the team actually needs. The options are compared in [Evaluation](9-evaluation.md).

## Survey

The survey went to the 6 developers of the team; all of them answered.

| # | Question | Answers |
|---|---|---|
| 1 | Which mode did you mainly use? (agent / chat / autocomplete) | agent: 6 |
| 2 | What did you use AI for? (multiple choice) | writing code: 6, explaining code: 6, documentation: 6, debugging: 5, research: 5, refactoring: 2, Git / pull requests: 1 |
| 3 | Roughly how many hours per week did you use AI? (0–2 / 2–5 / 5–10 / more than 10) | 5–10: 6 |
| 4 | How often did a limit or waiting time slow you down? (never / rarely / often) | never: 3, rarely: 3 |
| 5 | What is indispensable for you? (open) | agent mode: 3, no answer: 3 |
| 6 | Which computer do you use? | MacBook M1, 16 GB RAM: 3; MacBook M1 Pro, 16 GB RAM: 1; Windows laptop with Intel i7, 16 GB RAM: 1; Windows laptop with AMD Ryzen 5000, 8 GB RAM: 1 |

The answers to question 4 refer to the current paid subscription. Which models run on the computers from question 6 is examined in [Our resources](8-our-resources.md).

## Requirements list

| Priority | Requirement | Source |
|---|---|---|
| Must-have | Agent mode | Survey questions 1 and 5 |
| Must-have | Cheaper than the current subscription | [Costs and budget](#costs-and-budget) |
| Must-have | Enough usage for 5–10 hours of agent mode per person and week | Survey questions 3 and 4 |
| Must-have | Context of at least 64K tokens | [Context](5-models.md#how-much-context-for-what) |
| Must-have | Usable with VS Code or in the terminal | Our development environment |
| Must-have | Our code is not used for training | Data privacy |
| Nice-to-have | Free choice of model | Flexibility for later changes |
| Nice-to-have | Local model for simple questions | [Our resources](8-our-resources.md#result-for-the-laptops) |
| Nice-to-have | Autocomplete | Survey question 1: not the main mode |

## Costs and budget

- **Current cost:** about 24.50 € per person per month for the Claude subscription. For 8 team members, this is about 196 € per month.
- **Budget:** there is no fixed budget. The alternative should cost as little as possible and less than the current subscription, while meeting the must-haves.
