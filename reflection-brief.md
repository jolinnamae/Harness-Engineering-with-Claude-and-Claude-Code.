# Reflection Brief — Harness Engineering Capstone

**Name:** Jolinna Mae Tibulan  
**Date:** November 16, 2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted.

**Environment**

- Model(s): Claude models provided through the Udacity/Vocareum environment
- OS / Python: Vocareum browser-based Linux workspace / Python 3.13.18
- Approx. API spend: System 1 final run = $0.0961. System 2 used the provided Claude API for its context build and evaluations. System 3 required no API call, and System 4 was run offline using the supplied recorded response.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

In run `20261116_112611`, `runs/20261116_112611/traces/claim_05_auto_collision.jsonl` recorded the sequence `tool_use → tool_use → tool_use → end_turn`. The decision logic is implemented by `run()` in `claims_intake/loop.py`. When `response.stop_reason == "tool_use"`, the harness executes the requested tools, adds their results to the working messages, and continues; when it is `"end_turn"`, `run()` returns the `FinalState`. Any other value raises `UnexpectedStopReason`, so loop termination depends on a structured API signal instead of parsing assistant prose.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

`tests/test_antipatterns.py::test_no_string_membership_against_text_in_loop` prevents magic-string membership tests against assistant text from controlling the loop. This matters because normal model wording is not a reliable termination protocol; a word could appear in an explanation without meaning that processing should stop. In `runs/20261116_112611/traces/claim_05_auto_collision.jsonl`, termination instead followed the structured `stop_reason` values. A text-matching approach could have stopped the claim prematurely or continued after completion, while the structured implementation passed all **30 System 1 tests**.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

`route_to_adjuster` and `escalate_to_human` are both terminal tools, but their descriptions establish different decision boundaries. `route_to_adjuster` is for a classified claim with confidence of at least 0.6 and an assessed severity, while `escalate_to_human` covers low confidence, unresolved ambiguity, missing critical facts, or policy disputes. In run `20261116_112611`, `claim_05_auto_collision` followed the routing path and completed as `routed` in four turns. Errors use the structured fields `is_error`, `error_category`, `is_retryable`, and `message`, allowing the agent to distinguish a retryable transient failure from a permanent invalid input instead of treating every error as an undifferentiated string.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

In `runs/20261116_112611/summary.md`, `claim_05_auto_collision` routed successfully in **4 turns**, using **14,831 input tokens and 926 output tokens**, with an estimated claim cost of **$0.0195**. The complete eight-claim run had an estimated cost of **$0.0961**. The exercise README gives an approximate full-run cost of about **$0.05 on Haiku 4.5**, so my observed run was higher. Live model responses and resulting tool-use paths can vary, so the reflection uses the actual values from run `20261116_112611` rather than assuming the README example is deterministic.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

In run `20261116-113253`, `budget.json` recorded **38,708 baseline tokens** and **16,910 assembled tokens**, a **56.31% reduction**. The active conversation dominates the assembled context; in the build evidence, the active section alone accounted for **15,789 tokens**, compared with 204 for `case_facts`, 386 for `resolved_refund`, and 512 for `resolved_subscription`. Keeping the active section verbatim preserves the precise current interaction where wording, unresolved details, and conversational position can still affect the next answer. Older resolved material can be compressed much more aggressively because its outcome is already known.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

The strategy summarizes older resolved material while preserving current active context and precise facts needed for reliable answers. My System 2 build showed `case_facts` at **204 tokens**, `resolved_refund` at **386**, `resolved_subscription` at **512**, and `active` at **15,789** tokens. The completed run `20261116-113253` contained **16,910 assembled tokens** against a **38,708-token baseline**, showing that most savings came from compressing material that no longer needed full conversational fidelity. The rule is therefore to compress completed history while preserving information whose exact content or position can still determine the next response.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

`runs/20261116-113253/eval.jsonl` passed **6/6 evaluation questions** using the complete assembled context. In `eval_control.jsonl`, the case-facts-stripped control produced the required regression: **1 of 2 control questions regressed**, with **Q6** being the regression observed in my run. This demonstrates that the facts block is not redundant context. Removing it measurably reduces answerability even when the remaining compressed conversation is still available.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

In `System 3 - Evidence/.claude/rules/react.md`, the path-scoped frontmatter is `paths: - "src/components/**/*" - "src/pages/**/*"`. This means the React conventions load specifically for components and pages rather than applying indiscriminately to unrelated repository content. Path-scoped rules are useful for cross-cutting technical conventions because applicability follows matching files instead of simply following a directory hierarchy. The configuration was successfully validated in my System 3 execution, where all **35 tests passed** and `python -m ecommerce_team_config .` returned **OK**.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

`System 3 - Evidence/.claude/skills/deploy-check/SKILL.md` declares `context: fork` followed by `allowed-tools:` containing `Read`, `Grep`, `Glob`, and restricted commands including `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git rev-parse:*)`, `Bash(git ls-files:*)`, and `Bash(gh pr view:*)`. `context: fork` isolates deployment investigation from the main working session, while the allowlist gives the fork enough capability to inspect repository state without general mutation privileges. Without the fork, investigative context could pollute the main session; without the restricted allowlist, a check intended to be observational could have a larger operational blast radius. The **35-passed + OK** validator result confirms that this configuration satisfies the supplied specification.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

My validator run returned **OK** after all **35 tests passed**, confirming the supplied configuration hierarchy. The `.claude/` directory saved in `System 3 - Evidence` is project-level configuration: its rules, `/review` command, and `/deploy-check` skill travel with this repository and define shared team behavior. User-level Claude Code configuration belongs at user scope rather than inside this project hierarchy, so personal settings can apply across repositories without becoming repository requirements. Separating these levels prevents a developer's personal Claude configuration from silently changing the team's checked-in project behavior.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

In my corrected System 4 run using `--since 2026-04-01T00:00:00Z`, the indexed `defects_since` query returned **17 new defects out of 40 total defects in the warm tier**. The shift output reported `new=17` and `shift C: 17 new defects`, while a direct count of `data/warm.sqlite` reported `warm-tier total: 40`. This demonstrates that filtering is pushed down into the indexed SQL query rather than loading the complete defect history into Python or model context. The model therefore receives only the relevant **17-defect slice**, not all **40 warm-tier records**, keeping context bounded as historical data grows.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

The System 4 test output shows the recovery boundary clearly: states at **1, 29, and 30 minutes** can resume, while **31 minutes** selects `fresh`; a completed state also selects `fresh`. `test_threshold_constant_is_30_minutes` confirms the staleness threshold is **30 minutes**, and `test_resumed_prompt_has_both_sections` passed in the same **33-test** run. Once transient reasoning becomes too old, starting a fresh session with recovered information is safer than assuming stale runtime context still represents the current system state.

Forked investigations remain isolated because `fork.py` gives each fork a **copy of the base state** and an **independent scratchpad**, rather than allowing the investigation to modify the main state directly. The same **33-test** System 4 run passed `test_fork_for_hypothesis_copies_state_without_mutating_base`, `test_two_forks_produce_independent_scratchpads`, and `test_merge_findings_appends_without_rewriting_existing`. Findings are merged back explicitly only after the investigation, allowing competing hypotheses to be explored without silently mutating or contaminating the base orchestration state.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

After my recorded Shift C execution, `wc -c data/hot_state.json` reported exactly **643 bytes**. This is well below the project's approximately **5 KB** hot-state budget, and the hot-state round-trip and size-budget tests passed within the **33-test** System 4 suite. A once-per-shift service may run indefinitely, so unbounded state would eventually create increasing storage and context-loading costs. Detailed history can remain in warm and cold storage while the hot state stays small, predictable, and inexpensive to load every shift.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.

→ **Model:** System 2's `runs/20261116-113253/eval.jsonl` records **6/6 successful evaluations** against the assembled context. This layer is where the model consumes context prepared by the surrounding system and generates semantic answers.

→ **Harness:** System 1's `claims_intake/loop.py` and `runs/20261116_112611/traces/claim_05_auto_collision.jsonl` show the harness controlling tools and termination. The observed sequence `tool_use → tool_use → tool_use → end_turn` demonstrates the harness translating model decisions into bounded execution.

→ **Orchestration:** System 4's `data/hot_state.json` and `data/shift_scratchpad.jsonl` represent state that survives across shift invocations. The hot state remained only **643 bytes**, demonstrating that cross-session orchestration is handled through explicit external state rather than an indefinitely growing model conversation.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code and one guided by prompt. When is each right?

System 1 guarantees loop termination behavior deterministically in `claims_intake/loop.py`: `end_turn` returns, `tool_use` continues, and unexpected stop reasons raise `UnexpectedStopReason`; all **30 System 1 tests passed**. In contrast, choosing `route_to_adjuster` versus `escalate_to_human` is guided by tool descriptions and model reasoning over the claim facts, as demonstrated by the outcomes in run `20261116_112611`. Code enforcement is appropriate for invariants, resource limits, and safety boundaries that must always hold. Prompt guidance is appropriate for semantic decisions where the correct action depends on interpretation rather than a simple fixed branch.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?

System 2 manages context inside a conversation: run `20261116-113253` reduced **38,708 tokens to 16,910**, a **56.31% reduction**, while still passing **6/6 evaluations**. System 4 applies the same bounded-context principle across sessions by externalizing history into storage tiers and keeping `data/hot_state.json` at only **643 bytes** after my recorded shift. System 2 compresses and selectively preserves conversational content, whereas System 4 persists durable history externally and loads only a small operational state. Both systems prevent accumulated history from overwhelming model context, but they apply that principle at different time scales.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

System 4's `test_hotstate_atomic_write` passed as part of the **33-test** suite. A normal successful shift cannot demonstrate atomic-write protection because no interruption occurs during that happy-path execution. The test validates behavior needed when a write is interrupted, which protects the next shift from consuming partially written operational state. This matters before shipping because reliability over repeated executions depends on handling rare failure paths, not merely demonstrating one successful run.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

For System 1, the blast radius of incorrect behavior is an incorrectly routed or escalated insurance claim. Terminal operations are constrained to `route_to_adjuster` and `escalate_to_human`, while the dispatcher rejects another terminal operation after `session.terminal_called`; structured tool errors are returned rather than crashing the loop. `claims_intake/loop.py` also uses a `Budget` as a safety mechanism and raises on unexpected `stop_reason` values. In run `20261116_112611`, all eight fixtures terminated and the System 1 test suite reported **30 passed**, showing that these enforcement points were functioning in my environment.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)

The first issue I encountered was environmental rather than a defect in the reference implementations: normal clipboard paste did not work inside the Vocareum browser-based VS Code terminal. I worked around the terminal-input limitation and verified the environment directly with `python --version`, which reported **Python 3.13.18**. I then validated each supplied final solution through its complete test suite: **30 tests for System 1, 30 for System 2, 35 for System 3, and 33 for System 4**. This helped separate a workspace interaction issue from an actual problem with the systems under evaluation.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

I would make System 1's final routing policy somewhat more deterministic after the model has already produced structured classification, confidence, and severity. In my final run `20261116_112611`, several fixtures whose expected outcome was `routed` were instead escalated, even though all claims terminated successfully and all **30 tests passed**. I would preserve model reasoning for ambiguous classification, but I would consider a narrow harness-level enforcement point for the documented confidence/severity threshold once those structured fields exist. This would reduce some model autonomy at the terminal decision point in exchange for more repeatable operational behavior.