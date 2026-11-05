# Reflection Brief — Harness Engineering Capstone

**Name:** NIDUMOLU BALA ROHITH
**Date:** November 5, 2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): `claude-haiku-4-5-20251001` (primary inference engine across Systems 1, 2, and 4)
- OS / Python: Windows 11 Enterprise (x64) / Python 3.14.6
- Approx. API spend: ~$0.19 USD total (~$0.1118 for System 1 live run, ~$0.08 for System 2 live run, $0.00 for System 3 local validator, $0.00 for System 4 recorded response)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In run artifact `runs/20261105_101519/traces/claim_03_water_damage.jsonl`, the turn-by-turn `stop_reason` sequence across 5 turns is:
   ```json
   Turn 1: {"stop_reason": "tool_use", "tool_calls": ["lookup_policy", "record_claim_fact"...]}
   Turn 2: {"stop_reason": "tool_use", "tool_calls": ["request_clarification"]}
   Turn 3: {"stop_reason": "tool_use", "tool_calls": ["record_claim_fact", "classify_claim", "assess_severity"]}
   Turn 4: {"stop_reason": "tool_use", "tool_calls": ["route_to_adjuster"]}
   Turn 5: {"stop_reason": "end_turn", "tool_calls": []}
   ```
   Loop termination is decided in `claims_intake/loop.py` inside the `run()` function (lines 103–132). When `response.stop_reason == "end_turn"`, the function immediately halts execution and returns a `FinalState` dataclass containing the completed transcript and token tallies. If `response.stop_reason == "tool_use"`, the loop executes each requested tool, appends the tool results as a user turn, and issues the next model call; any other stop reason raises an explicit `UnexpectedStopReason` exception.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_antipatterns.py` checks for `test_no_string_membership_against_text_in_loop`, which statically inspects the AST of `loop.py` to prevent string-membership expressions (e.g., `if "ROUTE" in assistant_text`) from governing control flow. If our run used string membership to detect completion, any minor lexical drift by Claude—such as saying "I will now route this claim to the property adjuster" without calling the terminal tool, or acknowledging a policy with "Claim routed to review" in Turn 1—would falsely trigger termination before facts, classifications, or severities were recorded. In our run of `claim_03_water_damage`, this would have halted the agent at Turn 1 or Turn 2 without ever calling `classify_claim` or `route_to_adjuster`, corrupting queue outputs and stranding the claimant.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → The two overlapping tools are `route_to_adjuster` and `escalate_to_human` in `claims_intake/tools.py`, both of which represent terminal routing actions that accept claim summaries and candidate queues. Their descriptions prevent misrouting by establishing crisp, mutually exclusive confidence boundaries: `route_to_adjuster` explicitly states it must be called only when "classification confidence is at least 0.6 and severity has been assessed," whereas `escalate_to_human` is specified for when "classification confidence is below 0.6 even after clarification, or when the claim cannot be routed safely." When `_t_route_to_adjuster` encounters missing prerequisites, it returns a structured JSON error `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "classify_claim must be called before routing"}` rather than a generic string or uncaught exception. This structured payload informs the agent of the exact invariant violation, allowing it in Turn 3 to dynamically invoke `classify_claim` and `assess_severity` before successfully calling `route_to_adjuster` in Turn 4.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → From `runs/20261105_101519/summary.md`, `claim_03_water_damage` took 5 turns, asked 1 clarification, consumed 18,629 input tokens and 1,145 output tokens, took 15.3 seconds of wall-clock time, and cost an estimated $0.0244 USD. In the project README baseline sample, `claim_03_water_damage` completed in 4 turns at an estimated cost of ~$0.018 USD with 0 clarifications. The variance occurred because the live `claude-haiku-4-5-20251001` model in our run detected genuine ambiguity between internal plumbing failure and exterior storm flooding, prompting an extra interactive turn using `request_clarification` ("Do you know what caused the flooding...?") before committing to the property damage classification.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → From artifact `runs/20261105-101519/budget.json`:
   - Baseline tokens: 47,144
   - Assembled tokens: 19,767
   - Reduction percentage: 58.07% (exceeding the ≥50% capstone requirement)
   The submitted `budget.json` sections are Case Facts at 149 tokens, resolved refund at 36 tokens, resolved subscription at 42 tokens, and the active issue at 19,538 tokens. The active section dominates the assembled context and is preserved completely verbatim because the ongoing payment-method dispute contains unresolved customer utterances, verification details, error codes, and temporal state transitions where lossy summarization could introduce hallucinations or discard critical instructions.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The architectural rule is: **Resolved historical segments whose business outcomes are finalized are summarized into concise structured text, while the current active, unresolved segment is preserved byte-exact.** In the submitted `budget.json`, the resolved refund is 36 tokens, the resolved subscription is 42 tokens, and the persistent Case Facts block is 149 tokens; together these compact sections retain the transactional identifiers needed for later questions. The active payment-method update conversation remains at 19,538 tokens, so every live negotiation turn stays available without lossy compression.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In `runs/20261105-101519/eval.jsonl`, all 6 evaluation questions passed (6/6, 100%), with Q6 correctly returning the structured status token derived from `# Case Facts`. In contrast, in `runs/20261105-101519/eval_control.jsonl` (where the `# Case Facts` block was stripped), Question 6 regressed from `PASS` to `FAIL`: instead of providing the exact case status token `AVS_MISMATCH` / structured status, the model fell back to conversational paraphrasing ("Active issue"). This regression empirically proves that the `# Case Facts` section is not redundant fluff; it provides essential, high-density factual anchoring for exact structured fields that narrative dialogue summaries inevitably discard.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → From `.claude/rules/react.md`, the frontmatter specifies:
   ```yaml
   ---
   description: Conventions for React components and pages
   paths:
     - "src/components/**/*"
     - "src/pages/**/*"
   ---
   ```
   Path-scoped rules with glob frontmatter are superior to directory-level `CLAUDE.md` files because modern architectural concerns are cross-cutting rather than strictly hierarchical. For example, testing standards (`.claude/rules/tests.md` targeting `**/*.test.*`) or API conventions span multiple nested directories across `src/components/`, `src/pages/`, and `src/api/`. A directory-level `CLAUDE.md` would require duplicating identical instructions across every individual folder, quickly creating configuration drift, whereas path-scoped rules automatically attach to files matching the glob pattern regardless of directory depth.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → In `.claude/skills/deploy-check/SKILL.md`, lines 4 and 6–16 declare:
   ```yaml
   context: fork
   allowed-tools:
     - Read
     - Grep
     - Glob
     - Bash(git status:*)
     - Bash(git diff:*)
     - Bash(git log:*)
     - Bash(git rev-parse:*)
     - Bash(git ls-files:*)
     - Bash(gh pr view:*)
     - Bash(gh pr checks:*)
   ```
   Running in a forked context isolates verbose intermediate inspection output—hundreds of lines of `git status --porcelain`, diff listings, and PR status checks—within a child sub-agent so that only the final structured pass/fail summary returns to the main conversation. The read-only `allowed-tools` restriction guarantees that pre-deployment checks cannot accidentally modify source code, push untested commits, or trigger remote deployments. Without `context: fork`, massive diagnostic logs would pollute the main session's working context window, triggering premature context compaction and diluting the model's attention.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator (`python -m ecommerce_team_config .`) confirmed the full configuration hierarchy and printed `OK` (exit code 0). 
    - **Project-level scope**: Conventions committed to version control in the repository that apply uniformly to all developers, exemplified by `./CLAUDE.md` and `.claude/standards/api.md` (which enforces API error schemas and HTTP status codes across the engineering team).
    - **User-level scope**: Private developer preferences located in `~/.claude/CLAUDE.md` or personal skill folders (such as a local `~/.claude/commands/morning.md` or personal git commit style preference) that are intentionally excluded from git so they never override shared repository standards.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → From running Shift C against `data/warm.sqlite`, the total defect inventory in the warm tier was 40 defects (`WarmStore.count() == 40`), while the SQL query returned exactly 17 new defects (`shift C: 17 new defects` since `2026-04-01T00:00:00Z`). The query executed was `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` via method `WarmStore.defects_since()`, indexed by `idx_defects_ts` on column `(ts)`. The model never sees the entire historical defect log because feeding hundreds or thousands of historical rows into the prompt causes severe token bloat, increases operational latency and cost, and introduces "lost in the middle" attention degradation; pushing time-boundary filtering down to SQLite ensures the model only reasons over the relevant delta.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In `shift_monitor/recovery.py`, the `decide()` function evaluates incomplete manifest logs against `STALE_RESUME_THRESHOLD_MINUTES = 30`. If the time elapsed between the current time and the last completed step timestamp is within 30 minutes, it returns `"resume"`; if the outage exceeds 30 minutes (or if the manifest has no prior steps or was already completed), it returns `"fresh"`. A fresh start with an injected summary is significantly more reliable than resuming an old session because conversational threads left suspended across shifts accumulate stale working memory, dead references, and hallucinated partial reasoning; starting fresh clears conversational noise while the injected summary provides pristine, validated prior conclusions.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → After processing Shift C, artifact `data/hot_state.json` measured exactly 980 bytes on disk (well below the 5,120-byte / 5 KB budget enforced by `test_hotstate_roundtrip_and_size_budget`). A strict byte budget is critical for a perpetual, shift-based monitoring system because unbounded state growth across 3 shifts per day (1,095 shifts/year) would eventually exceed LLM context windows, slow down prompt ingestion, inflate per-run API costs, and introduce cumulative drift. Limiting `hot_state.json` to 20 recent defect hashes and concise alert summaries forces the orchestrator to offload older telemetry into warm (SQLite) and cold (monthly summaries) storage tiers.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model Layer:** `claims_intake/client.py` (System 1) and `retail_context/client.py` (System 2). This layer handles raw client construction, model identifier bindings (`claude-haiku-4-5-20251001`), authentication headers, and direct invocation of `client.messages.create()`.
    → **Harness Layer:** `claims_intake/loop.py` & `claims_intake/tools.py` (System 1) and `retail_context/assemble.py` (System 2). This layer encapsulates prompt engineering, tool dispatching, structured error formatting (`_err()`), token counting, and context assembly before handing payloads to the model.
    → **Orchestration Layer:** `shift_monitor/pipeline.py` & `shift_monitor/recovery.py` (System 4). This layer sits above individual model conversations, coordinating multi-shift lifecycles, SQLite warm data pre-filtering, crash recovery manifests, and hypothesis session branching across time.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → **Guaranteed in code:** The strict read-only tool allowlist in `deploy-check/SKILL.md` (System 3) and atomic state writes in `shift_monitor/hot.py` (`HotState.save_atomic()`). In `hot.py`, the system writes state to a temporary file, flushes, `os.fsync`s, and replaces atomically, deterministically preventing corrupted state files regardless of what the LLM generates.
    → **Guided by prompt:** The classification confidence guidance in `claims_intake/tools.py` ("If confidence is below 0.6, prefer escalate_to_human") and severity determination rationales.
    → **When each is right:** Deterministic programmatic enforcement is required for hard invariants—safety gates, authentication, file persistence integrity, and security sandboxes where failure cannot be tolerated. Prompt guidance is appropriate for semantic interpretation, fuzzy entity matching, natural language summarization, and domain reasoning where programmatic heuristics are too brittle to capture contextual nuance.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → In System 2 (intra-session context management), the copilot managed context within a single multi-turn dialogue by compressing the transcript from 47,144 baseline tokens down to 19,767 assembled tokens (a 58.07% reduction), isolating key metadata into a 149-token `# Case Facts` table while preserving the 19,538-token active issue verbatim. In System 4 (cross-session context management), the orchestrator managed context across multiple 8-hour manufacturing shifts by querying an indexed SQLite database for only 17 new defects out of 40 total warm defects and maintaining an ultra-compact `hot_state.json` file of only 980 bytes (<5 KB). Both systems implement the same core architectural principle—**protecting model attention and token efficiency by strictly bounding working context to the current operational delta**—but System 2 uses LLM-driven linguistic summarization within an ongoing conversation, whereas System 4 uses tiered storage hierarchies (Hot JSON / Warm SQLite / Cold JSON) and SQL-level push-down filters across operational shifts.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → The crash recovery truth table in `tests/test_us03_crash_recovery.py` (`test_recovery_decide_truth_table`) verifies boundary transitions at 1, 29, 30, and 31 minutes for both complete and incomplete states. A single successful shift run will only ever execute either a clean fresh start or a single resume; it cannot reveal whether a network crash at minute 31 correctly falls back to a fresh session with an injected summary or whether a partial write at minute 29 safely resumes. Validating this matrix in automated tests guarantees that unexpected factory crashes at shift boundaries will recover deterministically without operator intervention or state corruption.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In System 1 (Insurance Claims Intake), the blast radius of an unconstrained or misbehaving agent is erroneous claims routing, improper claims escalation, or infinite loop execution that exhausts API budgets and delays policyholder support. The system implements multi-layered kill switches:
    1. **Token & Time Budgeting:** `claims_intake/budget.py` enforces hard caps on cumulative input tokens and monotonic wall-clock elapsed time, raising `BudgetExceeded` to terminate execution immediately if the agent stalls.
    2. **State Machine Invariants:** `claims_intake/tools.py` checks `session.terminal_called`; calling `route_to_adjuster` or `escalate_to_human` more than once is blocked with a permanent error, preventing duplicate dispatching.
    3. **Mandatory Prerequisite Gates:** In `_t_route_to_adjuster`, if `session.classification` or `session.severity` is missing, the tool refuses to write to `queues/*.jsonl`.
    4. **Audit Trail:** Every turn logs its `stop_reason` and latency to `traces/<claim_id>.jsonl`, enabling immediate post-mortem telemetry.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → During initial setup on Windows 11 with Python 3.14, running `retail_context` tests and the API call initially failed: first, the local environment had inherited an `ANTHROPIC_BASE_URL` pointing to an external OpenRouter proxy (`https://openrouter.ai/api`), which returned HTTP 404 on `messages.count_tokens`; second, printing Claude's Unicode greeting emoji (`👋`) threw a Windows CP-1252 `UnicodeEncodeError`. We resolved both issues by setting `$env:ANTHROPIC_BASE_URL = "https://claude.vocareum.com"`, injecting the Vocareum API key, and forcing UTF-8 I/O encoding via `$env:PYTHONUTF8 = "1"`, after which all 30 tests in System 2 passed and the 6/6 evaluation ran cleanly.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → In System 4's `shift_monitor/hot.py`, `HotState` manages defect deduplication by appending individual defect IDs to a single flat `recent_defect_hashes` list capped at 20 items (`MAX_RECENT_DEFECTS = 20`). In high-defect shifts (such as Shift C which added 17 new defects in one go), this naive FIFO list immediately rolls off 85% of prior defect IDs, destroying longitudinal defect pattern recognition across preceding shifts. If redesigning this system, I would replace the raw ID list with a compact component-level counter map or a rolling Bloom filter: this would track recurrence frequency and defect concentration across dozens of shifts while remaining well within the 5 KB hot state byte budget.
