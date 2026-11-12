# Reflection Brief — Harness Engineering Capstone

**Name: Vishal Bhakuni**
**Date: October 20, 2026**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): claude-haiku-4-5-20251001
- OS / Python: Windows (Win32 / MINGW64) / Python 3.14.2
- Approx. API spend: ~$0.12 (total across active test runs)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

→ 1. Sequence: "stop_reason": "tool_use" $\times 4$ followed by "stop_reason": "end_turn" (as captured in trace_sample.jsonl).

2. Implementation: The continue-vs-stop decision logic resides in claims_intake/loop.py within the run() function. It evaluates response.stop_reason inside a while True: loop: when the stop reason is "tool_use", the harness executes the requested tool calls and feeds the results back into the model context to continue iterating; when it hits "end_turn", the loop terminates and returns a structured FinalState containing the final routing or escalation outcome.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

→ 1. Check: test_antipatterns.py in System 1 checks for brittle anti-patterns such as hard-coded iteration caps using integer literals or relying on fragile natural-language string/regex parsing of tool outputs instead of relying on model-driven stop_reason inspection.

2. Impact: If the loop used hard-coded turn counts or string-matching workarounds instead of dynamic stop_reason handling, it would fail to handle variable-length claim flows adaptively (such as claims requiring varying clarification or tool turns like our 5-turn claim_01_kitchen_fire), breaking test suites like test_loop.py (29 passed in test_output.log).

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

→ 1. Prevention: Tools like lookup_policy and record_claim_fact have distinct operational boundaries explicitly defined in their docstrings (e.g., policy retrieval vs. attribute recording).

2. Benefit: A structured error (such as a typed validation exception listing missing fields) lets the harness catch schema violations programmatically rather than handling vague string messages, allowing the agent to self-correct.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

→ 1. Metrics: For claim_01_kitchen_fire, it took 5 turns, 16,752 input tokens, 1,111 output tokens, costing $0.0223 (total run cost for 8 claims: $0.1173 in summary.md).

2. Comparison: Differs based on claim complexity and required clarification turns, optimizing token usage strictly per tool iteration.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

→ 1. Metrics: Baseline tokens: 47,144; Assembled tokens: 20,389; Reduction: 56.75% (budget.json).

2. Dominance: The active segment dominates (19,538 tokens out of 20,389) and is kept byte-exact to ensure zero information loss for ongoing operational troubleshooting.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

→ Rule: Resolved segments are summarized via LLM (resolved_refund: 345 tokens, resolved_subscription: 369 tokens), while active working turns and case facts (case_facts: **135 tokens**, active`: 19,538 tokens) remain pristine and byte-exact.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

→ 1. Comparison: In eval_control.jsonl (where case facts were stripped), Q6 failed/regressed or lost precision compared to passing all 6/6 questions in eval.jsonl.

2. Proof: This proves structured snake_case status tokens reside exclusively in the durable case-facts scratchpad and are lost when summarized away.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

→ 1. Quote: paths: ["**/*.test.ts", "**/*.test.tsx"] (from .claude/rules/tests.md).

2. Rationale: Path-scoped rules apply conventions selectively to matching file patterns across the entire monorepo, avoiding directory bloat and ensuring cross-cutting test standards (like prohibiting UI snapshots and raw setTimeout) enforce automatically wherever co-located test files live without requiring redundant CLAUDE.md files in every subdirectory.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

→ 1. Quote: context: fork and allowed-tools: [Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(gh pr checks:*)] (from .claude/skills/deploy-check/SKILL.md).

2. Rationale: Running forked isolates verbose intermediate tree-walk and check-execution output out of the main session. Read-only allowlists restrict mutations, ensuring safety during pre-deployment validations. Without context: fork, lengthy enumeration logs clutter the parent context; without read-only tools, subagents could inadvertently modify the working tree.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

→ 1. Project-Level Example: Repository-specific modular rules, commands, and skills inside .claude/ (validated as OK by python -m ecommerce_team_config .).

2. User-Level Example: Global developer preferences, personal tool permissions, or custom override skills residing in ~/.claude/ outside the repository version control.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

→ Artifact Citation: Citing shift_output.log from run 20261111, the indexed query defects_since in warm.py queried data/warm.sqlite and returned 17 new defects (Shift C active slice) out of the warm-tier total, pushing historical filtering down to SQL. The model never sees full history because SQL time-window indexing and structured scratchpads replace loading raw historical narrative logs.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

→ Artifact Citation: Governed by recovery.py and validated by test_recovery_decide_truth_table in test_us03_crash_recovery.py (33 passed in test_output.log), the decision rule dictates resuming if manifest age <= 30 minutes, and starting fresh otherwise. A fresh start with an injected summary is more reliable because expired tool leases and asynchronous state mutations in old API histories cause validation failures.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

→ 1. Size: 980 bytes (hot_state.json).

2. Importance: Prevents token bloat, latency spikes, and escalating API costs across thousands of automated shift invocations.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.

→ Model Layer: retail_context/client.py (handles Anthropic API calls).

→ Harness Layer: retail_context/assemble.py (enforces structured context assembly and layout).

→ Orchestration Layer: retail_context/run.py (manages execution flow, logging, and token accounting).

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?

→ Artifact Citation: Guaranteed in code: Atomic write via temporary file replacement (tempfile + os.replace in System 4's state.py, verified by test_hotstate_atomic_write in test_us01_tiered_state.py). Guided by prompt: Extracting 12 required JSON keys from unstructured transcript dialogue. Code enforcement is right for security, atomic persistence invariants, and budgeting; prompt guidance is right for semantic extraction and natural language summarization.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?

→ 1. Intra-session (System 2): Compresses 47,144 baseline tokens down to 20,389 tokens.

2. Cross-session (System 4): Persists compact state (hot_state.json at 980 Bytes). Both apply information condensation, but one uses token reduction and the other uses state serialization.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

→ Artifact Citation: Unit tests like test_pruner.py (verified in System 2's 30 passed test suite output) guarantee that missing fields raise PrunerMissingFieldError rather than silently propagating nulls. A single successful run only tests happy paths; edge-case negative tests ensure graceful failure under corrupt upstream tool payloads before production deployment.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

→ System 4: In System 4, forked investigations use fork_for_hypothesis in fork.py to duplicate active session state and scratchpads into an isolated branch without mutating the base state (verified by test_fork_for_hypothesis_copies_state_without_mutating_base in test_us04_fork_scratchpad.py). Validated findings merge back via non-destructive append (merge_findings). The blast radius of agent misbehavior is strictly contained by this copy-on-write state separation, and the kill switch is the comprehensive test suite combined with atomic persistence guards.
---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)

→ Artifact Citation: On the first test run in Windows PowerShell (run log error: [WinError 206] The filename or extension is too long), passing full transcripts via subprocess arguments failed. This was fixed by setting $env:ANTHROPIC_API_KEY to route calls natively through the Anthropic Python SDK client.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

→ Artifact Citation: Grounded in observing pre-segmented transcript boundaries in System 2 (assemble.py), I would implement dynamic semantic chunking based on topic shift detection to automatically identify when a customer transitions between support issues without requiring manual transcript fixtures.
