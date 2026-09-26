# Reflection Brief — Harness Engineering Capstone

**Name:** Sarthak Barke

**Date:** 2026-09-25

## Environment

- Model(s): Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) for System 1; System 2 used the configured Anthropic model; Systems 3–4 were primarily local/CLI validation and recorded-response execution.
- OS / Python: Linux/WSL environment; Python 3.13.0.
- Approx. API spend: System 1 estimated total cost **$0.1096**; System 2 API usage is recorded in `budget.json`; Systems 3–4 used local validation/recorded responses.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.**

→ In `evidence/system1/20260925_045024/traces/claim_04_neighbor_injury.jsonl`, the `stop_reason` sequence is `tool_use → tool_use → tool_use → tool_use → end_turn`. The continue-vs-stop decision is implemented in `claims_intake/loop.py`, function `run()`. When Claude returns `tool_use`, the loop executes the requested tool and continues; when Claude returns `end_turn`, the loop terminates. This makes termination depend on the model/tool protocol rather than a fixed number of turns.

2. **Anti-pattern.**

→ The anti-pattern is a **hard-coded integer iteration cap**, checked by `test_no_integer_literal_iteration_cap_in_loop`. A fixed turn limit could terminate a legitimate claim before the required facts, classification, or routing decision had been completed. `claim_04_neighbor_injury` required **5 turns**, so an unnecessarily low cap could have stopped it before routing. The System 1 test suite passed **29 tests**.

3. **Tool design.**

→ Two related claim tools are fact recording and claim classification because both operate on information from the same claim. Their structured descriptions and schemas separate their responsibilities: fact recording stores extracted claim information, while classification determines the claim type/severity from the available facts. A structured tool error preserves machine-readable error information so the loop can recognize a failed tool operation and decide whether another action is needed, whereas a generic string provides less reliable information for recovery. The relevant behavior is covered by the System 1 test suite and the traces under `evidence/system1/20260925_045024/traces/`.

4. **Your numbers.**

→ `claim_04_neighbor_injury` used **5 turns** and had an estimated cost of **$0.0215**, according to `evidence/system1/20260925_045024/summary.md`. The complete run processed **8 fixtures** with an estimated total cost of **$0.1096**. The live turn count differs from a simple README example because the actual Claude response determines how many `tool_use` iterations are required before `end_turn`.

### System 2 — Context strategy

5. **The reduction.**

→ `evidence/system2/20260925-054053/budget.json` records **38,708 baseline tokens**, **16,885 assembled tokens**, and a **56.38% reduction**. The `active` section dominates the assembled context at **15,789 tokens**. It is kept largely verbatim because it contains information relevant to the unresolved customer issue and therefore directly affects the next answer.

6. **Summarize vs preserve.**

→ The strategy summarizes resolved historical material while preserving current actionable information. The measured sections are `case_facts` **204 tokens**, `resolved_refund` **393 tokens**, `resolved_subscription` **517 tokens**, and `active` **15,789 tokens**. Resolved sections can be compressed because their issues are already settled, while active information is retained because changing or losing details could affect the current response. Overall, the context was reduced from **38,708 to 16,885 tokens**.

7. **Facts block.**

→ `evidence/system2/20260925-054053/eval.jsonl` achieved **6/6 passed**. In `eval_control.jsonl`, Q1 was an unexpected pass while **Q6 failed**. Q6 asks for the exact structured status `in_progress`, showing that the controlled context assembly preserved an answer-critical fact that the uncontrolled control condition did not reliably preserve. This demonstrates the value of targeted context assembly rather than simply reducing context without protecting important facts.

### System 3 — Claude Code config

8. **Path-scoped rules.**

→ `.claude/rules/api.md` uses the path scope `paths: - "src/api/**/*"`. Path-scoped rules apply based on the files being edited rather than requiring a separate `CLAUDE.md` in every directory. This is useful for cross-cutting conventions because the same rule can selectively apply to matching files throughout the repository. The System 3 validator returned **`OK`** and the test suite passed **35 tests**.

9. **Forked skill.**

→ `.claude/skills/deploy-check/SKILL.md` specifies `context: fork` and an `allowed-tools` list restricted to read-oriented operations. The fork isolates deployment-check reasoning from the main conversation context, while the read-only allowlist prevents unnecessary repository modifications. Without the fork, deployment-check context could pollute the main task; without the restricted tools, the validation skill would have broader permissions than necessary. The configuration was validated by the **`OK`** validator result and **35 passing tests**.

10. **Scope.**

→ Project-level configuration is represented by the repository's `CLAUDE.md` and `.claude/` directory, including `.claude/rules/`, `.claude/commands/`, and `.claude/skills/`. A user-level example is `~/.claude/CLAUDE.md`, which represents personal Claude Code instructions outside the repository. The project validator returned **`OK`**, and the System 3 test suite recorded **35 passed**.

### System 4 — Orchestration

11. **Push work down.**

→ `shift_monitor/warm.py` implements `defects_since()` using the indexed SQL query `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`. The warm database has indexes including `idx_defects_ts` and `idx_defects_shift_ts`. The fixture/test verifies **17 April rows** for the relevant April query, rather than passing the complete historical defect set to the model. SQL performs deterministic filtering first, so the model receives only the relevant defect slice instead of the full warm-tier history.

12. **Crash recovery.**

→ `shift_monitor/recovery.py` uses `STALE_RESUME_THRESHOLD_MINUTES = 30`. If incomplete state is recent enough, the system resumes it; if it is older than 30 minutes, the system starts fresh. A fresh start with an injected summary can be more reliable than resuming stale scratch state because stale intermediate state may contain outdated assumptions, while the summary re-establishes the current operational context without replaying unnecessary history. System 4's test suite passed **33 tests**.

13. **Small state.**

→ `evidence/hot_state.json` is **643 bytes**, compared with the project's **5,120-byte** hot-state budget. The bound matters because the system is intended to run repeatedly across shifts. Without a bounded hot state, persistent context could grow indefinitely, increasing token usage, latency, cost, and the amount of stale information carried into future runs. The System 4 suite passed **33 tests**.

---

## Part 2 — Synthesis

14. **Three layers.**

→ **Model:** `claims_intake/loop.py` and `evidence/system1/20260925_045024/traces/claim_04_neighbor_injury.jsonl` show Claude responses driving tool execution through `stop_reason`.

→ **Harness:** `CLAUDE.md`, `.claude/rules/`, and `.claude/skills/deploy-check/SKILL.md` define repository instructions, scoped rules, commands, and permission boundaries around model work.

→ **Orchestration:** `shift_monitor/pipeline.py`, `shift_monitor/warm.py`, `shift_monitor/recovery.py`, and `shift_monitor/fork.py` coordinate model calls with SQL retrieval, persistent state, recovery, and isolated hypothesis exploration.

15. **Deterministic vs prompt.**

→ A deterministic enforcement example is the **read-only `allowed-tools` list** in `.claude/skills/deploy-check/SKILL.md`; the configuration restricts which tools the skill may use. A prompt-guided example is System 1's tool descriptions that guide Claude toward policy lookup, fact recording, classification, and routing. Deterministic controls are appropriate for permissions, schemas, budgets, and safety boundaries, while prompts are appropriate for reasoning, prioritization, and choosing among already-permitted actions.

16. **Context, two faces.**

→ System 2 reduced **38,708 tokens to 16,885**, a **56.38% reduction**, while preserving active information. System 4 used bounded persistent state with `hot_state.json` at **643 bytes** and SQL filtering to retrieve only the relevant defect slice. System 2 manages context within a session through summarization and selective assembly, while System 4 manages context across shifts through persistent state, database filtering, and recovery. Both use the same principle: retain context that is useful for the next decision and avoid repeatedly carrying unnecessary history.

17. **Reliability you can't see in one run.**

→ `test_no_integer_literal_iteration_cap_in_loop` guarantees through automated inspection that the agentic loop does not contain a hidden hard-coded iteration limit. A single successful runtime execution would not reveal this problem because the claim might finish before the hidden limit was reached. This matters because the system should remain correct for claims requiring more tool-use iterations than the specific run happened to require. The System 1 suite passed **29 tests**.

18. **Blast radius.**

→ System 4 has a potentially broad blast radius because it is designed to run repeatedly across shifts and maintain persistent state. Its controls include SQL-side filtering, a bounded **643-byte** hot state, scratchpad persistence, recovery rules, and isolated hypothesis forks, while the `shift-monitor run-shift` CLI provides a discrete operational boundary. The operational kill switch is to stop or disable the scheduled/manual `shift-monitor` invocation before another shift run occurs, preventing additional state changes while the persisted state is inspected. These boundaries reduce the chance that one bad model response compounds indefinitely across future shifts.

---

## Part 3 — Honest assessment

19. **What broke.**

→ System 1 initially failed because `anthropic 0.39.0` was incompatible with `httpx 0.28.1`, producing `Client.__init__() got an unexpected keyword argument 'proxies'`. I fixed the environment by installing `httpx<0.28`, which resulted in `httpx 0.27.2`. I did not modify the project source code for this fix. After the dependency correction, the System 1 suite passed **29/29** and the live run completed successfully.

20. **What you'd change.**

→ I would make dependency compatibility constraints explicit earlier so that a clean environment cannot reproduce the `anthropic`/`httpx` mismatch. The observed failure was environmental rather than an application-code failure, but it prevented the live System 1 run until the dependency versions were compatible. More generally, the four systems showed that reliable agentic engineering benefits from combining model behavior with deterministic tool contracts, testable harness configuration, bounded context, database-side filtering, and persistent-state controls.