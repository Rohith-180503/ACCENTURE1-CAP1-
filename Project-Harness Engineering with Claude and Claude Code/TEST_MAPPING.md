# Test Mapping

Explains why the collected test counts differ from the rubric counts (System 2: 30 vs 17, System 4: 33 vs 28).

## System 2: Context strategy (30 collected, rubric 17)

The suite is cumulative across the project's stories. The 17 core tests cover context assembly and evaluation; the other 13 cover supporting modules from earlier stories that assembly depends on.

### Core context-strategy tests (17)

| File | Tests | Rubric requirement covered |
|---|---|---|
| `test_assemble.py` | 3 | Active-segment preservation, section order, no interleaving |
| `test_case_facts.py` | 5 | Case-fact extraction (12 required fields, fixed key order, error on missing fields) |
| `test_compressor.py` | 4 | Summarization; refuses to compress the active segment |
| `test_antipatterns.py` | 5 | Guards: single token counter, byte-exact active segment, budget sections sum consistently |

### Supporting tests (13)

| File | Tests | Role |
|---|---|---|
| `test_pruner.py` | 4 | Tool-output pruning to the contracted field set (under 200 tokens) |
| `test_tokens.py` | 4 | Single canonical token-counting module (used for budget tracking) |
| `test_transcript.py` | 5 | Transcript loading, partition boundaries, canonical token function |

These 13 are needed for correctness: assembly and budget tracking rely on them. They are just not counted in the rubric's 17.

Evidence: `system2-context-strategy/test_output.txt` (all 30 pass; the 17 core tests are the entries from `test_assemble.py`, `test_case_facts.py`, `test_compressor.py` and `test_antipatterns.py`).

## System 4: Orchestrator (33 collected, rubric 28)

The suite has 28 test functions. pytest reports 33 items because `test_recovery_decide_truth_table` is one function parametrized 6 ways (1, 29, 30, 31 minutes incomplete; 1 and 60 minutes complete), which adds 5 extra items (33 - 5 = 28).

| File | Test functions | Rubric requirement covered |
|---|---|---|
| `test_us01_tiered_state.py` | 9 | Tiered state: hot-state size budget, hash cap, atomic write, warm SQLite + index, cold monthly summary, SQL-filtered `defects_since` |
| `test_us02_invocation_pipeline.py` | 6 | Invocation pipeline: SQL passthrough, no Python-side filtering, prompt under 4000 chars, single client call |
| `test_us03_crash_recovery.py` | 9 (14 items) | Crash recovery: manifest fsync, incomplete/complete load, resume-vs-fresh truth table, 30-minute threshold, resumed prompt |
| `test_us04_fork_scratchpad.py` | 4 | Fork/scratchpad: ordering, fork does not mutate base, independent scratchpads, merge without rewriting |
| **Total** | **28 functions** | 33 collected items |
