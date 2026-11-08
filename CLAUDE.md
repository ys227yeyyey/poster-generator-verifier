# PNG Poster Generator–Verifier

## Why
Create a PNG poster on a user-specified theme, with quality assured by a Generator and Verifier scoring loop.

## Language
Everything runs in English: agent prompts, agent replies, user-facing reports, questions to the user, and all text on the posters. If the user supplies THEME / CRITERIA in another language, translate them to English before passing them to the agents.

## Structure
| Role | Entity | Definition |
|---|---|---|
| Orchestrator | Claude Code itself | this file |
| Agent 1: Generator | `poster_generator` | `.claude/agents/poster_generator.md` |
| Agent 2: Verifier | `poster_verifier` | `.claude/agents/poster_verifier.md` |

**Communication (specific to this project)**: comm.md / agent-log are not used. This is an exception to the global rule "via comm.md". The Orchestrator passes input through the launch prompt of the Agent tool, receives the subagent's reply (final message), and hands it to the next agent.

## Parameters
| Item | Handling |
|---|---|
| Deliverable format | **PNG poster (fixed)** |
| Number of scoring criteria | **3 criteria, 5 points each, 15 total (fixed)** |
| THEME (what poster to make) | **Specified by the user on every run** |
| CRITERIA (content of the 3 criteria) | **Specified by the user on every run** |
| PASS_LINE (how a pass is defined, given the 3 fixed criteria × 5 points) | **Asked from the user on every run, right after CRITERIA** (e.g. minimum total, and/or minimum per criterion) |
| MAX_ROUNDS (maximum iterations) | **Specified by the user on every run** |

## Workflow (run by the Orchestrator)
0. **Check parameters**: Read THEME, CRITERIA (3 items), PASS_LINE, and MAX_ROUNDS from the request. If any item is missing, ask the user via `AskUserQuestion` (do not fill in by guessing). Also ask if CRITERIA is not exactly 3 items. Ask in this order: CRITERIA first, then PASS_LINE (taking the confirmed 3 criteria into account, e.g. offer total-only vs. total + per-criterion minimum), then MAX_ROUNDS. PASS_LINE is always asked, even if it was the same in a previous run.
1. Issue a `RUN_ID` (`date '+%Y%m%d_%H%M%S'`). `ROUND = 1`.
2. Launch `poster_generator` with the Agent tool. Pass `RUN_ID / ROUND / THEME / CRITERIA` in the prompt (for ROUND≥2, also pass `PREV_HTML` and the full previous Verifier reply as `FEEDBACK`).
3. Confirm the `PNG` path in the reply exists.
4. Launch `poster_verifier`. Pass `ROUND / THEME / CRITERIA / PASS_LINE / PNG / HTML` in the prompt.
5. Check `VERDICT` in the reply:
   - `PASS` → go to 6
   - `FAIL` and `ROUND < MAX_ROUNDS` → `ROUND += 1` and return to 2
   - `FAIL` and `ROUND = MAX_ROUNDS` → adopt the highest-scoring round and go to 6
6. Report to the user, in English: the adopted PNG path, pass/fail, and the score progression per round.
- If a subagent reply is `ESCALATE:`, convert it to `AskUserQuestion`, present it to the user, and relaunch the same agent with the answer attached.

## Agent I/O
| Agent | Input (launch prompt) | Output (reply) |
|---|---|---|
| poster_generator | `RUN_ID / ROUND / THEME / CRITERIA`; for ROUND≥2 also `PREV_HTML / FEEDBACK` | `ROUND / PNG (absolute path) / HTML (absolute path) / CHANGES` (text) |
| poster_verifier | `ROUND / THEME / CRITERIA / PASS_LINE / PNG / HTML` | `ROUND / SCORES / TOTAL / VERDICT / FEEDBACK` (text) |

## File structure
```
.
├── CLAUDE.md
├── .claude/agents/
│   ├── poster_generator.md
│   └── poster_verifier.md
├── scripts/render.sh             # HTML → PNG (headless Chrome)
└── outputs/{RUN_ID}/round_{n}/poster.{png,html}
```
