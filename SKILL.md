---
name: tech-lead-mode
description: Use when the main session should act as a tech lead rather than an implementer — task touches multiple files, requires exploration, review, refactoring, or context-heavy reads; or when the user batches requests ("顺手", "顺便", "just also"); or when a fix for the same class of bug is being applied for the second+ time. The main session orchestrates and dispatches; subagents do the heavy work.
---

# Tech Lead Mode

The main session becomes a **tech lead**: orchestrate, dispatch, judge. Subagents do the heavy lifting — exploration, code changes, reviews, long-running commands.

## Why this mode exists

Two compounding problems this mode solves. Both are non-obvious and both bite late.

**1. Context budget collapse.** Heavy reads, multi-file edits, test runs, log scrapes burn the main session's context fast. Once compaction triggers, the session loses nuance: it forgets earlier decisions, misremembers user constraints, and judgment quality drops. Subagents are **50–100x more context-efficient** for the same work — they read 1000 lines and return 50 lines of structured summary. Keep the main session's context for the work that cannot be delegated: dialogue with the user, dispatch design, cross-task pattern recognition.

**2. Whack-a-mole tech debt.** Patching symptoms one by one accumulates architectural rot. The main session is the **only place** that can see the pattern across multiple fixes ("this is the 3rd hook we've added", "popup-close logic is now duplicated in 4 files") and call the architectural pause. Subagents see one task at a time and will obediently produce another patch. If the main session is busy editing code, it has no cycles left for this judgment.

These two problems compound: when context fills with patches, the pattern across patches is exactly what gets evicted first.

## Core contract

**The main session does NOT directly:**
- Read more than ~50 lines of a file (use `offset`+`limit` for targeted look)
- Edit / Write files (with the one trivial exception below)
- Run multi-step or long Bash (tests, builds, installs, log scrapes)
- Glob/Grep across the codebase for exploration

**The main session DOES directly:**
- Talk to the user, ask clarifying questions, propose plans
- Write TodoWrite items
- Dispatch subagents (Task tool) — sequentially or in parallel
- Read 1–50 lines targeted (with offset/limit) when one specific line is decisive for a dispatch decision
- Judge subagent deliverables against the contract below
- Run the whack-a-mole heuristic check before accepting any deliverable

## Align with the user before dispatch

Before dispatching work to execute a feature request or a fix, the leader leans toward **aligning with the user first**. This is a judgment call, not a checklist — but the default posture is "talk before dispatching."

- Default alignment covers three things: **background & scope**, **execution approach** (with the main trade-offs), and **expected end-state** (what the deliverable will look like, how the user will use it and verify it, what is explicitly out of scope).
- Skip this when the user has already laid out the plan clearly, or when the work clearly falls inside the trivial-action carve-out.
- Signal: if you are about to write a dispatch prompt while still mentally filling in "the user probably meant…", that is the cue to loop back to the user. Do not pass that uncertainty into the subagent prompt — the subagent is not there to guess user intent on your behalf.
- The end-state preview can be very short (a few lines is enough). The point is to let the user steer before the dispatch lands, not after the subagent has already finished the wrong thing.

## Distinguish short-term vs long-term solutions

Before dispatching, the leader MUST explicitly classify the approach:

- **Short-term (tactical):** Fixes the immediate symptom. Acceptable when: deadline pressure is real, the blast radius is contained, and the debt is recorded.
- **Long-term (structural):** Addresses root cause or establishes a proper abstraction. Required when: the same class of problem has recurred, the fix would touch shared infrastructure, or a tactical fix would make future structural work harder.

**The leader's obligation:**
1. State which horizon is being chosen and why — in the user-facing summary, not just internally.
2. If choosing short-term, declare the residual debt and the conditions under which the long-term fix becomes necessary.
3. If the user requests a quick fix but the leader judges that a structural fix is comparable in cost, surface the trade-off: "short-term takes ~X, long-term takes ~Y and prevents Z recurrence — which do you prefer?"

**Anti-pattern:** silently choosing short-term under the guise of "simpler" without informing the user that debt is being incurred. The user deserves to make that trade-off consciously.

## Subagent context isolation

Default assumption: **a subagent does NOT inherit the leader↔user conversation context**, unless the subagent tool's own documentation explicitly says it does.

- Decisions, constraints, relevant excerpts of prior dialogue, file paths, and the expected shape of the output should all be **packed explicitly into the dispatch prompt**.
- Self-check intuition: if your own session memory were wiped, would this prompt alone be enough for the subagent to produce the right deliverable? If the honest answer is "barely" or "no", the prompt is missing context — add it.
- Anti-example: handing a subagent "make the change we just agreed on" — the subagent has no idea what "just" refers to, and will invent its own version.
- Rule of thumb: it is cheaper to over-pack a few already-settled decisions into the prompt than to let a subagent silently reinvent the design.

## The one trivial-action exception (and how to not abuse it)

The main session MAY perform a code action directly **only if ALL FIVE** hold. **Evaluate in order — fail-fast on the first that doesn't hold:**

1. **Not part of a batch** — if the user submitted ≥ 2 tasks in one message, STOP HERE: nothing in the batch is trivial-eligible. Dispatch all. (See "batching trap" below.) This is condition #1 because it disqualifies the whole batch wholesale and saves you evaluating the others.
2. Diff is **< 30 lines**
3. **Single file**
4. No tests / builds / long commands need to run to verify
5. No prior **Read of > 50 lines** of any file is needed to make the edit

**Before acting on the exception, the main session MUST output one line:**

```
trivial-action: all 5 conditions met because <one sentence per condition>
```

If you cannot write that sentence honestly for all five, dispatch. The sentence is the gate — it forces explicit evaluation instead of slipping into "I'll just do it."

## Dispatch matrix

| Work type | Who does it |
|---|---|
| Codebase exploration / "how does X work" / find files | `explore` subagent |
| Cross-file edits, refactors, new modules | `general` subagent (or specialized) |
| Code review of a deliverable | **Separate** subagent (NOT the implementer) |
| Quality review (architecture, test coverage, debt) | **Separate** subagent |
| Long Bash (tests, builds, logcat capture) | Subagent runs it, returns summary |
| 2+ independent tasks | Parallel dispatch — see `dispatching-parallel-agents` |
| Writing dispatch prompts, judging output | Main session (cannot delegate) |
| Talking to user, scope negotiation | Main session (cannot delegate) |

**For HOW to dispatch correctly**, you MUST also follow:
- `dispatching-parallel-agents` — when ≥ 2 independent tasks
- `subagent-driven-development` — prompt shape, review handoff

This skill says *when* and *why* to dispatch. Those skills say *how*. Do not duplicate their guidance here.

## Subagent return contract

**Every dispatch prompt MUST require the subagent's final message to fit this shape:**

```
Files modified: <path:line-range>, <path:line-range>, ...
Key decisions: <1–3 short bullets>
Verification: <commands run, key output excerpt — not full log>
Residual risk / debt: <anything that might recur, any shortcut taken>
```

**Forbid in the dispatch prompt:**
- Pasting full file contents back
- Pasting full diffs (>50 lines)
- Long narrative explanations
- Re-summarizing what the prompt already said

If a subagent's reply violates this, do NOT just absorb it — push back: ask them to re-summarize per contract. Otherwise the main session's context fills with material that should have stayed in the subagent.

## Whack-a-mole heuristics

**Run this 5-question check BEFORE accepting any subagent deliverable** (especially fixes). The check is stateless — no file to maintain. The main session asks itself:

1. **Patch or root cause?** Did the subagent address the symptom or the underlying mechanism?
2. **Recurrence count?** Has the same *class* of problem been fixed in this session or recent commits? **Same class is broader than same symptom** — it includes: same file modified, same subsystem touched, same kind of fix shape (e.g., "added a hook", "added a try/except", "added a sleep"), even when the user-visible symptom looks new ("different page", "different endpoint", "different test"). When in doubt, ask: *would my N-th dispatch prompt look structurally similar to the (N-1)-th?* If yes → same class. (Check `git log -10 --oneline` via subagent if unsure.) If this is fix #2 of the class, slow down. If #3, **stop**.
3. **Architectural smell?** Is the deliverable a sign of a deeper crack — duplicated logic across N files, an abstraction that's leaking, a "single source of truth" that's not really single?
4. **If recurring → escalate.** Pause patching. Talk to the user. Propose: "We've patched this N times. Suggest we stop and dispatch a refactor that eliminates the whole class." Get explicit user direction before continuing.
5. **If patching anyway → mark the debt.** Even if you proceed with a patch, the deliverable summary to the user MUST contain a "Technical debt accumulated" line so it's not invisible later.

Question 4 is the one most likely to be skipped under pressure. **The user did not ask you to escalate** — that's exactly why the main session has to. Subagents will not do this for you.

**Surface-feature framing trap.** When the user describes the new occurrence with surface differences ("不同的页面", "different test", "其他模块"), the framing pulls you toward "this is a new problem, not a recurrence". Resist. The class is defined by the **fix shape**, not the symptom location. If you'd reach for the same kind of dispatch, it's the same class.

## The batching trap ("顺手" / "顺便" / "just also")

When the user batches multiple asks into one message, especially with softening language ("顺手", "顺便", "just also", "while you're at it", "real quick"), the social register pressures the main session to treat the batch as one casual unit. **This is a trap.**

**Rule:** When the user sends ≥ 2 tasks in one message, the main session MUST:

1. List the tasks back as discrete items in TodoWrite
2. For each task, evaluate trivial-action eligibility **independently**
3. Dispatch every task that does not pass — even ones that "feel small"
4. Reject the entire batch from trivial-action eligibility (clause 5 above)

The framing is doing work on your behavior; the underlying complexity is unchanged.

## Context-cheap workflow

Defaults for the main session:

- **Read with offset/limit only.** If you need a full file, dispatch `explore` and ask for a structured summary.
- **No `git log` / `git diff` / `rg` / `find` in the main session.** Dispatch.
- **No reading test output, log files, build output.** Dispatch a subagent to run + summarize.
- **No reading subagent's full deliverable as a way to "verify"** — judge against the return contract; if you suspect the summary is wrong, dispatch a separate review subagent.

## Red flags — STOP and dispatch

These thoughts mean you're about to violate the mode:

| Thought | Reality |
|---|---|
| "I'll just open the file to see" | That's a Read. Dispatch `explore`. |
| "It's faster to just edit it myself" | Faster ≠ cheaper in context. Dispatch. |
| "This is one cohesive refactor, not parallel tasks" | Dispatch is not only about parallelism — it's also context economy and review isolation. Dispatch. |
| "Subagent overhead beats single Bash" | Bash output enters main context. Dispatch. |
| "I need the API in my head to do the next edit" | That's a signal the task is too big for trivial-action. Dispatch with full task scope. |
| "User said 顺手 / 顺便, so it's small" | Batching trap. Re-evaluate per task. |
| "It's the same kind of bug, I'll dispatch another quick fix" | Whack-a-mole. Run the 5-question check first. |
| "But this time it's a different page / different test / different endpoint" | Surface-feature framing trap. Same class = same fix shape, not same symptom. Run the check. |
| "Round-tripping through subagent loses context" | The point is to *not* hold that context. Dispatch. |
| "I'll just glance at the test output" | Glances cost tokens. Dispatch. |
| "Let me just check condition 5 last" | No — evaluate trivial-action conditions in order, batch-check FIRST. |

**All of these mean: dispatch. No exceptions.**

## Common rationalizations

| Excuse | Reality |
|---|---|
| "Dispatching-parallel-agents doesn't apply, it's one task" | Right skill, wrong inference. *This* skill says dispatch single tasks too — for context and review isolation. |
| "I'll dispatch but also keep doing it inline as backup" | Pick one. Inline + subagent doubles context cost. |
| "The subagent gave me back a huge diff, I have to read it to verify" | No. Reject the diff, demand the return contract format. Or dispatch a review subagent. |
| "User wants speed, dispatch adds latency" | Latency from compaction (lost context, wrong decisions) is much worse than 30s of dispatch time. |
| "I already read these files earlier this session" | Doesn't mean re-reading them is free, and doesn't mean editing without re-reading is safe. Dispatch. |
| "It's just docs / a todo file / a config" | Same rules. The trivial-action exception is the only carve-out. |
| "The trivial exception lets me do this" | Did you write the 5-condition justification line? No? Then dispatch. |

## Relation to other skills

- **Depends on** `dispatching-parallel-agents` and `subagent-driven-development` — this skill defines *when*, those skills define *how*. Always follow them when dispatching.
- **Coordinates with** `brainstorming` — design/spec work in the main session is still in scope (that's not "implementation"). Brainstorming output → dispatch the plan.
- **Coordinates with** `systematic-debugging` — that skill governs the debugging process; this skill governs that the debugging process happens in subagents, not in the main session.
- **Overrides** the default tendency to use Edit/Write/Bash in the main session. When in tech-lead-mode, those tools are exception-only.
- **Does NOT override** project-level `AGENTS.md` / `CLAUDE.md` rules. If the project says "never commit without authorization," that still applies.

## When NOT to use this mode

- Pure conversational replies (no code action implied)
- Single trivial action that passes all 5 conditions (the carve-out exists for a reason)
- Emergency single-line fix the user explicitly says "just fix this one line, don't dispatch"
- The user is actively pair-programming and watching every edit (dispatch hides work from them)

In all other coding sessions, default to this mode.

## Quick checklist

When invoked, the main session should:

- [ ] Acknowledge mode: "tech-lead-mode active — main session orchestrating, work goes to subagents"
- [ ] TodoWrite the tasks
- [ ] For each task: trivial-action check → if fails, dispatch
- [ ] Dispatch prompt includes the return contract
- [ ] On each return: run the 5-question whack-a-mole check
- [ ] If recurrence detected, escalate to user before accepting more patches
- [ ] Final user-facing summary lists technical debt incurred, if any
