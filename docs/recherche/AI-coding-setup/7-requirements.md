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
