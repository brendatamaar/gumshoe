# Gumshoe — Technical Design (Phase 1)

Oct 31, 2026 · @brendatama · Status: Draft v0.1 · Companion to the PRD

## Purpose and scope

This document turns the PRD into an implementable design for **Phase 1**: the `core` contract, the async orchestrator, the tool router with self-healing retries, the cost kill-switch, and OpenTelemetry tracing. It also fixes the parts of `core` that later phases depend on (pause/resume and replay), so they are not redesigned later.

**In scope:** run state machine, event and command schemas, policy interface, event store, orchestrator, tool router, kill-switch, tracer, API, CLI.

**Out of scope for Phase 1** (designed only where `core` needs it): approval gate, sandbox, context budget, memory, sanitizer, replay and CI gate, shadow runs, swarm, browser fallback.

**Conventions.** Values marked *default* or *proposed* are starting points to confirm during the build. Open decisions are collected in the last section.

## System overview

Every run is driven by one loop that alternates between a **pure core** and an effectful **shell**.

&#91;embedded content: step cycle · pure core, effectful shell\]

1. The shell folds the run's events into its current state and takes the next input, for example an LLM response or a tool result.
2. The core computes `(state₂, commands, events) = step(state₁, input)`. Inside `step`, every policy judges each command before it is returned. The core never does I/O.
3. The shell appends the events to the log, then executes the commands: LLM calls and tool calls.
4. Each result becomes the next input, and the loop repeats until the run reaches a terminal state.

Because state is always a fold over the log, a run can resume on any worker, replay offline, and be traced without extra instrumentation.

## Functional core and shell

The core follows the Model Functional Effect method: `(state₂, commands) = step(state₁, input)`, with the LLM and the tools as effects. The project profile is `distributed`, because a crash or a redelivery must not duplicate a tool side effect or lose a result. The plugin's hook enforces the rules below on edits made by Claude. Ruff and pyright enforce them in CI for every other edit.

| Rule | How Gumshoe applies it |
| --- | --- |
| Plain data in core | States, inputs, commands, events and verdicts are frozen dataclasses, grouped in unions that end in `assert_never`. Validation sits in smart constructors. `core` does not import Pydantic. |
| Parse at the boundary | Pydantic wire models live in `runtime`. They validate HTTP bodies, database rows, LLM replies and tool arguments, then map onto core types. A rejection is a returned value, never an exception. |
| Time is data and a command | Every input carries `now`. Timeouts, backoff and lease checks are `ScheduleInput` commands, stored in the outbox with `due_at`. `core` never imports `time`, `random` or `asyncio`. |
| Unhandled pairs are no-ops | A `(state, input)` pair with no transition returns the same state and no commands. |
| Results are matched | Results carry `call_id`, `step_id` and `attempt`. A stale or duplicate result is ignored. Counters for spend, steps, repeats and attempts live in `RunState`. |
| Persistence is one transaction | Inbox, events, projection and outbox commit together. Ids are deterministic. A worker, not `step`, drains the outbox. |
| One interpreter | The shell maps each command to a capability behind a `Protocol`: LLM, tool executor, event store, scheduler. `asyncio` is used only here. |

## Core concepts

| Term | Meaning |
| --- | --- |
| Run | One execution of an agent for one task. Identified by `run_id`, with a status and an event log |
| Step | One turn of the loop: an input goes in, commands and events come out. Identified by `step_id` |
| Event | An immutable, versioned fact about a run, appended to the log |
| Command | A request for a side effect, returned by the core and executed by the shell (`CallLLM`, `CallTool`, ...) |
| Input | What the shell feeds back to the core: a result, a decision, a cancellation |
| Policy | A pure check that returns a verdict for a proposed command |
| Verdict | `allow`, `deny`, `pause`, or `downgrade` |
| Snapshot | A cached fold of the log at a given sequence number, used only to speed up loading |
| Shell | Everything that does I/O: database, LLM providers, tools, clock, HTTP API |

## Run state machine

Six states: `PENDING`, `RUNNING`, `PAUSED`, `SUCCEEDED`, `FAILED`, `HALTED`. The last three are terminal and absorbing.

| From | Trigger | To | Notes |
| --- | --- | --- | --- |
| PENDING | Scheduler claims the run | RUNNING | A lease is acquired |
| PENDING | Cancel requested | HALTED | The run never started |
| RUNNING | A step finishes and work remains | RUNNING | The normal loop |
| RUNNING | The agent returns its final answer | SUCCEEDED | Terminal |
| RUNNING | Retries exhausted, or a fatal error | FAILED | Terminal |
| RUNNING | A policy halts the run, or cancel requested | HALTED | Terminal |
| RUNNING | A policy returns `pause` | PAUSED | State is persisted and the worker releases the run (Phase 2 feature) |
| PAUSED | Approval decision or resume trigger | RUNNING | Phase 2. A denial is fed to the agent as a denied tool result |
| PAUSED | Approval timeout or cancel | HALTED | Terminal |

**FAILED versus HALTED.** `FAILED` means the run could not do its job (errors, retries exhausted). `HALTED` means the run was stopped on purpose (a policy, a cancel, a timeout). Dashboards and alerts treat them differently.

**Forbidden transitions.** Anything out of a terminal state; `PENDING` straight to `PAUSED`, `SUCCEEDED` or `FAILED`; `PAUSED` straight to `SUCCEEDED` or `FAILED`. When `step` receives an input that does not fit the current state (a late result, a duplicate, a cancel after the run ended), it returns the same state and no commands. It never raises, and it never emits a command from a terminal state. The shell records the input as ignored in the inbox, so the case stays visible in logs and metrics.

`PAUSED` is part of the schema from day one even though the approval gate ships later, so the event log and the state never need a breaking change to add it.

## Events

Every event shares one envelope. In `core`, each payload is a frozen dataclass chosen by `type`. In `runtime`, a matching Pydantic wire model validates and serializes it.

| Field | Type | Meaning |
| --- | --- | --- |
| `event_id` | UUID | Globally unique |
| `run_id` | UUID | The run this event belongs to |
| `seq` | integer | Per-run, starts at 1, gapless. Unique together with `run_id` |
| `step_id` | integer | Step number within the run (0 for run-level events) |
| `type` | string | Discriminator for the payload model |
| `schema_version` | integer | Version of this event type's payload |
| `command_id` | UUID, optional | Links a result event to the command that caused it |
| `ts` | timestamp (UTC) | Supplied by the shell through the injected clock |
| `payload` | JSON | Validated by the wire model for `type` |

**Phase 1 event types**

| Type | Emitted when | Key payload |
| --- | --- | --- |
| `RunSubmitted` | A run is created | Agent reference, input, budget, price table version |
| `RunStarted` | PENDING to RUNNING | Worker id |
| `StepStarted` | A step begins | Step number |
| `LLMRequested` | A `CallLLM` command is issued | Model, messages, tools |
| `LLMResponded` | The LLM result arrives | Content, tool calls, token usage, cost |
| `ToolCallProposed` | The model asks for a tool | Tool name, raw arguments, call id |
| `PolicyEvaluated` | Policies judged a command | Each policy and its verdict, reason |
| `ToolCallValidated` | Arguments passed validation | Validated arguments |
| `ToolCallRejected` | Arguments failed validation | Truncated error trace, attempt number |
| `ToolExecuted` | The tool returned | Result, duration |
| `ToolFailed` | The tool raised | Error class, truncated message, attempt |
| `RetryScheduled` | A self-healing or transient retry | Attempt, error class, backoff |
| `CostRecorded` | After every LLM call | Cumulative cost, model |
| `RunPaused`, `RunResumed` | Phase 2 | Reason, decision |
| `RunSucceeded` | Final answer | Output |
| `RunFailed` | Unrecoverable error | Reason |
| `RunHalted` | Policy halt or cancel | Reason, policy name, numbers that triggered it |

**Evolution.** Events are never edited. New fields are added as optional with defaults. A breaking change means a new `schema_version` plus an upcaster that converts old payloads when they are loaded.

**Content capture.** Replay needs the recorded LLM and tool contents, so the default capture mode is `full` in development. A `metadata_only` mode is available for production, and a redaction hook runs before any event is appended. Runs recorded in `metadata_only` cannot be replayed (see open decisions).

## Commands and the pure step function

| Command | Fields | Executed by | Phase |
| --- | --- | --- | --- |
| `CallLLM` | `command_id`, model, messages, tools, parameters | LLM adapter | 1 |
| `CallTool` | `command_id`, `call_id`, tool name, validated arguments | Tool executor | 1 |
| `WriteMemory` | `command_id`, key, value | Memory backend | Stub in 1 |
| `RequestApproval` | `command_id`, action, context | Approval service | Stub in 1, real in 2 |

**Idempotency.** `command_id` is derived deterministically from (`run_id`, `step_id`, `index`), for example with `uuid5`, and the outbox row id is `<input_id>#<index>`. After a crash the same command gets the same ids, so the shell can tell it already ran, and side-effecting tools receive `command_id` as an idempotency key. `uuid5` is a pure function and is allowed in `core`; `uuid4` is not.

The step function takes the current state and one input, and returns the new state, the commands to execute, and the events to append:

```python
@dataclass(frozen=True)
class StepResult:
    state: RunState
    commands: tuple[Command, ...]
    events: tuple[Event, ...]

def step(state: RunState, input: Input) -> StepResult: ...
```

Inputs are `Submit`, `LLMResult`, `ToolResult`, `ToolError`, `ApprovalDecision`, `Cancel` and `TimerFired`. Every input carries an `input_id` and `now`, the time read once at the edge. `LLMResult`, `ToolResult` and `ToolError` also carry the `call_id`, `step_id` and `attempt` of the command that produced them, and `step` ignores a result that does not match the pending call. A delay is a command, `ScheduleInput(after_ms, input)`, executed by the scheduler: it stores the input in the outbox with a `due_at` and feeds it back as `TimerFired` when it is due.

**Default agent.** Phase 1 ships a `ReActAgent`, itself a pure reducer. On `Submit` it issues `CallLLM`. When the model asks for tools it validates the arguments and issues `CallTool`. Each tool result is appended to the messages and followed by another `CallLLM`. A response without tool calls ends the run with `RunSucceeded`. Users customize the system prompt, tools, model and step limit, or replace the agent by implementing this protocol (proposed shape):

```python
class Agent(Protocol):
    def on_input(self, view: AgentView, input: Input) -> AgentDecision: ...
```

`AgentDecision` is one of: call the LLM or tools, `Finish(output)`, or `Fail(reason)`.

**Determinism rule.** `step` must not read the clock, randomness, environment, files or network, and `core` does not import `asyncio`. Time and identifiers arrive as data in the input (`now`, `input_id`). The same inputs always produce the same commands and events.

## Policy interface

```python
class Policy(Protocol):
    name: str
    def evaluate(self, ctx: PolicyContext, action: ProposedAction) -> Verdict: ...
```

`PolicyContext` is a read-only view of the run: status, step number, cumulative cost, counters, configuration. `ProposedAction` wraps the command about to be issued (`CallLLM` or `CallTool`) plus metadata such as estimated cost and tool risk tags.

| Verdict | Effect |
| --- | --- |
| `Allow` | The command is issued unchanged |
| `Deny(reason, halt)` | With `halt=true` the run moves to `HALTED`. With `halt=false` the agent receives a denied result and may adapt |
| `Pause(reason, approval)` | The run moves to `PAUSED` and waits for a decision (Phase 2) |
| `Downgrade(replacement)` | The command is issued with replaced parameters, for example a cheaper model |

**Where policies run.** Inside `step`, before a command is returned. Policies are pure, so this keeps the core deterministic and replay includes the policy decisions. The shell never executes a command the core did not return.

**Combination.** All policies run in declared order, without short-circuiting, so every decision is logged. The result is the most restrictive verdict: `Deny`, then `Pause`, then `Downgrade`, then `Allow`. Downgrades compose in policy order.

**Purity.** A policy holds no counters. Spend, step counts and repeat counts live in `RunState` and reach the policy through `PolicyContext`. Each evaluation emits one `PolicyEvaluated` event listing every policy and its verdict.

## Cost kill-switch

The kill-switch is a policy in `guard` that reads spend and loop counters from the run state.

| Setting | Default | Meaning |
| --- | --- | --- |
| `max_cost_usd` | 0.50 | Hard cap per run |
| `soft_limit_ratio` | 0.8 (proposed) | Fraction of the cap at which the run is downgraded |
| `fallback_model` | none | Cheaper model used on downgrade. Downgrade is disabled without it |
| `max_steps` | 25 (proposed) | Loop depth limit |
| `max_repeat_calls` | 3 (proposed) | Identical tool calls with identical arguments allowed before halting |
| `unknown_model` | `deny` | What to do when the price table has no entry for the model |

**Evaluating each `CallLLM`**

1. Compute `projected = spent + estimated input cost + max output tokens x output price`.
2. If `spent` is at or above `soft_limit_ratio x max_cost_usd` and a fallback model exists, return `Downgrade` to the fallback and recompute the projection with its prices.
3. If the projection still exceeds the cap, return `Deny(halt=true)` and record the numbers in `RunHalted`.
4. For any command, return `Deny(halt=true)` when the step count passes `max_steps` or the same tool call repeats more than `max_repeat_calls` times.

**Price table.** A versioned configuration that maps each model to input and output prices per million tokens. The version in use is recorded in `RunSubmitted`, so a replay recomputes the same costs. Prices come from provider documentation and are updated by hand.

**Token estimate.** Use the provider's token counting when it exists, otherwise a conservative heuristic with a safety margin. Cumulative cost always uses the real usage reported in `LLMResponded`.

**Invariant.** No `CallLLM` is issued when its projected cost exceeds the remaining budget. Real cost can still differ from the estimate, so overshoot is bounded by estimation error. That is documented behavior, not a bug.

## Tool router and self-healing

**Registry.** Tools are registered with a decorator. Each tool has a name, a description, a Pydantic arguments model (its JSON Schema comes from `model_json_schema()`), a timeout, an idempotency flag, and optional risk tags used by the approval gate later.

**Validation flow.** The shell parses the model's reply at the boundary: the tool must be registered, and its arguments are decoded with the tool's Pydantic model (`model_validate`). Nothing unparsed reaches `core`. A valid call arrives in `LLMResult` as a typed proposal, and an invalid one arrives as a `malformed` call that carries the error list. The core answers a valid call with `ToolCallValidated` and `CallTool`. It answers a malformed call with `ToolCallRejected` and, while attempts remain, a new `CallLLM` that carries the error.

**Error classes**

| Class | Examples | Handling | Counts toward attempts |
| --- | --- | --- | --- |
| `VALIDATION` | Invalid JSON, wrong types, missing field, unknown tool | Feed the error back to the model as a tool result and ask it to correct the call | Yes |
| `TOOL` | The tool raised, or the target API rejected the input | Feed a truncated error back. The model may fix the arguments or pick another tool | Yes |
| `TRANSIENT` | Timeout, rate limit, 5xx, connection reset | Retry the same command after a backoff, with no new LLM call. The core emits ScheduleInput with a delay computed from the attempt count and a jitter value supplied in the input | No, has its own limit (default 3) |
| `POLICY` | A policy returned `deny` | Not retried. Returned to the agent as denied, or the run halts | No |
| `FATAL` | Authentication failure, configuration error, unknown model | Fail the run | Not applicable |

**Attempts.** At most 3 self-healing attempts per tool call, counting `VALIDATION` and `TOOL` errors. When the third attempt fails, the run moves to `FAILED` with reason `retries_exhausted`.

**Error feedback.** What goes back to the model is the validation error list or the tool error message, not an internal stack trace. It is truncated (proposed limit: 2,000 characters) and passed through the redaction hook, because it enters both the prompt and the event log.

**Structured output.** When the provider supports native structured output, use it to reduce validation failures. Constrained decoding applies only to self-hosted models.

**Parallel tool calls.** Sequential in Phase 1, for deterministic ordering. Parallel execution can come later without changing events.

## Orchestrator

A worker is one process with one asyncio event loop. Several workers can run against the same Postgres.

- **Claiming.** Workers claim runs with `SELECT ... FOR UPDATE SKIP LOCKED` on runs that are `PENDING`, or `RUNNING` with an expired lease. A claim sets `lease_owner` and `lease_expires_at`.
- **Lease renewal.** While a run is active the worker renews its lease on a heartbeat (about a third of the lease length). If renewal fails, the worker cancels that run's task, because it no longer owns the run.
- **Concurrency.** An `asyncio.Semaphore(max_concurrent_runs)` bounds active runs (default 100). Each run is one task. LLM and tool calls are awaited with timeouts. Synchronous or blocking tools run through `asyncio.to_thread`, or a process pool when CPU-bound, so the loop is never blocked.
- **Backpressure.** Workers claim only when a semaphore slot is free. `POST /runs` returns 429 when the pending queue passes `max_pending`.
- **Cancellation.** `POST /runs/{id}/cancel` sets `cancel_requested` on the run. The worker checks it at each step boundary and also cancels the in-flight task, so a cancel takes effect promptly. The run ends `HALTED`.
- **Crash recovery.** When a worker dies its lease expires and another worker claims the run. Each claim increments `lease_epoch`, which fences the old owner: a late commit from it fails the version check. The new worker rebuilds state by folding the event log, then drains the outbox. Every row not marked done, including timers whose `due_at` has passed, is executed again with the same row id and `command_id`. A re-issued LLM call may be billed twice, and this is documented.
- **Unsafe resume.** A tool not marked idempotent is never re-executed automatically. The run is marked `FAILED` with reason `unsafe_resume` for manual review.
- **Graceful shutdown.** On `SIGTERM` the worker stops claiming, lets running steps finish within a grace period, and releases its leases.
- **Error isolation.** An exception inside one run's task never stops the loop. It is caught at the run boundary and recorded as `RunFailed`.
- **No process-level globals.** All run state lives in the event log.

## Event store

Four tables hold everything. `events` is the source of truth, and `runs` is a projection of it, so claims and listings do not need to fold the log. `inbox` records every input once, by `input_id`, so a redelivered input is ignored. `outbox` holds the commands the core emitted, including timers, until a worker marks them done. One transaction writes all four.

```sql
CREATE TABLE runs (
  run_id            uuid PRIMARY KEY,
  status            text NOT NULL,
  agent_ref         text NOT NULL,
  created_at        timestamptz NOT NULL,
  updated_at        timestamptz NOT NULL,
  lease_owner       text,
  lease_expires_at  timestamptz,
  lease_epoch       bigint NOT NULL DEFAULT 0,
  cancel_requested  boolean NOT NULL DEFAULT false,
  last_seq          bigint NOT NULL DEFAULT 0
);

CREATE TABLE events (
  run_id          uuid NOT NULL REFERENCES runs (run_id),
  seq             bigint NOT NULL,
  event_id        uuid NOT NULL UNIQUE,
  step_id         integer NOT NULL,
  type            text NOT NULL,
  schema_version  integer NOT NULL,
  command_id      uuid,
  ts              timestamptz NOT NULL,
  payload         jsonb NOT NULL,
  PRIMARY KEY (run_id, seq)
);

CREATE TABLE inbox (
  run_id       uuid NOT NULL REFERENCES runs (run_id),
  input_id     text NOT NULL,
  received_at  timestamptz NOT NULL,
  outcome      text NOT NULL,          -- handled | ignored
  PRIMARY KEY (run_id, input_id)
);

CREATE TABLE outbox (
  outbox_id   text PRIMARY KEY,        -- <input_id>#<index>
  run_id      uuid NOT NULL REFERENCES runs (run_id),
  command_id  uuid NOT NULL,
  kind        text NOT NULL,           -- CallLLM | CallTool | ScheduleInput | ...
  payload     jsonb NOT NULL,
  due_at      timestamptz NOT NULL,
  done_at     timestamptz
);

CREATE INDEX runs_claimable ON runs (status, lease_expires_at);
CREATE INDEX outbox_due ON outbox (due_at) WHERE done_at IS NULL;
```

**Port.** The core defines the interface; implementations live in the runtime.

```python
class EventStore(Protocol):
    async def commit(self, run_id: UUID, expected_seq: int, epoch: int, input_id: str,
                     events: Sequence[Event], outbox: Sequence[OutboxRow]) -> CommitResult: ...
    async def load(self, run_id: UUID, after_seq: int = 0) -> list[Event]: ...
    async def claim(self, worker_id: str, limit: int, lease_seconds: int) -> list[Claim]: ...
    async def renew(self, run_id: UUID, worker_id: str, epoch: int, lease_seconds: int) -> bool: ...
    async def due(self, limit: int) -> list[OutboxRow]: ...
    async def mark_done(self, outbox_id: str, epoch: int) -> None: ...
```

**Commit semantics.** In one transaction: insert the inbox row (a duplicate `input_id` ends the call as a no-op), check that `runs.last_seq` equals `expected_seq` and that `lease_epoch` matches the caller's epoch, insert the events with consecutive `seq` values, update `status`, `last_seq` and `updated_at`, and insert the outbox rows. A mismatch raises a concurrency error, and the caller reloads and retries. Two workers writing to the same run can never both succeed.

**SQLite for development.** Same schema with text and integer types. SQLite has no `SKIP LOCKED`, so multi-worker behavior is tested on Postgres only. The same contract test suite runs against both stores.

**Deferred.** Snapshots (folding is cheap until runs get long), retention and archival, and per-tenant partitioning.

## Tracing

The tracer is a subscriber to events after they are appended. It is not part of `step`, so tracing can never change what a run does.

**Placement.** The mapping from events to span descriptions is pure and lives in `core`. Creating OpenTelemetry spans and exporting them lives in `runtime`. This refines the PRD, which lists the tracer under `core`: the contract stays in `core`, and the I/O stays at the edge.

| Span | Built from | Notable attributes |
| --- | --- | --- |
| `gumshoe.run` (root) | `RunStarted` to the terminal event | `gumshoe.run_id`, `gumshoe.run.status` |
| `gumshoe.step` | `StepStarted` to the next step or terminal event | `gumshoe.step_id` |
| LLM call | `LLMRequested` to `LLMResponded` | `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gumshoe.cost_usd` |
| Tool call | `ToolCallValidated` to `ToolExecuted` or `ToolFailed` | `gen_ai.tool.name`, `gen_ai.tool.call.id` |
| Policy decision | `PolicyEvaluated` | Span event on the step: `gumshoe.policy.name`, `gumshoe.policy.verdict` |

**Timing.** Spans use the events' own timestamps, so a trace rebuilt offline from a stored log looks the same as the live one.

**Convention status.** The OpenTelemetry GenAI semantic conventions were still marked as being in development in the source consulted, so attribute names can change. Pin the convention version in the code and keep the mapping in one table that is easy to update ([source](https://clickhouse.com/pt-BR/resources/engineering/what-is-ai-agent-observability)).

**Exporters.** Phase 1 ships OTLP only (Jaeger or Grafana Tempo in development). Other backends are reached through their OTLP endpoints if they offer one, or through adapters later.

## API, CLI and mascot

**HTTP API (FastAPI)**

| Endpoint | Purpose | Response |
| --- | --- | --- |
| `POST /runs` | Submit a run: agent reference, input, optional budget overrides | 202 with `run_id`, or 429 when the queue is full |
| `GET /runs/{run_id}` | Status, cost so far, last event sequence | 200 |
| `GET /runs/{run_id}/events?after_seq=` | The event log, paginated | 200 |
| `POST /runs/{run_id}/cancel` | Request cancellation | 202 |
| `GET /healthz` | Liveness, running runs, pending queue depth | 200 |

Phase 2 adds an approvals endpoint. Phase 1 authentication is a single bearer token from the environment, or none for local development. It is not multi-tenant.

**CLI (proposed commands)**

| Command | Purpose |
| --- | --- |
| `gumshoe demo` | Offline demo: 100 runs with a fake LLM, one runaway halted |
| `gumshoe serve` | Start the API |
| `gumshoe worker` | Start a worker |
| `gumshoe run <agent> --input ...` | Submit a run |
| `gumshoe ps` | List recent runs with status and cost |
| `gumshoe trace <run_id>` | Print the event timeline |
| `gumshoe cancel <run_id>` | Cancel a run |

**Mascot states.** The detective appears next to each status in CLI output. It is drawn as small text art, and is disabled for non-interactive output, when `NO_COLOR` is set, or with `--no-mascot`.

| Status | Pose |
| --- | --- |
| `PENDING` | Briefing the case, notebook open |
| `RUNNING` | Investigating, magnifying glass up |
| `PAUSED` | Waiting by the door, checking a watch |
| `SUCCEEDED` | Case closed, stamping the file |
| `FAILED` | Puzzled, question mark over the file |
| `HALTED` | File sealed, stop sign |

## Testing strategy

| Layer | What it checks | Tools |
| --- | --- | --- |
| Pure core | Transitions, `step`, policies | pytest, Hypothesis |
| Event store contract | One suite, run against SQLite and Postgres | pytest parametrized, Postgres container |
| Orchestrator integration | Fake LLM and tools, real store | pytest-asyncio |
| Provider adapter | Recorded provider responses | pytest-recording (VCR.py) |
| Event schema stability | Golden JSON for every event type, so accidental schema changes fail | syrupy snapshots |
| Load | 100 concurrent runs, event-loop lag | Custom harness, not in default CI |
| Crash | Kill a worker mid-run, another finishes it | Subprocess plus lease expiry |

**Invariants.** These must always hold. Check them with Hypothesis state-machine tests.

1. `seq` is gapless and strictly increasing within a run.
2. Terminal states absorb: no input changes a terminal run.
3. `fold(events)` equals the live state after every step.
4. The same inputs through `step` give identical commands and events (determinism).
5. No `CallLLM` is issued when its projected cost exceeds the remaining budget.
6. A command is never executed twice with side effects: re-execution after a crash reuses its `command_id`.
7. Every allowed transition is covered by a test, and every forbidden transition is rejected.

Invariants added by the functional-core rules, each checked after every step of a generated input sequence:

- A `(state, input)` pair with no transition returns the same state and no commands.
- A result whose `call_id`, `step_id` or `attempt` does not match the pending call changes nothing.
- Spend, steps and attempts never exceed their limits: the step that would exceed one ends in `FAILED` or `HALTED`.
- Every non-terminal state either emits a command or is waiting for a named input (a result or a timer).
- Replaying the recorded inputs through `step` from the initial state gives the final state of the live run.
- Outbox ids are a pure function of `input_id` and command index, so a replayed step produces the same rows.

**Fakes.** `FakeLLM` (scripted responses, configurable latency, failures and token usage), `FakeTool`, `FakeClock`, a sequential `IdGenerator`, and a runaway fake agent that keeps requesting the same failing tool. The runaway agent serves both the kill-switch tests and the demo.

**Overhead measurement.** The PRD target is a runtime overhead under 10 ms per step at p95. Define it as the time from receiving a result to issuing the next command, excluding I/O waits, and measure it in the load harness.

## Security and privacy

- **Arguments are untrusted.** Model output is validated against the tool's schema before anything runs. The schema is the trust boundary.
- **Feedback is sanitized.** Error text sent back to the model is truncated and redacted, since it enters both the prompt and the event log.
- **Secrets stay out of events.** Tool executors read secrets from the environment or a secret manager. Secrets are never part of state, commands or events.
- **Capture modes.** `full` records contents (needed for replay). `metadata_only` records hashes, sizes and timings. A redaction hook runs before append in both modes.
- **No sandbox in Phase 1.** Tools run in the worker process, so only trusted tools should be registered. The sandbox executor arrives in Phase 2.
- **Tool results are untrusted content.** A tool result can contain instructions aimed at the model. The sanitizer that addresses this is a Phase 3 item. Until then, keep tools that fetch untrusted content separate from tools with side effects.
- **Logs.** Application logs never contain raw prompts or tool contents by default.
- **Supply chain.** Pin dependencies in the lock file, and publish packages through PyPI Trusted Publishing from CI.

## Decisions and open questions

**Decided in the PRD and carried here**

- Python 3.13 with uv, Ruff and pyright strict; asyncio, FastAPI and Pydantic v2.
- PostgreSQL for the event store and queue; SQLite for development.
- Event sourcing built in-house on Postgres; the core is a pure function of state and input.
- One policy interface for every guard.
- OpenTelemetry tracing exported through OTLP.

**Open questions.** Each has a recommended default so the build is not blocked.

| # | Question | Recommended default |
| --- | --- | --- |
| 1 | Structured output: PydanticAI, Instructor, or in-house validation? | In-house validation and retry in Phase 1, because the retry loop is part of the product and emits its own events. Use provider-native structured output where available |
| 2 | How to estimate tokens for projected cost? | Provider token counting when available, else a conservative heuristic with a margin |
| 3 | Where do prices come from and who updates them? | A versioned price table in the repo, updated by hand from provider documentation |
| 4 | What does an approval denial do? | Return a denied result to the agent and continue |
| 5 | Default content capture? | `full` in development, choose per environment in production, and document that `metadata_only` runs cannot be replayed |
| 6 | Tool idempotency contract | A flag per tool. Non-idempotent tools are never re-executed automatically, the run fails with `unsafe_resume` |
| 7 | How to generate time-ordered ids? | Check what the chosen Python version offers in the standard library, otherwise use a small library |
| 8 | Do all dependencies support Python 3.13? | Verify in the first milestone, and fall back to 3.12 if a key library lags |
| 9 | Import and distribution names, given the existing `gumshoe` package on PyPI | Decide in the first milestone, together with the GitHub and domain checks |
| 10 | Which LLM provider first? | The one you use daily, behind the adapter port |

## Related documents

- [Product Requirements Document](https://claude.ai/artifact/YYfatFCfAKu8DYe4LXMcAU): what Gumshoe is, why, and the roadmap
- [Phase 1 Implementation Plan](https://claude.ai/artifact/YXHUooeGy1BSXpzxN55oeK): what to build, in what order
- [Developer Guide](https://claude.ai/artifact/CD1e4Pxuz5KNzPgqpLu6p6): repository layout, tooling, conventions, CI, AGENTS.md
