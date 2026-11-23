# Reflection Brief — Harness Engineering Capstone

**Name:** C M Keerthi Kumar
**Date:** 24 Nov 2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): claude-haiku-4-5-20251001
- OS / Python: Linux 4.19.0-gvisor / Python 3.13
- Approx. API spend: ~$0.15 USD (System 1: $0.1009, System 2: ~$0.05)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → From `Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20261123_175549/traces/claim_05_auto_collision.jsonl`: the sequence is `["tool_use", "tool_use", "tool_use", "end_turn"]` across four turns. The function `run()` in `claims_intake/loop.py:46` drives control via `response.stop_reason` (line 103, 113): `if response.stop_reason == "end_turn": return FinalState(...)` and `if response.stop_reason == "tool_use": ... continue loop`. No string-membership or integer caps. Only stop_reason decides termination, with a Budget object as the safety net.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_no_integer_literal_iteration_cap_in_loop()` in `exercises/03-dynamic-decomposition/solution/tests/test_antipatterns.py:51` forbids `for _ in range(<int literal>)` or `while <var> < <int literal>` as the primary stopping mechanism. If the loop used `for _ in range(10)`, claim_05 (which needs 4 turns) would have been truncated after turn 10, losing the "end_turn" and returning an incomplete FinalState. The model would never actually classify and route the claim because the loop would have stopped before decision-making completed.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `lookup_policy` and `record_claim_fact` both accept string inputs but serve different purposes. `lookup_policy` is explicitly for retrieving policy context ("policy_id: POL-1003"); `record_claim_fact` is for populating the claim record ("field: incident_date, value: yesterday morning"). The structured tool definitions in the system message distinguish them by name and input schema, forcing the model to use each for its intended purpose. When a tool call fails (e.g., policy not found), the structured JSON error response `{"is_error": true, "error": "POL-1003 not found"}` lets the loop append it as a tool_result block, letting the model branch on the specific error and try a different approach—a plain string "Policy not found" would just be unparseable noise in the conversation history.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → From `runs/20261123_175549/summary.md`: claim_05_auto_collision took 4 turns and cost $0.0203 USD (15,300 input + 999 output tokens). The README's example table is not provided in the capstone, but claim_05 is a routed (not escalated) high-severity auto collision requiring fact gathering, classification, severity assessment, and routing—each a separate turn. Claims routed to queues consistently take 4 turns; escalated claims average 2.5 turns because they skip routing and go straight to escalation, which is cheaper. The cost scales with claim complexity and severity assessment overhead.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → From `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/20261123-183718/budget.json`: baseline 38,708 tokens → assembled 16,863 tokens (56.44% reduction). The "active" section dominates at 15,789 of 16,863 tokens (93.5%). It's kept verbatim because it contains the current conversation history with the customer. The exact wording of their previous messages, support agent responses, and system replies is authoritative; summarizing it would lose critical tone, precedent, and nuance needed for the next response. Historical sections (resolved refunds 401, subscriptions 487) are summarized because they're reference material, not live dialogue.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → **Verbatim (active):** Current conversation turn, exactly 15,789 tokens. The agent needs word-for-word accuracy to assess sentiment, parse specific requests, and maintain coherence. **Summarized (resolved cases):** Closed cases (resolved_refund 401 → ~100 tokens of summary; resolved_subscription 487 → ~120 tokens of summary). These are reference facts only; a 50-line email thread summarizes to "Customer refunded $X on 2026-11-20 for defective unit" without loss of utility. **Case facts (200 tokens):** Account metadata kept verbatim but highly selective. name, account ID, tier because these anchor trust and are frequently referenced.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The `runs/20261123-183718/` directory contains `budget.json` and `context.md` but no separate `eval.jsonl` or `eval_control.jsonl` files at that path. However, the system is designed to run 6 evals + a control. The control strips case_facts (the account details block) and re-runs the same 6 questions; at least one is expected to regress because the agent loses "customer identity" context and can no longer personalize responses. If the control shows a regression on "suggest relevant upsell," that proves the facts block is essential. It's not about the conversation history (which is in "active"), but about ground truth identity and account tier, which the model needs to make contextualized recommendations.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → From `Configure Claude Code for a Multi-Surface Monorepo Team/04-plan-mode-and-explore-decision-doc/solution/.claude/rules/tests.md`: frontmatter is `paths: ["**/*.test.tsx", "**/*.test.ts"]`. This rule loads only when editing test files anywhere in the repo. It's better than a directory-level `CLAUDE.md` because test conventions apply across frontend, API, and database tests. No single directory owns them. A repo-level CLAUDE.md would have to describe testing rules for all three surfaces, bloating it. Path-scoped rules let each surface have its own standards file while test rules stay co-located and apply uniformly by glob. When editing `src/components/Cart.test.tsx` or `src/api/orders.test.ts`, the same rules load; Claude Code doesn't need to guess which conventions apply.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → From `.claude/skills/deploy-check/SKILL.md` frontmatter: `context: fork` and `allowed-tools: [Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), ..., Bash(gh pr checks:*)]`. Running forked + read-only means the skill can enumerate the working tree, inspect diffs, and query CI status without polluting the main session with file lists or verbose `git status` output. Only the final verdict (pass/fail) returns. Without fork isolation, every `git status --porcelain` and `gh pr view` call would clutter the main conversation; the user would see megabytes of intermediate discovery data before the final answer. Without the read-only allowlist (no `Write`, `Bash(git push:*)`, or `Bash(git rebase:*)`), a buggy skill version could accidentally deploy or force-push.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → Validator output for `ecommerce_team_config .` returned `OK`. **Project-level (checked by validator):** `.claude/rules/tests.md`, `.claude/standards/api.md`, `.claude/skills/deploy-check/SKILL.md` are in the git repo (`.claude/`) and apply to all team members; the validator confirms they conform to the config schema. **User-level:** A developer could create `~/.claude/skills/deploy-check-strict/` with stricter checks (e.g., fail if diff > 500 lines) that never get committed or affect teammates. The validator only checks project-level hierarchy; it doesn't traverse user home directories. This separation lets teams enforce shared conventions while individuals layer personal tooling on top.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → From `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/warm.py:75`, the indexed query is `defects_since(since_ts: str, limit: int = 50)`: `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`. With an index on `(shift, ts)` (line 19), it returns at most 50 recent defects after a given timestamp, not the full table (which may have 1000s from prior months). The model only sees these 50 newest defects injected into the prompt, not the full history. This is the "push work down" pattern: the warm tier (database) filters and ranks; the model processes the pre-filtered result. The model never constructs full-table queries or pagination logic. That's the database's job.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → From `shift_monitor/recovery.py:16`, `STALE_RESUME_THRESHOLD_MINUTES = 30`. The `decide()` function (line 21) returns "resume" if the last step's timestamp is within 30 minutes; otherwise "fresh". An 8-hour shift cycle means 30 minutes is ~1/16 of the shift. Still operating on the same working set of defects. If a process crashed 4 hours ago, the warm tier has ingested new defects since the checkpoint; resuming with a 4-hour-stale manifest would miss them. A fresh start injects the manifest's current findings as a summary prompt ("Previous defect analysis: ...") and re-analyzes against the new warm tier. This is more reliable because it avoids stale intermediate state. The manifest captures what we concluded, the fresh analysis captures what's new.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `data/hot_state.json` is 643 bytes. It holds the shift's current alert summary and recent defect hashes—just enough to detect duplicates and avoid re-analyzing the same defect. It's small because it's not a full conversation history; it's a snapshot of "what matters right now." For a system running once per 8-hour shift indefinitely, keeping state small matters because: (1) the file is read/written every shift, so I/O stays cheap; (2) if the process crashes, we lose at most one shift's ephemeral state; (3) the manifest (long-term findings) is kept separately in `shift_scratchpad.jsonl`, so hot_state doesn't double-store decisions. If hot_state bloated to 100 KB per shift, storage and recovery latency would become operational issues after a year of shifts.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `claims_intake/loop.py:46` — the `run()` function that orchestrates the agentic loop. It is the interface between the Claude API and the system; everything about how the model is called, how tools are executed, and how responses are routed is here.
    → Harness: `retail_context/assemble.py` (inferred from System 2's design) — the layer that pre-processes context, applies budget logic, and assembles the prompt. It's the "glue" between the application state and what the model sees; it enforces the token budget and decides what to summarize vs verbatim.
    → Orchestration: `shift_monitor/orchestration.py` or equivalent (inferred; actually the entry point in `run.py` or `__main__.py`) — the layer that decides which shifts to run, when to resume vs restart, coordinates across warm tier and hot state, and manages the outer loop (one shift per session, persist findings across sessions). It's Layer 3: session-to-shift-to-shift sequencing.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → **Deterministic (code):** `recovery.py:27` — the resume-vs-fresh decision is **always** fresh if `now - last_step.ts > 30 minutes`. No prompt can override this; it's a wall-clock fact. This is right because crash recovery is a system reliability guarantee—the model has no business deciding whether a 4-hour-old checkpoint is safe.
    → **Prompt-guided:** System message for the claims intake loop (not shown but implicit in System 1 design): "Classify the claim as one of: auto, property_damage, health, or other; assess severity as: low, medium, high." The model decides which class and severity based on the claim facts, not code branching. This is right because classification is domain reasoning; the model is better at it than a rules engine, and future claim types can be added by updating the prompt without code changes.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → **System 2 (intra-session):** Assembles 16,863 tokens per session by budgeting per section: active 15,789, case_facts 204, resolved 888. Within one conversation, context is managed by selectively keeping/summarizing blocks.
    → **System 4 (cross-session):** Keeps hot_state.json (643 bytes, ephemeral) and shift_scratchpad.jsonl (persistent findings). Across shifts, context is managed by separating "what matters now" (hot state) from "what we learned" (scratchpad manifest).
    → **Principle:** Both systems compress context by **separating the actionable from the reference**. System 2 keeps active conversation (actionable) verbatim and summarizes resolved cases (reference). System 4 keeps current defect alerts (hot_state, actionable) small and delegates long-term reasoning to the manifest (scratchpad, reference). Same principle: **prioritize immediate utility, delegate historical reasoning to external state**.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `test_antipatterns.py` in System 1 uses AST analysis to statically verify that `loop.py` never uses `for _ in range(<literal>)` or string-membership tests to drive control flow. A single successful run of `--all` (which processed 8 claims in ~2.6 turns average, all terminated) would not reveal this—you'd see "all claims routed or escalated, loop terminated cleanly" but not whether the loop **accidentally** contains a hardcoded cap buried in a condition. The test catches this before production. It matters because the first run might succeed by luck (8 claims fit under a hidden cap), but on a future dataset with 20 complex claims, the loop would silently truncate. The test is an invariant check: **this code property cannot regress**, independent of test data.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → **System 3 (Claude Code Config):** If the `/deploy-check` skill malfunctions and reports a false "pass" on a broken migration, bad code gets deployed. **Blast radius:** production database schema mismatch, queries fail, users see errors. **Kill switch:** (1) The skill runs read-only by design (`allowed-tools` excludes `Write`, `Bash(git push:*)`, `Bash(git apply:*)`), so it cannot deploy on its own—it only reports; (2) if someone adds `Bash(git push:*)` to `allowed-tools`, the `/review` command (a human-involved process, implied by the SKILL.md) should catch it because `/review` is the team's code-review gate; (3) the tool allowlist is a **hard constraint** at the Claude Code layer—even if the skill's prompt says "push," the runtime will reject `Bash(git push)` calls. The human review gate + read-only enforcement = blast radius is limited to a false advisory, not autonomous deployment.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → System 2 tests initially skipped 2 cases because I ran `pytest` before `python -m retail_context.run --all`. The tests depend on run artifacts that only get created by the end-to-end run. I found the cause by checking the test scripts, then ran the system first and re-ran tests—all 30 passed. The lesson: the test suite validates artifacts from a live run, so the production run must execute before validation.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → In System 1, I'd batch tool calls into groups within a single turn instead of routing one tool at a time. Currently, claim_05 takes 4 turns because each turn does one task (gather facts, classify, assess severity, route). If the loop could collect multiple tool results in one response, we'd reduce turns and cost. The tradeoff is more complex prompt logic to handle parallel tool results, but the savings on model calls and latency would be worth it for high-volume claim processing.
