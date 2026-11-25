---
name: plan-council
description: Trigger this if the user says 'plan council'. Drafts a lean implementation plan and runs a council design review without over-planning.
---

# Plan Council

Draft a concise implementation plan, then run it past the council for design
review.

Use the `council` skill to run the delegates and format the score table.
Where `plan-council` and `council` disagree—such as review prompts, member
guidance, and stopping rules—this skill takes precedence.

A plan captures **decisions, boundaries, and sequence**—not the
implementation written out in prose. Either the user or an implementing agent
should be able to read it in under ten minutes and know what gets built, why,
in what order, and how it will be verified. The implementer will read the
source code anyway; do not duplicate it here.

## 1. Draft the plan

Inspect enough of the codebase to verify real file paths, symbol names, and
existing primitives you can reuse, then write the plan with these seven
sections:

1. **Goal & non-goals** — A few sentences on the objective, plus a short list
   of what is explicitly out of scope.
2. **Key decisions** — Up to ~8 choices, 1–2 sentences each: what you picked
   and why it beats the obvious alternative.
3. **Changes by area** — Per package or module, which files change and the
   new behavior they introduce.
4. **PR sequence** — Small, independently shippable PRs and their
   dependencies.
5. **Verification** — For each PR, the tests or live checks that prove it
   works.
6. **Known limits & deferred work** — Edge cases and follow-ups you are
   intentionally leaving out for now.
7. **Open questions** — Only decisions that genuinely require the user.

### Size and restraint

- **Word budget:** Target under **2,000 words**; hard ceiling **3,000 words**.
  If the draft exceeds the ceiling, cut before sending it to review.
- **No line numbers:** Reference files and symbols, never line numbers.
- **No prose code:** Skip pseudo-code, SQL, full schemas, and lock hierarchies
  unless the core design cannot be understood without them.
- **Invariants over concurrency machinery:** Do not design fencing tokens,
  generation counters, or multi-layer race guards in a plan. State the
  invariant in one line (e.g., *"A cancelled run must release its lock"*) and
  leave the mechanism to the implementer and code review.
- **Cite, don't re-explain:** Name existing mechanisms and say you will reuse
  them (e.g., *"Reuse the existing run lease and `fencingEpoch` checks"*).
- **Honor the user's mode:** If the user asked for a POC or prototype, carry
  that through every section—no backwards-compatibility layers, no migration
  shims, and a lower severity bar.

## 2. Brief the council

Give each delegate the plan path, any prerequisite plans, the user's mode and
constraints, and this shared prompt:

> Review this plan as a **design**, not as code. Answer:
> 1. Does the design hold up? Where would a real user hit a failure?
> 2. Are any assumptions about the codebase or prerequisites wrong?
> 3. What can be cut or simplified without hurting the goal?
> 4. Is anything essential missing that blocks the goal?
>
> Do not ask for more implementation detail. Lock ordering, fencing, timing
> windows, and line numbers belong in code review.
>
> **Severity**
> - **Blocker:** The design won't work, contradicts the codebase or a
>   prerequisite, or plausibly causes data loss or cross-user/cross-tenant
>   exposure.
> - **Should:** A real user will plausibly hit it, or it is a clear
>   simplification.
> - **Nit:** Everything else, including narrow races and rare timing windows.
>
> Every **Blocker** or **Should** requires a concrete scenario: who does what,
> and what breaks. Without one, report it as a **Nit**. Report every issue you
> find, labeled by severity.
>
> **Score (1–10):** `9–10` = ready to implement; `8` = ready after minor
> edits; `≤7` = at least one Blocker. Nits never lower the score.

Append the member-specific line after the shared prompt (replacing the
code-review lines in `council`):

- **GPT Sol:** Don't be overly critical. If a failure requires a narrow timing
  window or an unusual sequence of actions, label it a nit.
- **Claude Opus:** Focus on where this design fails a real user or is more
  complex than necessary. Keep praise and confirmations to one line.

On **Round 2+**, also pass the **Declined** list and instruct delegates to
focus on what changed—and not to re-raise declined items unless they bring a
new concrete failure scenario.

## 3. Triage and fight bloat

Assign every finding one outcome:

- **Accept:** Update the plan. If the edit takes more than a sentence or two,
  you are sliding into implementation detail—compress it to a one-line
  invariant instead.
- **Decline:** Record it on the **Declined** list with a one-line reason.
- **Defer:** Move it to *Known limits & deferred work*, or leave it for code
  review.

Treat simplification suggestions as first-class edits—apply every cut you
agree with.

Track the plan's word count each round. If a round expands the plan by more
than **~15%** (or pushes it over **3,000 words**), prune before starting the
next round.

## 4. Stopping rules

- **Default cap:** At most **2 rounds** (Round 1: full design review;
  Round 2: targeted check of revisions). Ask the user before running a third
  round.
- **Early stop:** End after Round 1 if you accepted **no Blockers** (you can
  still fold in minor *Should* edits and cuts without a second pass).
- **No unanimity trap:** Delegates do not all have to agree. Record any
  holdout as a disagreement and stop.
- **Self-review bias:** If you wrote the plan and one delegate runs on your
  own model or provider, call that out in the summary and weight its score
  lower.

## 5. Final handoff

Include in your final reply:

- Plan path and final word count
- Rounds score table (using the `council` format)
- Accepted changes and cuts (brief)
- Declined items with reasons, plus any holdout disagreements
- Open questions for the user

Stop after presenting the plan—do not start implementing unless the user asks.
