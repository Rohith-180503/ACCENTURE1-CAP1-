# Harness Engineering Capstone Reflection Brief

**Name:** NIDUMOLU BALA ROHITH 
**Run date:** 2026-10-01  
**Environment:** Windows, Python 3.14.6

## Verification Status

The reference implementations were left unchanged. Each system has its own virtual environment. System 1 passed 29/29 tests; System 3 passed 35/35; System 4 passed 33/33. System 2 collected 30 tests: 28 passed and 2 artifact-dependent checks were skipped because no completed run was produced. These are the counts observed in this checkout; some project text lists inconsistent alternative counts.

System 3's config validator returned `OK`. System 4 completed an offline recorded-response run. Systems 1 and 2 lack successful end-to-end run artifacts: the configured API gateway returned an authentication/credit error, and the installed Claude CLI reported that it was not logged in. The attempt logs are preserved under the evidence folder. No reference outputs were presented as learner runs.

## 1. Agentic Loop

In `claims_intake.loop.run()` ([loop.py](Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/loop.py)), the loop returns only on `stop_reason == "end_turn"`, appends tool results and continues on `tool_use`, and raises on any other stop reason. `Budget` supplies token and wall-clock limits rather than acting as a fixed-turn completion rule. This avoids inferring completion from assistant prose and avoids silently truncating at a hard-coded iteration cap. The suite passed 29 tests ([System 1 test log](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/01-agentic-loop/setup-and-tests.txt)).

I could not verify a learner-generated claim outcome or turn trace because the API-backed `--all` run did not complete. There is no submitted `summary.md` or trace showing a real `tool_use`-to-`end_turn` sequence yet.

## 2. Conversation Context

The strategy places durable case facts at the top, compresses resolved issue segments, and preserves the active issue verbatim at the end. `case_facts.extract()`, `compressor.compress()`, and `assemble.build()` implement those fidelity choices; `pruner` removes irrelevant fields before they accumulate. This manages context within one conversation, unlike the orchestrator's persistent cross-shift state.

My run loaded the transcript and measured a 47,144-token baseline ([CLI attempt](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/02-context-strategy/cli-run.txt)), but stopped before extraction completed. The SDK attempt also failed before producing a budget or evaluations ([API attempt](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/02-context-strategy/sdk-run-heuristic-count.txt)). The offline test run passed 28 and skipped 2 of 30 tests ([test log](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/02-context-strategy/tests-offline-fallback.txt)). I cannot claim a measured reduction, evaluation pass count, or control regression from this run; the baseline alone does not establish the 50% target.

## 3. Claude Code Harness

The validator returned `OK`, and the suite passed 35/35 tests ([validator and structure](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/03-claude-code-config/validator-and-structure.txt), [test log](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/03-claude-code-config/setup-and-tests.txt)). `CLAUDE.md` imports shared standards, while `.claude/rules` contains API, React, and test rules with glob-scoped frontmatter.

A path-scoped rule is preferable to a directory-level `CLAUDE.md` when the convention applies to matching files across directory boundaries: a glob names the affected surface directly, whereas directory hierarchy scopes by location. The `/deploy-check` skill uses `context: fork` and a read-only tool allowlist. The fork keeps exploratory output out of the main context; the allowlist deterministically rules out writes and deployment side effects. The validator output identifies the command, rule, and skill files it checked.

## 4. Layer 3 Orchestration

The recorded Shift C run queried the warm SQLite store for defects since `2026-04-27T00:00:00Z`; the pipeline reported 2 selected defects ([run log](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/04-orchestrator/recorded-shift-run.txt)). It wrote a 695-byte hot-state file, below the 5 KB budget ([hot_state.json](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/04-orchestrator/hot_state.json)), and appended an entry citing those two defects ([shift_scratchpad.jsonl](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/04-orchestrator/shift_scratchpad.jsonl)). The suite passed 33/33 tests ([test log](Project-Harness%20Engineering%20with%20Claude%20and%20Claude%20Code/evidence/04-orchestrator/setup-and-tests.txt)).

`shift_monitor.recovery.decide()` ([recovery.py](Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/recovery.py)) resumes an incomplete manifest only if its last step is no more than 30 minutes old; empty, complete, or stale manifests start fresh. `fork_for_hypothesis()` ([fork.py](Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/fork.py)) copies base hot state into a hypothesis-specific directory. Each investigation has its own scratchpad, and findings enter the main stream only through explicit `merge_findings()`. The suite tests fork isolation, though this run did not execute a fork.

## 5. Tests and Architecture Synthesis

The orchestrator tests exercise the 30-minute recovery boundary and fork isolation, behaviors a single happy-path shift cannot establish by inspection. The config tests check path matching and reject write-capable skill tools. The captured 33-test and 35-test logs above document those guarantees. System 1's 29 passing tests verify loop and tool contracts, but they do not replace the missing live fixture run.

The Model layer is visible where `claims_intake.loop.run()` requests a model response and branches on its stop reason. The Harness layer is the `CLAUDE.md` import hierarchy plus `.claude/rules`, commands, and skills checked by the config validator. The Orchestration layer is `shift_monitor.pipeline.run_shift()`, which selects state, invokes the model once, and persists updated state.

Deterministic enforcement appears in executable checks: `decide()` implements the 30-minute recovery threshold, and the validator rejects invalid scopes or write-capable tools. Prompt-based guidance appears in system instructions and the deploy-check skill's written checks. Deterministic rules mechanically constrain behavior; prompts guide judgment but need tests or review to establish adherence.

Context management differs by lifetime. System 2's measured baseline is 47,144 tokens, but its assembled count and evaluation were blocked. System 4 kept cross-shift hot state to 695 bytes while selecting 2 recent defects from SQLite. Those measurements illustrate within-conversation versus across-session state, but do not establish the context strategy's reduction target.

## Remaining Evidence

To satisfy all run-artifact criteria, rerun System 1 with a valid Anthropic-compatible credential and complete System 2 after authenticating the Claude CLI or provisioning a gateway that supports the required API requests. Retain System 1's `summary.md` and a trace, plus System 2's `budget.json`, `eval.jsonl`, and `eval_control.jsonl`; then replace the explicitly unverified statements above with measured results.
