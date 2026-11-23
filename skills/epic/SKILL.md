---
name: epic
description: Trigger this if the user says 'epic'.
---

# Epic

If the user triggered this, they want a large goal built end to end by a team
of agents, with you as the lead. They are not watching this thread. Your
updates and your final message go unread until long after you stop. You own
the goal from its current state to every PR merged and a published report.

If the user did not explicitly request "epic" in some clear way, do not assume
full autonomy. If they did, everything below applies. If the goal fits in one
PR, use the `full-send` skill instead.

You are the manager, not the implementer. You plan, brief, track, decide and
verify. Agents write the code, review it and merge it. Fix something yourself
only when it is smaller than the brief you would write for it.

The user grants you permission to:

- Start implementer, reviewer, merger and analysis agents, in parallel, each
  in its own worktree.
- Let a merger merge a PR once the merge gate passes.
- Let agents force-push a PR's own branch with `--force-with-lease` after a
  rebase.
- Decide product questions the source of truth leaves open. Record each one
  as a lead decision.
- Use the test methods that Proof needs, without asking first.

## Testing permission

The `full-send` skill's testing permission applies to you and to every agent
on the team, for the tests this goal needs. Put it in the handbook so agents
know they have it.

## The team folder

Keep the team's shared docs in one folder outside the repo and outside
`/tmp`, for example `~/.claude/projects/<project>/team/`. A reboot wipes
`/tmp`. Agents read these docs, so briefs stay short.

- `HANDBOOK.md`: the goal, the source of truth, repo and git rules, shared
  machine rules, the quality bar, the review protocol, numbered lead
  decisions, and field notes.
- `PLAN.md`: slices in waves. Each has an ID, what it depends on, and its hot
  files.
- `LEDGER.md`: your tracking. Each slice's PR, status, agent task IDs and
  merge commit, plus incidents and what comes next.
- `REVIEWER.md` and `MERGER.md`: the playbooks for those roles.
- `gap-<area>.md`: the gap reports.
- `report/`: the final report.

Add to the handbook as you learn: field notes from incidents, amendments to
decisions, and review follow-ups named for the slice that owns them. Tell
running agents about changes that affect them.

## Steps

1. **Set the goal.** Name the source of truth: a design, spec, prototype or
   list. Read enough of it and the repo to split the work. Write the handbook.
2. **Find the gaps.** Start read-only analysis agents in parallel, one per
   area. Each compares the source of truth with the code and writes
   `gap-<area>.md`: what's missing, where, how big, what it depends on, and
   open questions.
3. **Decide and plan.** Answer the open questions as lead decisions. Give each
   shared piece (a schema, a column set, a mechanism such as approvals) one
   owning slice, so two PRs never build it twice. Write `PLAN.md` in waves: a
   slice starts only when what it depends on has merged. Assign shared slots
   up front, such as migration timestamps and ports.
4. **Run the waves.** Start an implementer for each ready slice. Each PR goes
   implementer → reviewer → merger, and each role is a different agent. Keep
   several slices in flight, but never two that rewrite the same files.
5. **Follow each PR to merged.** When an agent finishes, update the ledger and
   start the next one: a reviewer after the implementer, a fix round with the
   review's findings after CHANGES REQUESTED, a merger after APPROVE.
6. **Accept.** When the plan has merged, start an acceptance agent on main
   against the source of truth. It marks every item ✅, 🟡 or ❌ with
   evidence such as screenshots or command output. Each 🟡 and ❌ becomes a
   slice in the next round. Repeat until coverage is complete or the user says
   to stop.
7. **Report.** Publish the report as an Artifact: coverage, every PR with its
   merge commit, proof, lead decisions, incidents, and what's left. In your
   final message give the link, the counts, the main tip, and each decision
   you made for the user.

Stay active from step 1 to step 7. Agents run in the background, and you are
woken when each one finishes. Never poll them. If you end your turn with
nothing running and nothing watching, the work stops, because nobody will
prompt you again.

If the user interjects ("wrap up within the hour", "drop X"), update the plan
and the ledger, and follow what they said.

## Briefs

An agent knows only its brief and the team folder. Every brief has:

- Its role and slice ID, and "Read HANDBOOK.md first", plus its playbook.
- The slice's goal, what is in and out of scope, and who owns the shared
  pieces it touches.
- Its branch, its worktree, and its own database, ports and slots.
- The proof it must show, and its final line:
  `PR <url>, head <sha>, checks <state>`.
- For a fix round, the review's findings.

In T3 Code, start agents with `delegate_task` (`isolation: "worktree"`,
`mode: "async"`, `role: "implementation"` or `"review"`). Children never call
`watch_pull_request` and never start their own agents.

A reviewer gets the PR, not your opinion of it. For risky diffs (security,
permissions, data loss, concurrency), add a review from another model with
the `sol-loop` skill. A merger has neither written nor reviewed the PR.

## Shared machine rules

Agents share one machine, one database server, one set of ports and one
GitHub account. Put these rules in the handbook:

- Never touch the user's own checkout. Each agent works in its own worktree.
- Never reset, nuke or recreate shared services. Each agent makes its own
  database and drops it when done.
- Pick free ports and check them first. Kill only PIDs you started and
  checked are yours. Never kill by pattern, user, group or parent.
- If a shared service goes down, agents stop and tell the lead. Only the lead
  restarts it.
- One GitHub account means reviewers can't approve. They post
  `gh pr review --comment` with a written verdict. Every comment follows the
  `leaving-pr-comments` skill.

## Proof

Proof works as it does in `full-send`: evidence the agent observed itself.
Every PR body says how it was proven. Green mocked tests do not prove a user
experience. For UI and runtime behavior, an agent runs the app and looks.

An agent's "completed" is a claim, not proof. Read its final line and check
the PR yourself before you start the next role.

## Merge gate

A merger merges only when all of these are true on the latest commit:

- The latest review verdict is APPROVE, or every blocking item was fixed in
  later commits and the merger checked each fix in the diff.
- Changes pushed after an approval were checked against an explicit
  invariant from that review, not just re-read. For example: "may only keep
  more segments than the approved head".
- Required checks are green on this head, not on a run from before a rebase.
  If CI can't run (an outage, billing), the merger runs the full CI suite
  locally on the exact head, posts the commands, counts and SHA as a PR
  comment, and says so in the merge body.
- The branch doesn't conflict with the default branch, and no commits that
  landed there after the last green run touch this PR's files, confirmed just
  before the merge with `git fetch` and
  `git diff --name-only <base at last green run> origin/<default>` against the
  PR's files. Being behind alone doesn't block the merge.
- Shared slots, such as migration timestamps, are still later than
  everything on the default branch.

Merge with the method the repo uses. Never use `--admin` or other bypasses.

## Lessons

- Rules that keep users safe, such as "this needs approval" or "never save
  that data", fail closed where the action happens: where the element is
  resolved, on every save path. Static pattern checks miss cases. Expect
  these slices to take several review rounds, and plan for it.
- One owner per mechanism. When two slices need the same thing, one builds
  it and the other waits.
- Write each incident into the handbook as a field note, with the rule that
  prevents it, before you start the next agent.

## Asking the user

A question in a normal message goes unseen. Use the harness's question tool,
as in `full-send`, with all the context in the question: what blocks you, and
the options with your recommendation first.

Ask only when you cannot continue alone:

- The goal is ambiguous in a way that changes the plan.
- You can't read the source of truth.
- The next step is outside the grant above: deleting the user's data,
  touching production, changing shared infrastructure the user runs, or
  closing a PR.
- A slice stays disputed after its review budget, with evidence on both
  sides.

Decide everything else yourself, and record it as a lead decision.

If no question tool is available, or the tool returns no answer, take the safe
path: leave that slice's PR open and unmerged, carry on with the slices that
don't depend on it, and start your final message with the question.
