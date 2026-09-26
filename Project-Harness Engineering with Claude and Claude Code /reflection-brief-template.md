# Reflection Brief — Harness Engineering Capstone

**Name:** Meenakshi Yadav
**Date:** September 26, 2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): Claude 3.5 Haiku / Sonnet 

- OS / Python: Linux (Ubuntu/Debian via Udacity Vocareum) / Python 3.11+.

- Approx. API spend: Sum the estimated costs from your runs.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → For the claim_02_stolen_bike trace, the sequence is "tool_use" -> "tool_use" -> "tool_use" -> "end_turn". The file claims_intake/loop.py manages this control flow in the run() function. It explicitly checks response.stop_reason to decide whether to continue the loop (by executing tools) or terminate the loop and return the final state.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → The tests/test_antipatterns.py file checks for the test_no_string_membership_against_text_in_loop anti-pattern, ensuring the loop doesn't use expressions like "some_token" in <something> to drive control flow. If the loop relied on searching the raw text for a magic string instead of using the API's native stop_reason, it would be brittle; a hallucinated formatting error from the model or a prompt injection from the user could cause the loop to crash or execute unintended actions.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → The route_to_adjuster and escalate_to_human tools both accept summaries and handle claim conclusions, but their descriptions prevent misrouting by explicitly stating when to use each (e.g., route_to_adjuster explicitly requires a classification confidence of "at least 0.6", while escalate_to_human explicitly states to use it when "confidence is below 0.6"). A structured JSON error (like {"is_error": true, "message": "..."}) allows the loop to return the exact failure reason to the model as a tool_result, letting the model dynamically correct its parameters and retry, rather than raising a Python exception that crashes the entire loop.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → For claim_03_water_damage, the loop completed in 4 turns with an estimated cost of $0.0195 (14,123 input tokens and 1,076 output tokens). This differs slightly from static README samples because the agentic loop is dynamic; the model's exact phrasing, rationale length, and decisions (like whether to ask a clarification question) vary slightly per run, which changes the final token count and cost.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → From my budget.json run, the baseline transcript tokens were 38708, and the assembled tokens were 16978, achieving a 56.14% reduction. The "active issue" segment dominates the assembled context budget. We keep it verbatim because the issue is unresolved, meaning every turn-by-turn conversational nuance is potentially decision-load-bearing and requires exact fidelity.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The rule is: resolved issues are summarized because their facts have stabilized and can be condensed without information loss, whereas the active issue is kept byte-exact to maintain conversational coherence for the live problem. In my run, the resolved refund was compressed from 12,334 to 421 tokens, and the resolved subscription from 11,475 to 556 tokens, saving massive context space for the active thread.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → When comparing the results, Q6 (which asks for the structured status of the payment-method update) passed in eval.jsonl but failed in eval_control.jsonl. This proves that the case-facts block is strictly load-bearing for structured data; it acts as a durable scratchpad carrying specific system tokens (like in_progress) that narrative summaries naturally drop.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The api.md rule file uses the glob frontmatter: paths: ["src/api/**/*"]. This is far better than a directory-level CLAUDE.md because it conditionally injects these specific API conventions into the LLM's context only when an API file is being edited. Loading all rules for a whole repo into a root CLAUDE.md wastes token budget and dilutes the model's focus with irrelevant instructions, whereas path-scoped rules keep the context window small, cheap, and highly relevant.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → The skill is configured with context: fork and an allowed-tools: block restricted to read-only tools like - Read, - Grep, and - Bash(git status:*). Running forked buys you context preservation: it keeps verbose intermediate discovery output (like massive git diffs) out of the main session, returning only a clean summary. The read-only allowlist buys safety. Without the fork, your main context window would fill up with useless logs and degrade the model's attention; without the read-only restriction, a hallucinated command could accidentally modify files or push code.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → Project-level scope applies rules and tools to everyone working in the repository by committing them to version control. An example from this config is the shared deploy check skill located in .claude/skills/deploy-check/. User-level scope applies only to an individual developer's machine and is not checked into the repo. An example is creating a personalized, stricter variant of the deploy check located in ~/.claude/skills/deploy-check-strict/ which only affects that specific user.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The SQL query pushes filtering down to the database layer by querying only active or defective records rather than loading the entire historical shift database into the LLM's context window. The specific indexed query utilizes an indexed timestamp or status column on the shift log table. The model never sees the full raw history because the harness pre-aggregates and filters the data through deterministic SQL, feeding the model only the targeted subset needed for evaluation.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → The recovery.py module defines a staleness threshold of STALE_RESUME_THRESHOLD_MINUTES = 30 (roughly 1/16 of an 8-hour shift cycle). A fresh start with an injected summary is sometimes more reliable than resuming because a partial state older than 30 minutes may involve expired backend tool references, stale session IDs, or context desynchronization, whereas starting fresh with a summary provides a clean slate anchored by durable findings.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → The hot state file is kept strictly minimal (typically just a few kilobytes of JSON data tracking the active shift's state). This budget matters profoundly for a system run once per shift indefinitely because unbounded state growth would cause context inflation, leading to linearly increasing API latency and costs, eventually hitting the model's token limit and crashing the automated shift pipeline.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: The core intelligence layer, represented by the Claude Messages API configuration (e.g., model="claude-3-5-sonnet-20241022" in System 1's loop.py)

    → Harness: The programmatic control flow layer bridging the API, such as claims_intake/loop.py which enforces the strict stop_reason-driven state machine , guarantees bounded loops, and prevents raw string matching. 

    → Orchestration: The macro-level state management and persistence layer, such as shift_monitor/recovery.py and manifest structures in System 4, which manage long-running multi-shift memory, crash recovery thresholds, and historical state boundaries.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Guaranteed in code: The allowed-tools allowlist in System 3's /deploy-check skill file (.claude/skills/deploy-check/), which restricts execution strictly to read-only commands (git status, git diff, etc.) preventing any destructive shell actions.
    → Guided by prompt: Summarizing resolved issues versus preserving active threads verbatim in System 2's context strategy.
    → When is each right? Code enforcement is mandatory for security boundaries, invariants, control flow logic, and blast-radius limitations where failure is catastrophic. Prompt guidance is right for fuzzy text synthesis, trade-off evaluation, qualitative reasoning, and summarization where deterministic AST or regex checks cannot anticipate semantic nuances.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages intra-session context via prompt-budget algorithms in budget.json (e.g., reducing 38,708 baseline tokens down to 16,978 tokens, a 56.14% reduction) by dynamically summarizing closed past issues while keeping the active issue verbatim. System 4 manages cross-shift context over longer horizons by querying relational databases (shift_monitor/db.py) and injecting a compacted manifest summary to restart cleanly.

    → Same principle, different mechanism: Both face the finite token window limitation and combat context inflation by offloading cold/historical data (summarizing closed tickets or querying SQL) while retaining high-fidelity hot data (verbatim active issues or recent shift artifacts), but System 2 does it via turn-by-turn text compaction whereas System 4 does it via persistent external storage and state injection.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → Static analysis tests like tests/test_antipatterns.py guarantee that the loop does not use string-membership checks against assistant text to drive control flow. A single successful run might look completely functional even if the code contains fragile string-matching hacks, but a regression test ensures that prompt injections or formatting quirks cannot silently compromise the loop's control-flow integrity in production.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 4 (Multi-Shift Orchestrator). If it misbehaves, the blast radius includes executing erroneous shift escalations, corrupting historical shift state logs, or flooding downstream alerting queues with false-positive quality failures. The kill switch is enforced via the decide() crash-recovery check (shift_monitor/recovery.py), a strict 30-minute staleness threshold, and atomic file-write patterns (manifest.py) that safely abort partial runs and fallback to a clean, isolated state initialization rather than propagating corrupt states.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → When first launching System 1, the initial API execution failed with a authentication error (400 - Invalid API key) due to a missing environmental variable string in the Vocareum workspace. This was fixed by re-exporting the correct active Vocareum API key into the shell environment (export ANTHROPIC_API_KEY="...") and re-running the test suite.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would introduce a centralized, asynchronous telemetry wrapper around all tool executions to log exact latency distributions and token inflations per tool call, allowing us to proactively detect when specific tools (like policy lookups or SQL queries) introduce hidden context bloat before it triggers budget exceptions.
