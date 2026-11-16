# Gumshoe — Product Requirements Document

Oct 31, 2026 · @brendatama · Status: Draft v0.1

## Overview

Gumshoe is an open-source Python runtime that runs LLM agents in production with guardrails and replay. Every agent run is a durable, typed state machine, and every transition is written to an event log. From that one log, Gumshoe caps spend, blocks risky actions, pauses for human approval, traces every step, and replays runs as regression tests.

**One-line pitch:** Run your agents like production software: bounded, auditable, and replayable.

**The mascot.** Gumshoe is a small detective with a magnifying glass. Every run is a case: the detective investigates while it runs, waits by the door when it is paused for approval, closes the file when the kill-switch halts it, and reconstructs the scene when a run is replayed. The mascot appears in the CLI, the trace viewer, and the docs, and its pose always reflects the run's current state.

**Product shape.** One monorepo with five modules that share one contract: `core`, `runtime`, `guard`, `lab`, and `extras`. Phase 1 ships `core`, the orchestrator, the tool router with self-healing retries, and the cost kill-switch.

## Problem statement

Most agents work in a demo and break in production, because the loop around the model is not engineered like the rest of the stack. Five failures recur:

1. **Blocking loops.** A `while True` agent loop blocks the thread and cannot serve concurrent traffic or recover cleanly from errors.
2. **Untrusted outputs.** Models hallucinate tool parameters, so raw LLM output cannot be passed straight to an API.
3. **Runaway cost.** A loop that never converges keeps calling the model. It is an outage that arrives as an invoice.
4. **Unchecked actions.** High-stakes actions run without human sign-off, and generated code runs on the host without isolation.
5. **Silent regressions.** A prompt or code change alters which tools an agent calls, but teams only grade the final text, so a skipped safety check or a hallucinated call goes unnoticed.

These are usually solved one by one with separate libraries. Gumshoe solves them with one mechanism: a durable run state plus an event log that every module reads and writes.

## Goals and non-goals

**Goals**

- Run 100+ agent runs concurrently on one async event loop without blocking.
- Turn LLM output into validated, typed tool calls, with automatic retries that feed back the error.
- Stop a runaway run before it exceeds a spend or loop-depth budget, and fall back to a cheaper model first.
- Pause a run for human approval and resume it days later without holding a thread.
- Record every run so it can be traced, replayed deterministically, and diffed in CI.
- Stay framework-agnostic at the edges: OpenTelemetry out, adapters in.

**Non-goals**

- Gumshoe is not a prompt library or an agent framework for writing reasoning logic. Users bring their own agent logic and tools.
- It is not a hosted observability dashboard in v1. Traces export to existing backends (Jaeger, Tempo, Langfuse).
- It is not a general-purpose workflow engine like Temporal. Durable execution covers agent runs only.
- Multi-agent swarms and browser fallback are optional extras, not part of the core.

## Target users

| Persona | Context | What they need from Gumshoe |
| --- | --- | --- |
| AI engineer | Ships one or more agents to production, often alone or in a small team | A runtime that handles concurrency, retries, and cost caps so they can focus on agent logic |
| Platform / infra engineer | Owns the shared agent platform for several product teams | Central policies, audit trails, and isolation for code execution |
| Tech lead / reviewer | Approves prompt and code changes to agents | A CI gate that shows how a change altered tool-call paths and cost before merge |
| Operator / approver | Signs off on high-stakes actions (payments, deletions, outbound messages) | A clear approval request with the context needed to decide |

The primary user for v1 is the AI engineer: they adopt it bottom-up, and their trust drives later team adoption.

## Design principles

Three decisions shape every module; getting them right early makes the hardest features nearly free.

1. **Pure core.** Each step is `(state₂, commands) = f(state₁, input)`. LLM calls and tool calls are commands executed by a thin shell, never side effects inside the logic. This makes replay, shadow runs, and pause/resume deterministic and testable.
2. **Event log as the source of truth.** Every state transition emits a versioned event to an append-only store. Tracing, replay, the CI gate, and shadow comparison all read the same log.
3. **One policy interface.** Every guard implements `evaluate(state, proposed_action) → allow | deny | pause | downgrade`. The kill-switch, sanitizer, and approval gate are policies with the same shape, so new guards plug in without changing the runtime.

The payoff: a three-day approval pause, a CI replay, and a shadow run all rely on the same serializable state and event log. Built right once, they come almost for free. Built wrong, all three must be rewritten.

## Architecture

&#91;embedded content: Gumshoe architecture · 5 modules, 1 event log\]

The runtime writes every transition to the event log through `core`; guard plugs into the same policy interface; lab only reads the log. That is why replay and the CI gate need no extra instrumentation.

## Module requirements

Priority: **P0** = Phase 1 (MVP), **P1** = Phase 2, **P2** = Phase 3, **P3** = later. Each idea number from the original list is shown in brackets.

### core: the shared contract

| Feature | Requirement | Priority |
| --- | --- | --- |
| Event schema | Versioned events with `run_id`, `step_id`, type, payload, timestamp, schema version | P0 |
| Run state machine | States `PENDING → RUNNING → PAUSED / SUCCEEDED / FAILED / HALTED` with explicit allowed transitions | P0 |
| Command types | `CallLLM`, `CallTool`, `WriteMemory`, `RequestApproval` as typed commands returned by the pure core | P0 |
| Policy interface | `evaluate(state, action) → allow, deny, pause, downgrade` shared by all guards | P0 |
| Tracer \[#9\] | One OpenTelemetry span per transition, GenAI semantic conventions, exporters for OTLP, Langfuse, LangSmith, Arize | P0 |
| Event store | Append-only; SQLite for dev, PostgreSQL for production | P0 |
| State snapshots | Versioned snapshots so a run can resume after a schema change | P1 |

### runtime: the execution engine

| Feature | Requirement | Priority |
| --- | --- | --- |
| Async orchestrator \[#1\] | 100+ concurrent runs, concurrency limits, backpressure, cancellation, graceful error recovery, FastAPI to submit runs and read status | P0 |
| Tool router \[#3\] | Tool registry with JSON Schema from Pydantic models; validate every LLM output; use provider structured output when available | P0 |
| Self-healing retries \[#5\] | Up to 3 retries with the error trace fed back; classify errors as validation, tool, or transient | P0 |
| Context budget \[#2\] | Per-slot token allocation (system prompt, documents, tool schemas, history), trimming strategies, per-request usage report | P1 |
| Pause / resume | Persist state and resume on an external trigger without holding a thread | P1 |
| Memory \[#4\] | Interface first; then scratchpad, episodic compression, vector search with TTL eviction (pgvector) | P3 |

### guard: policies and isolation

| Feature | Requirement | Priority |
| --- | --- | --- |
| Cost kill-switch \[#12\] | Track loop depth and projected spend per run; downgrade to a cheaper model at a soft limit; hard stop at the cap (default $0.50, configurable) | P0 |
| Approval gate \[#7\] | Configurable list of high-stakes actions; Slack webhook approval; wait up to 3 days; audit who approved and when | P1 |
| Sandbox executor \[#11\] | Run generated code in E2B or gVisor with no internet egress and CPU, memory, and time limits | P1 |
| Sanitizer \[#10\] | Hook on inter-agent messages; flag suspicious instructions; tag message provenance | P2 |

### lab: evaluation and regression

| Feature | Requirement | Priority |
| --- | --- | --- |
| Record & replay | Replay a run from the event log with LLM and tool calls served from recordings | P2 |
| Trajectory diff | Compare a run against a golden DAG with tolerance: steps that may reorder versus steps that are required | P2 |
| CI gate \[#8\] | GitHub Action that replays recorded trajectories, blocks the PR on deviation, and posts a report on the PR; accepts any OpenTelemetry GenAI trace, not only Gumshoe runs | P2 |
| Shadow comparator \[#14\] | Mirror a share of traffic (default 5%) to a new topology silently; diff tool-call paths and cost | P3 |

### extras: built on top of the kernel

| Feature | Requirement | Priority |
| --- | --- | --- |
| Typed swarm \[#6\] | Agents hand off typed state objects, never raw text; LangGraph adapter | P3 |
| Browser fallback \[#13\] | When a REST call returns 404 or times out, finish the step with Playwright inside the sandbox | P3 |

## Key user flows

Four flows define the product. The first two are the Phase 1 demo.

**1. Run agents at scale.** The engineer defines tools as Pydantic models and an agent step function, then submits runs through the API or CLI. Gumshoe schedules them on the event loop, validates every tool call, retries failures with the error fed back, and streams each transition to the trace viewer.

**2. Stop a runaway agent.** A run starts looping on a tool that keeps failing. At the soft limit the kill-switch downgrades it to a cheaper model; at the hard cap it moves the run to `HALTED`. The trace shows every step, the spend curve, and the policy decision that stopped it.

**3. Approve a high-stakes action.** A run proposes an action on the approval list (for example, issuing a refund). The approval policy returns `pause`; the run is persisted as `PAUSED` and a Slack message asks for sign-off. When someone clicks Approve, even days later, the run resumes from its snapshot, and the audit log records who approved it.

**4. Catch a regression in CI.** A pull request changes a prompt. The CI gate replays recorded trajectories with LLM and tool calls served from recordings, compares each tool-call path against its golden DAG, and fails the check when a required step disappears. The PR comment shows which step changed and the cost difference.

**Phase 1 demo:** 100 concurrent runs, one runaway run halted by the kill-switch, and its full trace visible in Jaeger or Langfuse, with the detective mascot showing each run's state in the CLI.

## Tech stack

Python is the main language because the agent ecosystem (provider SDKs, OpenTelemetry GenAI, E2B, LangGraph) is Python-first.

| Area | Choice | Why |
| --- | --- | --- |
| Language & tooling | Python 3.13, uv, Ruff, pyright (strict) | Strict types matter because the event schema is the contract |
| Async & API | asyncio, FastAPI, Pydantic v2 | Pydantic models are also the source of tool JSON Schemas |
| Structured output | PydanticAI or Instructor; Outlines or xgrammar for self-hosted models | Proven retry-with-error patterns; constrained decoding only where the model is self-hosted |
| LLM providers | Thin in-house adapter over official SDKs; LiteLLM only if many providers are needed | The kill-switch needs exact token and cost data |
| Event store & state | PostgreSQL (SQLite in dev) | Append-only events, JSONB snapshots, transactions |
| Run queue | PostgreSQL with `SKIP LOCKED` or pgmq | One dependency; add Redis or Kafka only if Postgres becomes the bottleneck |
| Vector memory | pgvector | No separate vector database at first; TTL via an `expires_at` column |
| Tracing | OpenTelemetry SDK, GenAI semantic conventions | A shared standard, no lock-in to one backend |
| Trace backends (dev) | Jaeger or Grafana Tempo, self-hosted Langfuse | No custom trace UI needed early |
| Sandbox | E2B to start; gVisor (runsc) in Docker for self-hosting; Firecracker later | Fastest path first, hidden behind an executor interface |
| Browser fallback | Playwright | Standard and easy to run inside the sandbox |
| Testing | pytest, Hypothesis, syrupy, pytest-recording (VCR.py) | Property tests fit state machines; recorded LLM responses keep CI deterministic and free |
| CI gate | GitHub Action (Docker or composite) using the `lab` package | One diff engine, used locally and in CI |
| Packaging | uv workspace monorepo | Matches the module split |
| Deploy | Docker Compose first, Kubernetes later | No extra complexity before real load |
| Run viewer (optional) | SvelteKit, or HTMX with Jinja | Small and fast for replay and diff views |

**Build vs. buy for durable execution.** Gumshoe builds its own event-sourced execution on Postgres, because that is the product. The trade-off is owning the edge cases: a crash mid-step, idempotency, and versioning. DBOS is worth studying as a reference, since it is also library-based and Postgres-backed. Versions and the status of fast-moving libraries must be rechecked when the build starts.

## Roadmap

&#91;embedded content: Gumshoe roadmap · 4 phases, 3 gates\]

Each gate must pass before the next phase starts. Monetization starts once Phase 1 is complete, whether or not demand has shown up: the non-SaaS paths begin right away, and the hosted CI gate, the first paid product, is built as soon as the lab CI gate exists. No dates are committed yet; they depend on time available alongside a full-time job.

## Success metrics

Targets are starting hypotheses to revisit after the Phase 1 release.

| Area | Metric | Target |
| --- | --- | --- |
| Reliability | Concurrent runs on one worker without blocking | 100+ |
| Reliability | Runtime overhead per step, excluding LLM latency | Under 10 ms at p95 |
| Safety | Runaway runs stopped before the hard spend cap | 100% in the test suite |
| Correctness | Replays that reproduce the recorded tool-call path | 100% for recorded runs |
| Quality | Property-test coverage of state transitions | Every allowed and forbidden transition |
| Adoption | External contributors or issue reporters | 10+ |
| Demand signal | Requests for a hosted version or team features | 5+ |

The last row informs pricing and priorities; it is not a gate. Monetization starts once Phase 1 is complete.

## Monetization plan

The rule: what one developer uses is free and open source; what a team or organization needs is paid. Monetization starts once Phase 1 is complete, whether or not demand has shown up, and the first paid product is a hosted CI gate.

### Open-core split

| Module | Open source (Apache 2.0) | Paid (hosted or enterprise) |
| --- | --- | --- |
| core | Everything: event schema, state machine, tracer | Not sold; the contract must be open to be trusted |
| runtime | Orchestrator, tool router, self-healing, context budget | Managed runtime with autoscaling (no Postgres or workers to run) |
| guard (policies) | Kill-switch, sanitizer, policy interface | Hosted approval gate (always-on webhook, state held for days), central policies across teams, audit log, SSO |
| guard (sandbox) | Local gVisor executor | Managed sandbox, billed by usage |
| lab | Local replay and diff, CI gate for small repos | Hosted CI gate with trajectory history, regression dashboard, managed shadow comparator |
| extras | Everything | Not sold; drives adoption |

**License:** Apache 2.0 for all open-source code, which is the simplest choice for adoption and reputation. Paid parts are separate hosted services and stay closed. AGPL or a source-available license would slow adoption and is not worth it at this stage.

### First paid product: hosted CI gate

| Factor | Hosted CI gate (lab) | Hosted approval gate (guard) |
| --- | --- | --- |
| Buyer | AI engineers and teams with agents in production | Teams whose agents take high-stakes actions, plus compliance |
| Adoption | Bottom-up: one developer installs a GitHub Action | Top-down: needs security and management approval |
| Usage | Every pull request, recurring and easy to bill | Only on risky actions, low volume per customer |
| Operational load | Medium: batch jobs and trajectory storage | High: always-on endpoint, multi-day state, secure webhooks |
| Failure impact | A stuck or wrong gate, no real-world loss | Blocked actions, or worse, a forged approval |
| Sales cycle | Short | Long |
| Fit for a solo builder with a full-time job | Good | Heavy |

The CI gate wins: lighter to run, used on every PR, and it reaches buyers through GitHub and technical writing. It must ingest any OpenTelemetry GenAI trace, so it is not limited to Gumshoe users. The approval gate becomes a feature of the paid team plan. This flips if early customers are in regulated sectors (banking, state-owned enterprises, healthcare), where approvals and audit trails are worth a longer sale. Both can be sold in one team plan, so a buyer enters through either the CI gate or the approval gate. The build order stays CI gate first; the approval gate follows when the first regulated customer asks for it.

### Pricing models

| Product | Metric | Rationale |
| --- | --- | --- |
| Hosted CI gate | Per repo or seat, plus replays per month | Cost scales with PRs and trajectories |
| Team plan (dashboard, approvals, audit) | Per seat | Humans approve and review |
| Managed runtime | Per run or trace per month | Familiar observability-style pricing |
| Managed sandbox | Usage pass-through plus margin | Cost comes from infrastructure |

Price points are not set yet; they depend on competitor research before launch.

### Revenue paths that need no SaaS

1. **Consulting and implementation.** Usually the first real revenue: the open-source project is the showcase, and companies adopting agents hire help to install guardrails.
2. **Commercial support or license** for companies that need an SLA.
3. **Sponsorships, courses, and writing** on brendatama.dev: small income, but it attracts users, clients, and recruiters.

### Sequencing

1. Release Phase 1 as open source with a clear, replayable demo.
2. Publish 2–3 technical articles on specific problems: runaway cost and trajectory regressions.
3. Watch the signals: outside issues and questions, and requests for a hosted version. They shape pricing and priorities.
4. When Phase 1 is complete, start the monetization track whether or not demand has shown up: the non-SaaS paths first (consulting, support, sponsorships), then one paid product, the hosted CI gate, as soon as the lab CI gate exists. Not several at once.

## Competitive landscape

Each category is crowded on its own; Gumshoe's edge is the combination: one event log that powers guards, approvals, tracing, replay, and the CI gate together.

| Category | Examples | Gap Gumshoe fills |
| --- | --- | --- |
| Agent runtimes with policy and audit | [Lynx](https://pypi.org/project/lynx-agent/2.2.0/) (policy-gated tool calls, hash-chained audit), [Kronos Agent OS](https://pypi.org/project/kronos-agent-os/) (durable self-hosted agents) | Lynx v2 dropped durability and audit storage; neither ships trajectory replay or a CI gate |
| Agent SDKs | [Strands Agents](https://pypi.org/project/strands-agents/1.62.0/) (AWS), LangGraph, OpenAI Agents SDK | Frameworks own the loop logic; Gumshoe adds the production runtime and can wrap them via adapters |
| Observability | Langfuse, LangSmith, Arize, Helicone | Observe-only; Gumshoe acts (halts, pauses, blocks) and exports to them instead of competing |
| Record and replay | [agent-observability](https://pypi.org/project/agent-observability-trace/) (HTTP-level record/replay), agent-replay | Replay without policies, durable state, or golden-DAG gating |
| Gateways | LiteLLM, Portkey | Budgets at the request level, not run state or loop depth |
| Agent security | [Burrow](https://pypi.org/project/burrow-sdk/) (prompt-injection firewall, runtime monitoring) | Detection only; no execution engine or replay |
| Sandboxes | E2B, Modal | Execution environment only; Gumshoe uses them behind an executor interface |
| Durable execution | Temporal, DBOS, Restate, Inngest | General workflows; not agent-aware (no tool validation, spend caps, or trajectories) |

Examples without links are from general knowledge, not checked for this draft. Pricing and current features of all competitors need research before setting prices.

## Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Scope creep: 14 ideas for one builder | Everything half-done, nothing demoable | Ship Phase 1 only before starting Phase 2; extras stay deferred |
| Event schema or policy interface designed wrong | Replay, approvals, and shadow runs all need rewrites | Lock the schema first; property tests on every transition; version events from day one |
| Golden-DAG diffs too strict | False positives, so teams disable the gate | Tolerance rules for reorderable versus required steps, tested on real trajectories |
| Durable execution edge cases | Lost or duplicated actions after a crash | Idempotency keys on commands; crash tests in CI; study DBOS as a reference |
| Crowded market | Gumshoe is seen as one more tool in a known category | Lead with the combined story (guard + replay + CI gate on one log), not single features |
| Operating hosted services alongside a full-time job | Uptime, security, and support load | Start with the batch-style CI gate; defer the always-on approval service |
| Name collisions | The `gumshoe` package name on PyPI is already taken by an old Django project | Publish as `gumshoe-ai` or similar; confirm domain, GitHub organization, and trademark before launch |
| Intellectual property ownership | Conflict with the employment contract | Check the contract before release, especially for overlap with work projects |

## Open questions and next steps

**Open questions**

- Should Arcana (the planned sandbox orchestrator) become Gumshoe's sandbox executor, or stay a separate project?
- Which distribution name: `gumshoe-ai`, `gumshoe-agent`, or another?
- PydanticAI or Instructor for structured output?
- What is the default kill-switch cap per run, and is it set per agent or per tenant?

**Next steps**

- [ ] Draft the event schema and run state machine for `core`, with transitions and command types
- [ ] Define the policy interface and implement the kill-switch as its first policy
- [ ] Set up the uv workspace monorepo with the five packages
- [ ] Build the Phase 1 demo: 100 concurrent runs, one halted, traces in Jaeger or Langfuse
- [ ] Check name availability: PyPI, GitHub organization, domain, trademark
- [ ] Design the mascot's poses for each run state

## Sources

- [lynx-agent on PyPI](https://pypi.org/project/lynx-agent/2.2.0/)
- [kronos-agent-os on PyPI](https://pypi.org/project/kronos-agent-os/)
- [strands-agents on PyPI](https://pypi.org/project/strands-agents/1.62.0/)
- [agent-observability-trace on PyPI](https://pypi.org/project/agent-observability-trace/)
- [burrow-sdk on PyPI](https://pypi.org/project/burrow-sdk/)
- [gumshoe on PyPI](https://pypi.org/project/gumshoe) (existing package that holds the name)

## Related documents

- [Technical Design (Phase 1)](https://claude.ai/artifact/PTqii5ttUKJ7AbCkLTq4Ad): how the core, orchestrator, policies and tracing work
- [Phase 1 Implementation Plan](https://claude.ai/artifact/YXHUooeGy1BSXpzxN55oeK): milestones, acceptance criteria, demo script
- [Developer Guide](https://claude.ai/artifact/CD1e4Pxuz5KNzPgqpLu6p6): repository layout, tooling, conventions, CI, AGENTS.md
