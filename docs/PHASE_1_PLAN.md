# Gumshoe — Phase 1 Implementation Plan

Oct 31, 2026 · @brendatama · Status: Draft v0.2 · Companion to the PRD and the Technical Design

## Goal and exit criteria

**Goal.** Ship a runnable, documented Phase 1: the `core` contract, the orchestrator, the tool router with self-healing retries, the cost kill-switch, OpenTelemetry tracing, and an offline demo.

**Phase 1 is complete when all of these are true:**

- [ ] `gumshoe demo` runs offline and shows 100 concurrent runs completing
- [ ] One runaway run is downgraded at the soft limit and halted at the hard cap, and its trace shows both decisions
- [ ] A tool call with invalid arguments is retried up to 3 times with the error fed back, and a run that keeps failing ends `FAILED`
- [ ] Killing a worker mid-run loses nothing and duplicates no tool side effect: another worker finishes the run
- [ ] All state-machine invariants pass as property tests
- [ ] The README has a demo recording, the architecture diagram, and a quickstart that works from a clean machine

Design details live in the Technical Design. This plan only says what to build, in what order, and how to know each piece is done.

**Method.** The core follows Model Functional Effect with the `distributed` profile (Technical Design, Functional core and shell). From M0 on, every milestone ends with the plugin's check passing for `core`. M2 and M4 carry most of the added work: the inbox, the outbox, and timers as commands.

## Milestones at a glance

| Milestone | Output | Depends on | Size |
| --- | --- | --- | --- |
| M0 Repository and tooling | Workspace, CI, local services | none | S |
| M1 Core contract | Models, state machine, `step`, policy interface | M0 | M |
| M2 Event store | SQLite and Postgres behind one port, with inbox and outbox | M1 | L |
| M3 Runtime v0 | Agent loop, tool router, self-healing | M1, M2 | L |
| M4 Orchestrator, API, CLI | Concurrency, leases, crash recovery, endpoints | M2, M3 | L |
| M5 Cost kill-switch | Policy, price table, downgrade and halt | M1, M3 | M |
| M6 Tracing | Event-to-span mapping, OTLP export | M1, M2 | S |
| M7 Demo and release prep | Offline demo, README, 0.1.0 | all | M |

Sizes are relative (S, M, L), not time estimates. No dates are committed, in line with the PRD roadmap.

**Order.** M0, M1, M2, M3, M4, then M5, M6 and M7. After M3 exists, M5 can run in parallel with M4, and M6 only needs M1 and M2, so it can start as soon as the event store works.

**Rule.** Do not start a milestone before the previous one meets its acceptance criteria. The event schema and the state machine (M1) are the most expensive things to change later.

## M0 · Repository and tooling

**Goal.** An empty but fully wired monorepo where lint, types, tests and CI run on day one.

**Tasks**

- [ ] Decide the final names: PyPI distribution name, import name, GitHub repository or organization, and domain. Confirm availability and check trademarks
- [ ] Create the repository with the Apache 2.0 license, a README stub, and the agent rules file (see the Developer Guide)
- [ ] Set up the uv workspace with five package skeletons: `core`, `runtime`, `guard`, `lab`, `extras`
- [ ] Configure Ruff, pyright (strict), pytest and Hypothesis at the root. Add .functional-effect.json (profile distributed, core paths packages/core/src/gumshoe/core/\*\* and packages/guard/src/gumshoe/guard/\*\*) and install the Model Functional Effect plugin so its hook checks edits to core
- [ ] Add pre-commit hooks and a Makefile with the common tasks
- [ ] Add Docker Compose with Postgres and an OTLP-capable trace backend
- [ ] Add GitHub Actions: lint, types and tests, with a Postgres service
- [ ] Add an import-boundary check that fails if `core` imports an I/O library, Pydantic or asyncio. Use a ruff.toml inside the core folder (banned imports, print, naive datetimes) and keep the plugin's hook for local edits
- [ ] Confirm every planned dependency supports the chosen Python version

**Acceptance.** `uv sync` and the full check pass on a clean clone. CI is green on the skeleton. A deliberate I/O import inside `core` makes the check fail, and so does a deliberate use of Pydantic, asyncio or datetime.now there.

## M1 · Core contract

**Goal.** The pure heart of Gumshoe, fully tested before anything touches a database or a model. See Technical Design: run state machine, events, commands, policy interface.

**Tasks**

- [ ] Event envelope and the Phase 1 event types as frozen dataclasses with smart constructors, each with `schema_version`
- [ ] Command types: `CallLLM`, `CallTool`, ScheduleInput (time as a command), and stubs for `WriteMemory` and `RequestApproval`
- [ ] Immutable `RunState` and `fold(events) -> RunState`
- [ ] The state machine with the full transition table. A (state, input) pair with no transition returns the same state and no commands
- [ ] `step(state, input) -> StepResult` with the default ReAct agent. Inputs carry input\_id and now, and results carry call\_id, step\_id and attempt, so a stale result is ignored
- [ ] Policy interface, verdict types and the combination rule, evaluated inside `step`
- [ ] Injected `Clock` and `IdGenerator` ports at the edge only (time and ids arrive as input fields, delays are ScheduleInput commands), and deterministic `command_id`
- [ ] Hypothesis state-machine tests for the invariants, the replay property (the final state equals the fold of the recorded inputs), and golden snapshots of every event type

**Acceptance.** Every allowed and every forbidden transition has a test. `fold(events)` equals the live state after each step. The same inputs always give the same commands and events. `core` has no I/O imports, passes the plugin's purity check, and an unhandled pair changes nothing.

## M2 · Event store

**Goal.** A durable, append-only log behind one interface, working on SQLite for development and Postgres for production. See Technical Design: event store.

**Tasks**

- [ ] Define the `EventStore` port in `core`
- [ ] SQLite implementation (single worker only)
- [ ] Postgres implementation with the schema from the design, including the `runs` projection, the inbox and the outbox, all written in the same transaction
- [ ] Optimistic concurrency on commit, fenced by lease\_epoch, with duplicate inputs rejected through the inbox
- [ ] `claim` and `renew` for leases, using `SKIP LOCKED` on Postgres
- [ ] Versioned migrations (plain SQL files or a migration tool)
- [ ] Outbox operations (due, mark\_done) with due\_at for timers, and one contract test suite run against both stores

**Acceptance.** The same contract tests pass on both stores. Two concurrent appends to the same run: exactly one succeeds. Two workers claiming at once never claim the same run. A duplicate input id is ignored, a commit with a stale epoch is rejected, and a due outbox row is returned until it is marked done.

## M3 · Runtime v0

**Goal.** One run works end to end: a model proposes a tool call, Gumshoe validates it, heals it if needed, executes it, and finishes. See Technical Design: commands, tool router and self-healing.

**Tasks**

- [ ] LLM adapter port, a `FakeLLM`, and one real provider adapter (the provider you use first)
- [ ] Tool registry with Pydantic argument models and JSON Schema generation
- [ ] Boundary parsing of model replies and tool arguments (Pydantic, in runtime) into typed or malformed calls, error classification, and the error classes from the design
- [ ] Self-healing: up to 3 attempts, with truncated and redacted error feedback to the model. The attempt number lives in RunState and travels with each result
- [ ] Transient retries as ScheduleInput commands, with exponential backoff, jitter supplied as an input field, and their own limit
- [ ] The shell loop for a single run: load, `step`, commit (inbox, events, outbox), drain the outbox, repeat
- [ ] Recorded-response tests for the real adapter, with keys scrubbed from the recordings

**Acceptance.** Invalid arguments are retried up to three times with the error fed back, and a later valid attempt succeeds. After three failures the run ends `FAILED` with `retries_exhausted`. Transient errors do not consume attempts. A late result from an abandoned call is ignored. A full run works against the real event store.

## M4 · Orchestrator, API and CLI

**Goal.** Many runs at once, across several workers, surviving crashes. See Technical Design: orchestrator and API.

**Tasks**

- [ ] Asyncio scheduler with a semaphore bounding concurrent runs
- [ ] Claim loop with lease heartbeat, epoch fencing and loss-of-lease handling
- [ ] Backpressure: claim only with a free slot, and return 429 from `POST /runs` when the queue is full
- [ ] Cancellation, both at step boundaries and by cancelling the in-flight task
- [ ] Crash recovery: rebuild state by folding the log, drain unfinished outbox rows, including due timers, with the same row id and `command_id`, and fail non-idempotent tools with `unsafe_resume`
- [ ] Outbox worker that executes commands, a scheduler that fires due timers, graceful shutdown, and per-run error isolation
- [ ] FastAPI endpoints, with a single bearer token for authentication
- [ ] CLI: `serve`, `worker`, `run`, `ps`, `trace`, `cancel`

**Acceptance.** 100 concurrent runs against a `FakeLLM` with latency all complete, and the event-loop lag measured in the load harness stays below a threshold you set after the first measurement. Killing a worker mid-run lets another worker finish the run without repeating a tool's side effect, and a timer set before the crash still fires after recovery. A cancel moves the run to `HALTED`.

## M5 · Cost kill-switch

**Goal.** A runaway agent is slowed, then stopped, before the bill grows. See Technical Design: cost kill-switch.

**Tasks**

- [ ] Versioned price table, with the version recorded in `RunSubmitted`, and the `unknown_model` behavior
- [ ] Projected-cost calculation and a token estimate with a safety margin
- [ ] Soft-limit downgrade to the fallback model
- [ ] Hard-cap halt that records the numbers in `RunHalted`
- [ ] Step limit and repeated-call detection
- [ ] Budget settings as run-level overrides of the defaults
- [ ] The runaway fake agent, used by both tests and the demo
- [ ] Property test for the invariant: no `CallLLM` is issued when its projected cost exceeds the remaining budget

**Acceptance.** The runaway agent is downgraded at the soft limit and halted at the cap. The halt event names the policy and the numbers behind it. The invariant test passes.

## M6 · Tracing

**Goal.** Every run is visible as a trace in a standard backend. See Technical Design: tracing.

**Tasks**

- [ ] Pure event-to-span mapping in `core`
- [ ] Subscriber in `runtime` that creates OpenTelemetry spans from appended events
- [ ] GenAI attribute mapping in one table, with the convention version pinned
- [ ] OTLP exporter configuration, and Jaeger in the Compose file
- [ ] A trace-tree test using an in-memory span exporter
- [ ] Span timing taken from event timestamps

**Acceptance.** One run appears as a tree: run, steps, LLM and tool calls, with token counts and cost. A halted run shows the policy decision that stopped it.

## M7 · Demo and release prep

**Goal.** A stranger can run the demo and understand the value in a few minutes.

**Tasks**

- [ ] `gumshoe demo`: offline, 100 runs with a fake LLM, one runaway
- [ ] Mascot poses for each run status in the CLI output
- [ ] README with a short demo recording, the quickstart, and the architecture diagram
- [ ] Docs for writing a tool and an agent, and an examples folder
- [ ] Version 0.1.0, changelog, and a dry-run publish to TestPyPI
- [ ] Security policy and contribution notes

**Acceptance.** On a fresh machine the demo runs in a few commands. Someone who did not build it reads the README and can describe what Gumshoe does. The release checklist below is complete.

## Definition of done

A task is done only when all of these hold:

- Code is typed, and pyright (strict), Ruff and the functional-core check pass
- Logic in `core` has tests, including property tests where an invariant applies
- No new I/O, async code or dependency in `core`
- If an event payload changed, its `schema_version` was bumped and an upcaster added, and the golden snapshot was updated on purpose
- The Technical Design is updated if behavior changed
- CI is green

## Demo script

Command names follow the proposed CLI in the Technical Design.

1. Start Postgres and the trace backend with Docker Compose.
2. Run `gumshoe demo`. It starts a worker and submits 100 runs against a fake LLM.
3. Watch the runs progress and complete, each shown with its status pose.
4. One run is the runaway. Show its cost climbing, the downgrade at the soft limit, and the halt at the hard cap.
5. Run `gumshoe trace <run_id>` on the runaway to read the event timeline.
6. Open the trace backend and show the same run as a tree of steps, LLM calls and tool calls, with the policy decisions.
7. Show a run whose tool call had invalid arguments: the rejection, the error fed back, and the corrected call.
8. Kill a worker mid-run and show another worker finishing it.

## Release checklist

- [ ] Names confirmed: PyPI distribution, import name, GitHub, domain, trademark check
- [ ] Apache 2.0 license file and headers in place
- [ ] README, quickstart and architecture diagram reviewed by a fresh reader
- [ ] Examples run in CI
- [ ] No secrets or personal data in the repository, recordings or fixtures
- [ ] Version 0.1.0 and a changelog
- [ ] Dry-run publish to TestPyPI
- [ ] PyPI Trusted Publishing configured for the real release
- [ ] Security policy with a contact
- [ ] A line in the README stating what is stable and what is not
- [ ] Employment contract checked for intellectual property conflicts, as noted in the PRD risks

## Risks and what to cut first

| Risk | Early signal | Mitigation |
| --- | --- | --- |
| The event schema or state machine needs a redesign late | Frequent payload changes during M3 and M4, or a step that needs the clock or a random value | Finish M1 first, protect it with invariants and golden snapshots, and version every event |
| Crash recovery is harder than expected | Duplicate or lost tool effects in the crash test | Write the crash test early in M4, and rely on `command_id` plus the `unsafe_resume` rule |
| Cost estimates are inaccurate | Real cost far from projected in the demo | Use provider token counting, keep a margin, and document that overshoot is bounded by estimation error |
| Scope creeps into Phase 2 (pause and resume, sandbox) | Work on approvals before M5 is done | Keep `PAUSED` in the schema but unused, and reject new features until the exit criteria pass |
| A key library does not support Python 3.13 | Install or type errors in M0 | Fall back to Python 3.12 |
| Limited time next to a full-time job | A milestone stalls for weeks | Cut in the order below |

**Cut first, in this order**

1. Provider breadth: ship one provider only
2. CLI polish and mascot art: plain text output is enough
3. Multi-worker support on SQLite: it only needs to work on Postgres
4. Refinements to backpressure

**Never cut:** the event schema, the state-machine tests, the kill-switch invariant, and crash recovery.

## Related documents

- [Product Requirements Document](https://claude.ai/artifact/YYfatFCfAKu8DYe4LXMcAU): what Gumshoe is, why, and the roadmap
- [Technical Design (Phase 1)](https://claude.ai/artifact/PTqii5ttUKJ7AbCkLTq4Ad): how each part works
- [Developer Guide](https://claude.ai/artifact/CD1e4Pxuz5KNzPgqpLu6p6): repository layout, tooling, conventions, CI, AGENTS.md
