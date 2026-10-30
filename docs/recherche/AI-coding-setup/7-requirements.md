# Requirements

The current Claude subscription exceeds our budget, so we are looking for a cheaper alternative for AI-assisted development. Before comparing tools, we collect what the team actually needs. The results are summarized under [Decisions](#decisions).

## Decisions and the questions they need

| # | Decision | Options | Question needed | How to decide |
|---|---|---|---|---|
| 1 | **Type of tool** | Agent tool (edits files and runs commands on its own) or chat / autocomplete | Which mode do you mainly use? What for? | Few people use agent mode → simple, often free tools are enough. Many use it → only tools with agent mode qualify. |
| 2 | **Pricing model** | Free tier, student offer, subscription, pay per use (API), local model | How many hours per week? How often do you hit limits? | Low usage and rarely limited → a free tier is enough. High usage or often limited → student offer or pay per use. |
| 3 | **Same tool for everyone?** | Everyone gets the same tool, or heavy users get a paid tool and the others a free one | Questions 3 and 4, looked at per person instead of as a team average | Usage differs a lot → paid tool only for heavy users, free tool for the rest. Usage is similar → the same tool for everyone. |
| 4 | **Local models possible?** | Local (e.g. Ollama) or cloud | Which computer do you use (model, RAM)? | Without powerful machines the local option is ruled out. |
| 5 | **Knock-out criteria** | Which tools drop out of the comparison | What is indispensable for you? | A tool without this feature is ruled out, however cheap it is. |

## Survey

1. Which mode did you mainly use? (agent / chat / autocomplete)
2. What did you use AI for? (multiple choice: writing code, explaining code, debugging, refactoring, documentation, Git / pull requests, research)
3. Roughly how many hours per week did you use AI? (0–2 / 2–5 / 5–10 / more than 10)
4. How often did a limit or waiting time slow you down? (never / rarely / often)
5. What is indispensable for you? (open)
6. Which computer do you use? (e.g. MacBook with M2, 16 GB RAM)

## Results

6 of 8 team members answered.

| # | Question | Answers |
|---|---|---|
| 1 | Mode | agent: 6 |
| 2 | Used for | writing code: 6, explaining code: 6, documentation: 6, debugging: 5, research: 5, refactoring: 2, Git / pull requests: 1 |
| 3 | Hours per week | 5–10: 6 |
| 4 | Slowed down by limits | never: 3, rarely: 3 |
| 5 | Indispensable | agent mode: 2 |
| 6 | Computer | MacBook M1, 16 GB RAM: 3; MacBook M1 Pro, 16 GB RAM: 1; Windows laptop with Intel i7, 16 GB RAM: 1; Windows laptop with AMD Ryzen 5000, 8 GB RAM: 1 |

## Decisions

| # | Decision | Result |
|---|---|---|
| 1 | **Type of tool** | Everyone mainly uses agent mode, so only tools with agent mode qualify. |
| 2 | **Pricing model** | Usage is medium to high (5–10 hours per week in agent mode). Limits were rarely hit, but with the current paid subscription. A free tier is therefore probably not enough; we need an offer with a similar volume. The comparison checks this. |
| 3 | **Same tool for everyone?** | Usage is similar for everyone, so everyone gets the same tool. |
| 4 | **Local models possible?** | No for agent mode. All computers have at most 16 GB RAM and no strong graphics card. Models that work reliably in agent mode need more memory and would run slowly on this hardware. Local models are at most an addition for simple tasks. |
| 5 | **Knock-out criteria** | Agent mode. |

The main uses are writing and explaining code, documentation, debugging and research. Refactoring and Git are rarely used and do not decide the choice.

This leaves cloud options with agent mode, one tool for the whole team:

- a subscription or student offer
- pay per token with a cheap model
- an own server, only if HTWG offers GPU resources

## Requirements list

| Priority | Requirement | Source |
|---|---|---|
| Must-have | Agent mode | Survey questions 1 and 5 |
| Must-have | Context of at least 64K tokens | [Context](5-models.md#how-much-context-for-what) |
| Must-have | Usable with VS Code or in the terminal | Our development environment |
| Must-have | Costs within the budget | [Costs and budget](#costs-and-budget) |
| Must-have | Our code is not used for training | Data privacy |
| Nice-to-have | Free choice of model | Flexibility for later changes |
| Nice-to-have | Autocomplete | Survey question 1: not the main mode |
| Nice-to-have | Local model for simple questions | [Our hardware](8-our-hardware.md#recommendation) |

## Costs and budget

- **Current cost:** about 24.50 € per person per month for the Claude subscription. For 8 team members, this is about 196 € per month.
- **Budget:** still to be set together with the Product Owner.

## Proposed candidates

| # | Candidate | Operating model | Notes |
|---|---|---|---|
| 1 | [GitHub Copilot Student](2-tools.md#vs-code-extensions) | Student offer | Free for verified students; new sign-ups were paused in April 2026 |
| 2 | OpenCode or Cline with an open-weight model through an API, e.g. [OpenRouter](https://openrouter.ai/models) | [Pay per token](3-operating-models.md#pay-per-token-api) | Costs depend on usage |
| 3 | HTWG server with 2 × A100 and an open-weight coding model through vLLM | [Own server](3-operating-models.md#own-server) | No token costs; access still to be clarified, see [Possible server](8-our-hardware.md#possible-server) |
| 4 | [Gemini CLI](2-tools.md#terminal-tools) (optional) | Free tier | 1,000 requests per day; free baseline for the comparison |
