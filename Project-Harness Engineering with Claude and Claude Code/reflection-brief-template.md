# Capstone Reflection Brief — Harness Engineering with Claude and Claude Code

**Author:** NIDUMOLU BALA ROHITH  
**Date:** 2026-10-02  
**Environment:** Windows 11 (build 26100), Python 3.14.6 / Linux (Python 3.13.0)  

---

## Executive Summary & Verification Matrix

All four production systems from the course have been stood up, verified against their automated test suites, executed against domain fixtures, and documented with empirical evidence.

| System | Focus Area | Collected Tests | Passing Tests | Status | Key Evidence Files |
|---|---|---|---|---|---|
| **System 1** | Stop-reason-driven loop & tool design | 29 | 29 | **PASS** | `evidence/01-agentic-loop/test_output.txt`, `summary.md`, `traces/claim_02_stolen_bike.jsonl` |
| **System 2** | Context strategy & token reduction | 30 | 30 | **PASS** | `evidence/02-context-strategy/test_output.txt`, `budget.json`, `eval.jsonl`, `eval_control.jsonl` |
| **System 3** | Claude Code monorepo configuration | 35 | 35 | **PASS** | `evidence/03-claude-code-config/test_output.txt`, `validator_output.txt`, `CLAUDE.md`, `.claude/` |
| **System 4** | Layer 3 multi-shift orchestration | 33 | 33 | **PASS** | `evidence/04-orchestrator/test_output.txt`, `run_shift_output.txt`, `hot_state.json`, `shift_scratchpad.jsonl` |
| **Total** | **All Four Systems** | **127** | **127** | **PASS** | Full suite verified (29 + 30 + 35 + 33) |

> **Test Suite Accounting:** As detailed in `TEST_MAPPING.md`, System 2's 30 tests comprise the 17 core context-strategy tests plus 13 supporting tests; System 4's 33 items correspond to 28 test functions where `test_recovery_decide_truth_table` is 6-way parametrized.

---

## Part 1 — Per-System Analysis

### System 1 — Claims Intake Agent (Agentic Loop & Tool Design)

#### 1. Loop Control Mechanics
- **Control & Termination Function:** Loop termination is governed deterministically in `claims_intake/loop.py` inside the `run()` function. Control flow branches strictly on `response.stop_reason`:
  - `response.stop_reason == "tool_use"`: The loop dispatches tool calls via `tool_executor()`, appends tool result blocks to the message history, evaluates safety budgets, and loops back for the next model turn.
  - `response.stop_reason == "end_turn"`: The loop immediately exits and returns `FinalState`.
  - Any unexpected stop reason (e.g., `"max_tokens"`) raises `UnexpectedStopReason`.
- **Trace Citation:** In `evidence/01-agentic-loop/traces/claim_02_stolen_bike.jsonl`, the turn progression demonstrates this exact lifecycle:
  - Turn 1: `stop_reason: "tool_use"` (`lookup_policy` for `POL-1007`)
  - Turn 2: `stop_reason: "tool_use"` (`record_claim_fact` for date, stolen items, damage amount)
  - Turn 3: `stop_reason: "tool_use"` (`classify_claim` as theft, `assess_severity` as low)
  - Turn 4: `stop_reason: "tool_use"` (`route_to_adjuster` to theft queue)
  - Turn 5: `stop_reason: "end_turn"` (no tools called; loop cleanly terminates).
- **Escalation Path:** In `evidence/01-agentic-loop/traces/claim_06_low_confidence_escalation.jsonl`, multi-asset storm damage triggers `request_clarification` on Turn 2, followed by `escalate_to_human` on Turn 4, and concludes with `stop_reason: "end_turn"` on Turn 5.

#### 2. Anti-Patterns Avoided
- **Target Anti-Patterns:** The loop avoids **string-matching on assistant prose** (e.g., searching for `"DONE"` or `"ROUTED"`) and avoids **fixed-count iteration loops** (e.g., `for _ in range(5)`).
- **Architectural Rationale:** Assistant prose is non-deterministic; subtle phrasing variations ("I have finished routing" vs. "Claim routing completed") break text-parsing logic. Hardcoded iteration limits arbitrarily terminate complex claims requiring clarification or multi-policy lookups. Relying strictly on API-level `stop_reason` guarantees deterministic control flow.
- **Verification:** `tests/test_antipatterns.py` inspects AST nodes to enforce that no module in `claims_intake/` matches assistant text for termination or hard-codes loop caps (29/29 tests pass; `evidence/01-agentic-loop/test_output.txt`).

#### 3. Tool Design & Contract Enforcement
- **Disambiguation at Schema Level:** In `claims_intake/tools.py`, `classify_claim` and `assess_severity` require reasoning strings alongside categorical values. Schema enums prevent cross-talk (`claim_type: ["auto", "property_damage", "liability", "theft"]` vs. `severity: ["low", "medium", "high"]`).
- **Structured Error Handling:** When tools encounter invalid parameters or missing prerequisites, the dispatcher returns structured JSON payloads:
  `{"is_error": True, "error_category": "<category>", "is_retryable": <bool>, "message": "<desc>"}`.
  This allows the model to differentiate transient recoverable errors from invariant schema violations and self-correct on the subsequent turn.

#### 4. Empirical Run Numbers & Costs
- **Run Metrics:** `evidence/01-agentic-loop/summary.md` records 8 claims processed end-to-end:
  - `claim_02_stolen_bike`: 5 turns, 17,239 input tokens, 819 output tokens, cost $0.0213, routed to `theft`.
  - `claim_04_neighbor_injury`: 5 turns, 17,375 input tokens, 849 output tokens, cost $0.0216, routed to `liability` (high severity).
  - `claim_05_auto_collision`: 4 turns, 14,940 input tokens, 941 output tokens, cost $0.0196, routed to `auto` (high severity).
  - Total Haiku run cost: **$0.1501 USD** across all 8 fixtures.

---

### System 2 — Retail Support Copilot (Context Strategy)

#### 5. Context Reduction Ratios
- **Empirical Measurements:** In `evidence/02-context-strategy/budget.json`:
  - **Baseline Tokens:** **38,708** (48 raw conversation turns)
  - **Assembled Context:** **16,889**
  - **Reduction:** **56.37%** (saving 21,819 tokens, surpassing the ≥50% rubric target).
- **Token Distribution:**
  - `case_facts`: **204 tokens** (1.2%)
  - `resolved_refund`: **399 tokens** (2.4%)
  - `resolved_subscription`: **515 tokens** (3.0%)
  - `active` thread: **15,789 tokens** (93.4%).
- **Why Active Dominates:** The active thread (Turns 29–48) concerns an unresolved `AVS_MISMATCH` payment card failure. Every customer detail and gateway message is decision-bearing for in-flight support. Settled issues (refunds and cancellations) are historical and tolerate compression.

#### 6. Summarization vs. Preservation Rules
- **Preserved Verbatim:** Structured `# Case Facts` at the top and the active conversation segment (Turns 29–48) byte-for-byte.
- **Summarized:** Inactive historical turns (Turns 1–14 compressed from 12,334 to 399 tokens; Turns 15–28 compressed from 11,475 to 515 tokens).
- **Rationale:** Compressing resolved threads eliminates distraction and frees context space, while pinning structured facts at the top guarantees accurate entity and status recall.

#### 7. Evaluation & Control Experiment
- **Full Run Evals (6/6 Passed):** In `evidence/02-context-strategy/eval.jsonl`:
  - Q1: Refund amount for ORD-77310 → `$22.14` (**passed**)
  - Q2: Cancellation reason for subscription → `duplicate` (**passed**)
  - Q3: Payment update failure code → `AVS_MISMATCH` (**passed**)
  - Q4: Last-4 of new card → `7782` (**passed**)
  - Q5: Proration refund received → `prorated` (**passed**)
  - Q6: Structured status of active issue → `in_progress` (**passed**).
- **Control Run Regression:** In `evidence/02-context-strategy/eval_control.jsonl` (with `# Case Facts` stripped):
  - Q1 still passed because `$22.14` was mentioned in the narrative refund summary.
  - **Q6 regressed and failed:** The model stated: *"I don't have access to a case record or structured status token for this active issue... only describe the current state qualitatively"* (**passed: false**).
- **Takeaway:** Natural language summaries introduce semantic drift; structured facts are strictly required for deterministic key-value state tracking.

---

### System 3 — E-Commerce Team (Claude Code Harness Configuration)

#### 8. Path-Scoped Rule Enforcement
- **Glob Scopes:** In `evidence/03-claude-code-config/.claude/rules/`:
  - `api.md`: `paths: "src/api/**/*"`
  - `react.md`: `paths: ["src/components/**/*", "src/pages/**/*"]`
  - `tests.md`: `paths: ["tests/**/*", "**/*.test.*", "**/*.spec.*"]`.
- **Architectural Advantage:** Subdirectory-level `CLAUDE.md` files scatter rules and cause configuration duplication. Path-scoped rules allow cross-cutting patterns (e.g., API contracts or test conventions) to apply globally across file patterns regardless of directory nesting, while keeping governance centralized.

#### 9. Forked-Context Skill & Read-Only Toolset
- **Skill Configuration:** In `evidence/03-claude-code-config/.claude/skills/deploy-check/SKILL.md`:
  - Frontmatter declares: `context: fork`
  - Allowed tools: `allowed-tools: [Read, Grep, Glob, Bash(git status), Bash(git log *), Bash(gh pr view *)]`.
- **Isolation Rationale:** Pre-deployment validation runs extensive tests, diffs, and diagnostics. Forking runs this work in an ephemeral sub-agent context, preventing the developer's main context from being polluted.
- **Safety Boundary:** Restricting tools to read-only commands deterministically prevents the verification skill from modifying files, committing, or deploying code.

#### 10. Scope Governance & Validation
- **Validator Outcome:** `python -m ecommerce_team_config .` reports `OK` with exit code 0 (`evidence/03-claude-code-config/validator_output.txt`).
- **Scope Division:**
  - *Project Scope (Versioned):* Root `CLAUDE.md` (importing standards via `@import`), `.claude/rules/`, `.claude/commands/review.md`, and `.claude/skills/deploy-check/`.
  - *User Scope (Uncommitted):* `~/.claude/CLAUDE.md` and personal scratchpads, keeping developer workstation preferences decoupled from the shared repository.

---

### System 4 — Quality Monitoring (Layer 3 Orchestration)

#### 11. Upstream Data Filtering (Pushing Work to SQL)
- **Targeted Slice:** `shift_monitor/warm.py` executes:
  `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` using index `idx_defects_ts`.
- **Empirical Artifact:** In `evidence/04-orchestrator/run_shift_output.txt`, the pipeline queries only the filtered defect delta rather than dumping all 40 fixture records into the LLM prompt.
- **Efficiency:** SQL pre-filtering prevents database bloat from inflating prompt token counts.

#### 12. Crash Recovery & Staleness Threshold
- **Decision Logic:** `shift_monitor/recovery.py` applies a 30-minute staleness threshold (`STALE_RESUME_THRESHOLD_MINUTES = 30`):
  - Incomplete manifest ≤ 30 minutes old: `"resume"` (continues prior execution).
  - Incomplete manifest > 30 minutes old: `"fresh"` (starts new shift).
  - Empty or completed manifest: `"fresh"`.
- **Why Stale Manifests Restart:** Resuming an hours-old aborted run risks acting on obsolete defect patterns. Starting fresh re-reads the authoritative state from warm storage.
- **Fork Isolation:** `shift_monitor/fork.py` implements `fork_for_hypothesis()`, creating an isolated copy of `hot_state.json` and a private `shift_scratchpad.jsonl`. Competing diagnostic threads explore distinct hypotheses without corrupting the main operational stream.

#### 13. State Space Management & Size Budget
- **State Footprint:** `evidence/04-orchestrator/hot_state.json` measures **695 bytes**, safely under the **5,120-byte (~5 KB)** ceiling enforced by `HotState.write_atomic()`.
- **Perpetual Operation:** Bounding hot state to recent hashes, threshold statuses, and active alerts ensures inference costs remain constant across indefinite multi-shift deployments.

---

## Part 2 — Cross-System Synthesis

### 14. Locating the Three Layers
1. **Model Layer (Probabilistic Reasoning):** Implemented in `claims_intake/system_prompt.py` and `tools.py` (System 1). Defines domain instructions and schemas that the model reasons over.
2. **Harness Layer (Deterministic Session & Tool Boundaries):** Implemented in `CLAUDE.md`, `.claude/rules/`, and `ecommerce_team_config/validator.py` (System 3). Manages imports, tool whitelists, and glob routing.
3. **Orchestration Layer (Lifecycle, State, & Workflow Scheduling):** Implemented in `shift_monitor/pipeline.py`, `warm.py`, and `recovery.py` (System 4). Controls tiered storage, SQL filtering, and crash recovery.

### 15. Deterministic Enforcement vs. Prompt Guidance
- **Deterministic Enforcement:** `HotState.write_atomic()` enforcing the 5,120-byte cap; `recovery.decide()` enforcing the 30-minute boundary; `allowed-tools` read-only whitelist. Hard constraints that throw exceptions upon violation.
- **Prompt Guidance:** `system_prompt.py` guiding how the model reasons about claim severity. Soft instructions where domain judgment and natural language synthesis are appropriate.

### 16. Context Management: Intra-Session vs. Cross-Session
- **Intra-Session (System 2):** Compresses a single active dialogue (38,708 → 16,889 tokens, 56.37% reduction) by summarizing resolved issues while preserving active turns verbatim.
- **Cross-Session (System 4):** Persists state across months of shift runs by offloading history to SQLite and passing only a 695-byte `hot_state.json` between invocations.

### 17. Guarantees of Automated Tests vs. Single Runs
A single run only verifies happy-path execution. Automated test suites guarantee:
- **System 4 (`test_us03_crash_recovery.py`):** Verifies the 30-minute boundary at 29, 30, and 31 minutes, and confirms that atomic `fsync` flushes prevent torn reads during crashes.
- **System 1 (`test_antipatterns.py`):** Uses AST parsing to guarantee that loop code contains no string matching or hard-coded turn counters.
- **System 3 (`test_us02_path_scoped_rules.py`):** Verifies that modifying an API file activates *only* API rules, while test files activate both Test and React rules.

### 18. Blast Radius & Fault Isolation
- **System 1:** Contained by `Budget` (max tokens and wall-clock time) and the dispatcher, preventing arbitrary tool execution.
- **System 3:** The `/deploy-check` skill runs in `context: fork` with read-only tools, preventing file mutations or unintended deployments.
- **System 4:** Invocations are strictly limited to one call per shift; oversized states are rejected before disk writes, protecting warm storage integrity.

---

## Part 3 — Retrospective & Lessons Learned

### 19. Environment Failures Hit & Remediated
1. **HTTPX Keyword Incompatibility in System 1:** `anthropic==0.39.0` passed `proxies=proxies` to HTTPX client constructor. In `httpx>=0.28.0`, `proxies` was removed in favor of `proxy`. Remediated by installing `httpx<0.28` (`httpx==0.27.2`).
2. **Windows Codepage (CP1252) vs. UTF-8 in System 2:** `test_assembled_context_active_segment_byte_exact` failed because em-dashes (`—`) in `context.md` were read via CP1252. Remediated by enabling `PYTHONUTF8=1`.
3. **Artifact-Dependent Tests in System 2:** Running tests without prior run artifacts skipped 2 tests. Resolved by populating completed run artifacts into `runs/20260922-202441` with `PYTHONUTF8=1`, achieving 30/30 passed.

### 20. Architectural Refinement
- **System 2 Refinement:** In the control experiment, Question 1 ($22.14 refund) passed even without `# Case Facts` because the figure was retained in the narrative summary. In production, summarizer prompts should explicitly omit exact dollar amounts and identifiers, forcing the model to rely solely on the canonical `# Case Facts` block. This prevents token redundancy and ensures a single authoritative source of truth.
