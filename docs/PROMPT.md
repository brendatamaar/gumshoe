# Gumshoe — Agent Prompt Playbook

Nov 1, 2026 · @brendatama

Copy-paste prompts for Claude Code CLI and Codex to build Gumshoe one session at a time. Each prompt reads the same files and ends by updating the same progress file, so you can stop in one agent and continue in the other.

## Setup, once

Agents cannot open claude.ai links, so do these four things before the first prompt:

1. Export the four documents as Markdown into the repository: `docs/prd.md`, `docs/technical-design.md`, `docs/implementation-plan.md` and `docs/developer-guide.md`. Export again when a document changes, and treat `docs/` as read-only for agents unless a prompt says otherwise.
2. Commit `AGENTS.md` (the text is in the Developer Guide) and make `CLAUDE.md` a symlink to it. Codex reads `AGENTS.md`, Claude Code reads `CLAUDE.md`.
3. Create `docs/PROGRESS.md` from the template at the end of this playbook. It is the only memory shared between sessions and between the two agents.
4. Work on one milestone per branch, one task per session where you can, and commit after every task.

|  | Claude Code CLI | Codex |
| --- | --- | --- |
| Rules file | `CLAUDE.md` (symlink to `AGENTS.md`) | `AGENTS.md` |
| Functional-core checks | The plugin's hook blocks impure edits to core files while it works. Use its review skill before merging. | No hook. The prompts ask for `make check` and a manual pass over the core rules. CI is the backstop. |
| Continue the last conversation | `claude --continue`, or `claude --resume` to pick one | `codex resume` (check your version) |
| Best for | Core, guard and anything under the functional-core rules | Shell work: adapters, API, CLI, tests, CI |

Prefer a fresh session with the resume prompt over continuing a long conversation. `docs/PROGRESS.md` and git are the source of truth, not the chat history.

In every prompt, replace text in `<angle brackets>`.

## Kickoff (first session, M0)

Use this once, in an empty repository that already contains `docs/` and `AGENTS.md`. It stops at the end of M0 and does not touch M1.

```text
You are implementing Gumshoe, an open-source Python runtime that runs LLM agents with guardrails and replay.

Read first, in this order: AGENTS.md, docs/prd.md (overview, design principles, module requirements), docs/technical-design.md (in full), docs/implementation-plan.md (the intro, M0 and M1), docs/developer-guide.md.

Task: milestone M0, Repository and tooling, from the implementation plan. Do not start M1.

Rules:
- Follow the Developer Guide for layout, configuration and conventions. Where it says to check option names or versions, check the current documentation of the tool.
- core and guard are pure packages (Technical Design: Functional core and shell). Set up .functional-effect.json, the nested ruff.toml and strict pyright exactly as the Developer Guide describes.
- Confirm that every planned dependency supports Python 3.13. If one does not, stop and tell me before falling back to 3.12.
- The final names (PyPI distribution, import name, GitHub, domain) are undecided. Use the placeholders from the Developer Guide. Do not publish anything and do not create remote resources.

Work in this order and commit after each step:
1. Repository, Apache 2.0 license, README stub, AGENTS.md.
2. uv workspace with the five package skeletons.
3. Ruff, pyright (strict), pytest, Hypothesis, Makefile, pre-commit.
4. compose.yaml and .env.example.
5. GitHub Actions workflow.
6. The core purity check. Prove it by adding a deliberate violation (import os, import pydantic, datetime.now) to core, confirm that Ruff fails, then remove it.

Done when the acceptance criteria of M0 pass: uv sync and make check pass on a clean clone, the CI file is valid, and the deliberate violation was rejected.

Finally, create docs/PROGRESS.md with these sections: Current milestone, Done, Decisions, Open questions, Next step, How to verify. Fill it in. Show me the output of make check.
```

After the session, open the repository diff yourself. M0 is mostly configuration, and a wrong rule here is expensive later.

## Start a milestone

Use this at the start of M1 to M7. The agent plans first and waits for your approval, which is the cheapest point to correct its direction.

```text
Start milestone <M#: name> of the Phase 1 plan.

1. Read AGENTS.md and docs/PROGRESS.md. Then read the <M#> section of docs/implementation-plan.md and these Technical Design sections: <sections from the table below>.
2. Confirm that the previous milestone met its acceptance criteria: run make check and the tests it names. If it did not, stop and tell me.
3. Create the branch <m#-short-name>.
4. Before writing code, give me a plan: the tasks in order, the files you will touch, the invariants you will test, and anything in the design that is unclear or conflicts with the existing code. Wait for my approval.
5. After I approve, work one task at a time: failing test first (a Hypothesis property for logic in core or guard), then the code, then make check, then a commit whose message names the task.
6. Do not change the event schema or the state machine unless the task says so. If the design needs to change, stop and propose the change (what, why, which section) instead of working around it.
7. At the end, check every acceptance criterion of <M#> one by one and update docs/PROGRESS.md.
```

| Milestone | Technical Design sections to read |
| --- | --- |
| M1 Core contract | Core concepts, Run state machine, Events, Commands and the pure step function, Policy interface, Functional core and shell |
| M2 Event store | Events, Event store, Orchestrator (claiming and crash recovery) |
| M3 Runtime v0 | Commands, Tool router and self-healing, Security and privacy |
| M4 Orchestrator, API, CLI | Orchestrator, Event store, API, CLI and mascot |
| M5 Cost kill-switch | Policy interface, Cost kill-switch |
| M6 Tracing | Tracing, Events |
| M7 Demo and release prep | API, CLI and mascot, Testing strategy, plus the release checklist in the plan |

M1 is the one to watch closely. Everything else depends on its event schema and state machine.

## Work on a task

The prompt for the loop you will run most often. Keep one task per session: a small scope keeps the diff reviewable and the context short.

```text
Implement this task from milestone <M#>: <task text copied from the plan>.
Design reference: Technical Design, <section>.
Scope: change only <package or paths>. Do not edit docs/.

Steps:
1. Read docs/PROGRESS.md and the design section above.
2. Write the failing tests first. For logic in core or guard, write a property per invariant and a case per transition row.
3. Implement the smallest change that makes them pass.
4. Run make check. Fix everything it reports. Do not weaken typing or the lint rules to get it green.
5. If you touched core or guard, check the rules in AGENTS.md one by one: no I/O, no clock, no async, no Pydantic, frozen dataclasses, no-op for an unhandled pair, results matched by call_id and attempt. (Claude Code: run the functional-effect review skill instead.)
6. Commit with a message that names the task.

If a test shows that the design is wrong or ambiguous, stop and ask me. Do not guess.
Report: what changed, which tests you added, and anything you deferred.
```

## Resume a session, or switch agents

Use this after a break, after the conversation got long, or when you move from one agent to the other. It makes the agent prove where things stand before it changes anything.

```text
Resume work on Gumshoe.

1. Read AGENTS.md and docs/PROGRESS.md.
2. Run git status, git log --oneline -15 and git diff --stat. Tell me about any mismatch with PROGRESS.md: uncommitted work, commits that are not recorded, a different branch.
3. Run make check and report the result before you change anything.
4. Summarize in five lines: current milestone, last finished task, what is half done, the next task, and any blocker.
5. Then wait for my go. If I said "continue", start the next task: failing tests first, then the code, then make check, then a commit.
```

Before you leave an agent, ask it to write the handoff:

```text
Prepare a handoff. Commit finished work. Put unfinished work on the current branch as a commit that starts with "wip:" and describe it. Update docs/PROGRESS.md: what is done, what is half done and where, decisions made and why, open questions, the next task, and the exact commands to verify. Do not start new work.
```

## Review before merge

Run this in a fresh session so the reviewer has not seen the code being written. It only reports. It does not edit.

```text
Review the changes on this branch against main for milestone <M#>. Do not edit any file.

Check, and cite file and line for each finding:
1. Acceptance criteria of <M#> in docs/implementation-plan.md: met, partly met, or not met.
2. Purity of core and guard: no I/O, clock, randomness, async or Pydantic. Time and ids arrive as input fields. Delays are ScheduleInput commands.
3. State machine: unions are closed and end in assert_never. A (state, input) pair with no transition returns the same state and no commands. Terminal states emit nothing.
4. Results are matched by call_id, step_id and attempt, so a stale or duplicate result is ignored.
5. Boundary: model replies, tool arguments, HTTP bodies and database rows are parsed in runtime before reaching core.
6. Persistence and delivery: one transaction for inbox, events, projection and outbox; deterministic ids; lease epoch checked on commit.
7. Event schema: no shipped payload was edited; any change has a new schema_version, an upcaster and a deliberate golden snapshot update.
8. Tests: a property for each invariant, a contract test for each fake, and a test that fails if the code under test is broken.
9. Anything outside the milestone scope.

List findings as blocker, should fix, or nit. End with a merge or do-not-merge recommendation.
```

In Claude Code, add: "For core and guard files, use the functional-effect review skill and include its findings." Codex has no such skill, so the checklist above does that job.

## Other cases

**A test or CI check fails.** The agent must find the cause, not make the failure disappear.

```text
<paste the failing output>
Find the root cause. Do not change or delete the failing test, and do not loosen a lint or type rule, unless you can show that the test itself is wrong, and then tell me first. Write a regression test that fails for the right reason, fix the cause, run make check, and commit. Explain the cause in two sentences.
```

**You want to change the design.** No code changes until the document changes.

```text
I want to change this: <describe the change>. Do not edit code or docs yet.
Read the affected sections of docs/technical-design.md and docs/implementation-plan.md. Then give me: the exact text that would change in each document, the effect on milestones already done, the tests that would change, and the risks. Wait for my approval. After I approve, update the docs/ copies and list the edits I must repeat in the claude.ai documents.
```

**The agent is stuck or the scope is growing.**

```text
Stop coding. Summarize in a few lines what you tried, what failed, and what you believe the cause is. Then give me two or three options with their cost, and recommend one. Do not continue until I choose.
```

**Check for drift before a milestone ends.**

```text
Compare the code with docs/technical-design.md for the parts built so far. List every place where the behavior differs from the document: names, event fields, state transitions, error classes, defaults. Do not fix anything. Mark each difference as code is wrong, document is wrong, or unclear.
```

**Release prep (M7).** The agent prepares; you publish.

```text
Prepare the 0.1.0 release without publishing anything. Work through the release checklist in docs/implementation-plan.md. Do each item you can do in the repository, and mark the items that need me: final names, trademark check, GitHub and PyPI settings, security contact, and the employment-contract check. Dry-run the build and the TestPyPI upload only if I give you a token in this session. Report the checklist with a status per item.
```

## docs/PROGRESS.md template

Create this file before the kickoff, or let the kickoff prompt create it. Every prompt above reads it, and the agent updates it at the end of every session.

```markdown
# Gumshoe progress

Last updated: <date> by <Claude Code | Codex>

## Current milestone
<M# name>, branch <branch>, status <not started | in progress | done>

## Done
- <task> (commit <hash>)

## Half done
- <task>: <where it stopped, what is left>

## Decisions
- <date> <decision> because <reason>. Document affected: <none | section>

## Open questions
- <question>, recommended default: <default>

## Next step
<the single next task, with the plan reference>

## How to verify
- make check
- <any milestone-specific command>
```
