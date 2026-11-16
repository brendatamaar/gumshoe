# Gumshoe — Developer Guide

Oct 31, 2026 · @brendatama · Status: Draft v0.2

## Start here

This guide is for working inside the Gumshoe repository day to day: setting it up, writing code, testing, running CI and releasing. Read the other documents first.

| Document | It answers |
| --- | --- |
| [Product Requirements Document](https://claude.ai/artifact/YYfatFCfAKu8DYe4LXMcAU) | What Gumshoe is, why, for whom, and the roadmap |
| [Technical Design (Phase 1)](https://claude.ai/artifact/PTqii5ttUKJ7AbCkLTq4Ad) | How Phase 1 works: state machine, events, policies, orchestrator |
| [Phase 1 Implementation Plan](https://claude.ai/artifact/YXHUooeGy1BSXpzxN55oeK) | What to build, in what order, and when each piece is done |
| This guide | How to set up, code, test and release |

**Suggested reading order.** The PRD sections on the overview, design principles and module requirements. Then the Technical Design in full. Then milestones M0 and M1 in the plan. Then the rest of this guide while setting up the repository.

## Repository layout

One monorepo, one package per module from the PRD. The import name `gumshoe` is assumed here; the distribution names are decided in milestone M0, because `gumshoe` on PyPI is already taken.

```text
gumshoe/
├── pyproject.toml          # uv workspace root and shared tool settings
├── .functional-effect.json # settings for the Model Functional Effect plugin
├── uv.lock
├── Makefile
├── compose.yaml            # Postgres and Jaeger for local development
├── AGENTS.md               # rules for coding agents
├── README.md
├── LICENSE                 # Apache 2.0
├── .env.example
├── .github/workflows/ci.yml
├── packages/
│   ├── core/               # pure: events, state machine, step, ports
│   │   ├── pyproject.toml
│   │   ├── ruff.toml       # banned imports, print and naive datetimes
│   │   ├── src/gumshoe/core/
│   │   └── tests/
│   ├── runtime/            # shell: orchestrator, router, stores, adapters, API, CLI, OTel
│   │   ├── pyproject.toml
│   │   ├── src/gumshoe/runtime/
│   │   └── tests/
│   ├── guard/              # pure policies: kill-switch now; approval, sandbox, sanitizer later
│   │   ├── pyproject.toml
│   │   ├── ruff.toml       # same banned imports as core
│   │   ├── src/gumshoe/guard/
│   │   └── tests/
│   ├── lab/                # skeleton in Phase 1: replay, diff and CI gate come later
│   └── extras/             # skeleton in Phase 1: swarm and browser fallback come later
├── examples/               # runaway agent, a simple tool, a custom agent
└── docs/                   # exported copies of the PRD, design and plan
```

**Dependency direction**

| Package | May depend on |
| --- | --- |
| `core` | The standard library only. Never on the other packages |
| `guard` | `core` |
| `runtime` | `core`. It uses `guard` only from the CLI and the demo |
| `lab`, `extras` | `core` and `runtime` |

**Namespace.** All packages share the top-level `gumshoe` namespace across separate distributions. Do not put an `__init__.py` in `src/gumshoe/`. Put one only inside `core/`, `runtime/`, and so on, so Python treats `gumshoe` as an implicit namespace package.

## Tooling and configuration

These are starting configurations. Check them against the current documentation of uv, Ruff and pyright when you set up, since options and defaults change.

**Root `pyproject.toml`**

```toml
[project]
name = "gumshoe-workspace"
version = "0.0.0"
requires-python = ">=3.13"
dependencies = ["gumshoe-core", "gumshoe-runtime", "gumshoe-guard"]

[tool.uv.workspace]
members = ["packages/*"]

[tool.uv.sources]
gumshoe-core = { workspace = true }
gumshoe-runtime = { workspace = true }
gumshoe-guard = { workspace = true }

[dependency-groups]
dev = ["pytest", "pytest-asyncio", "hypothesis", "pytest-recording", "syrupy", "ruff", "pyright"]

[tool.ruff]
line-length = 100
target-version = "py313"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "ASYNC", "S", "PT", "RUF"]

[tool.pyright]
typeCheckingMode = "strict"
pythonVersion = "3.13"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["packages"]
markers = [
  "slow: load and crash tests",
  "postgres: needs the Compose Postgres",
]
```

**`packages/core/pyproject.toml`**

```toml
[project]
name = "gumshoe-core"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = []   # the standard library only; Pydantic lives in runtime

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/gumshoe"]
```

The distribution names above are placeholders until milestone M0 settles the final names.

**Suggested runtime dependencies per package**

| Package | Dependencies |
| --- | --- |
| `core` | `none (standard library)` |
| `guard` | `gumshoe-core` |
| `runtime` | `gumshoe-core`, pydantic, `fastapi`, `uvicorn`, an async Postgres driver, an async SQLite driver, `opentelemetry-sdk` and an OTLP exporter, the first provider's SDK, a CLI library |
| `lab`, `extras` | `gumshoe-core`, `gumshoe-runtime` (skeletons in Phase 1) |

**Makefile**

```make
.PHONY: sync lint types test check up down
sync: ; uv sync --all-packages
lint: ; uv run ruff check . && uv run ruff format --check .
types: ; uv run pyright
test: ; uv run pytest -m "not slow"
check: lint types test
up: ; docker compose up -d
down: ; docker compose down
```

**Pre-commit.** Run Ruff's check and format hooks (see the `ruff-pre-commit` repository for the current hook ids) and pin the revision when you add it.

## Functional core enforcement

The core follows Model Functional Effect with the `distributed` profile (Technical Design, Functional core and shell). Two layers enforce it. The plugin's hook checks every edit Claude makes to a core file. Ruff and pyright check everything else in CI. `core` and `guard` are both pure, because policies run inside `step`, so both are listed as core paths.

**`.functional-effect.json`** at the repository root:

```json
{
  "core": ["packages/core/src/gumshoe/core/**", "packages/guard/src/gumshoe/guard/**"],
  "shell": ["packages/runtime/src/gumshoe/runtime/**"],
  "profile": "distributed",
  "verify": { "command": "make check", "timeout": 600 },
  "rules": { "py": { "allowImports": [], "forbidImports": [], "forbidCalls": [] } }
}
```

**`packages/core/ruff.toml`** (copy it to `packages/guard/ruff.toml`). Ruff reads the nearest config, so this applies only to the pure packages:

```toml
extend = "../../pyproject.toml"

[lint]
extend-select = ["TID251", "T20", "DTZ", "S311", "BLE", "ASYNC"]

[lint.flake8-tidy-imports.banned-api]
"os".msg = "Core is pure: filesystem, env and process access belong in the shell."
"sys".msg = "Core is pure: process and stdio access belong in the shell."
"subprocess".msg = "Core is pure: spawn processes in the shell."
"socket".msg = "Core is pure: network access belongs in the shell."
"asyncio".msg = "Core is pure: scheduling is a ScheduleInput command."
"threading".msg = "Core is pure: concurrency belongs in the shell."
"time".msg = "Core is pure: time is an Input field."
"random".msg = "Core is pure: randomness is an Input field."
"logging".msg = "Core is pure: logging is output; return a Command."
"httpx".msg = "Core is pure: network access belongs in the shell."
"pydantic".msg = "Core is plain data: wire models belong in runtime."
"datetime.datetime.now".msg = "Core is pure: time is an Input field."
"datetime.datetime.utcnow".msg = "Core is pure: time is an Input field."
"uuid.uuid4".msg = "Core is pure: generated ids are Input fields."
```

**Pyright.** Make `match` exhaustive, so a new variant that reaches `assert_never` fails the type check. Replace the `[tool.pyright]` table in the root `pyproject.toml` with:

```toml
[tool.pyright]
typeCheckingMode = "strict"
reportMatchNotExhaustive = "error"
pythonVersion = "3.13"
```

Check the option names against your Ruff and pyright versions. Then prove the setup works: add a throwaway `import os` to a core file. Ruff must report `TID251`, and the plugin's hook must block the edit. Remove the file afterwards.

**Review.** Before merging a change to a core file, run the plugin's review skill. It uses the `fe-reviewer` agent and checks the pure core, the sum types, the legal transitions, the boundary parsing and the property tests.

## Coding conventions

The first five rules are the ones that keep the design intact. Enforce them with automated checks, not review alone.

1. **`core` is pure.** No imports of I/O libraries: no `asyncio`, sockets, HTTP clients, database drivers, `os.environ`, `time`, `random` or `uuid4`, and no Pydantic. States, inputs, commands and events are frozen dataclasses with smart constructors. The plugin's hook and the nested `ruff.toml` fail on a violation.
2. **Everything is immutable.** All models are frozen. No module-level state, no singletons.
3. **Time and ids are data.** Every input carries `now` and an `input_id`, read or generated once at the edge. A delay is a `ScheduleInput` command, never a sleep. Deterministic ids such as `uuid5(...)` are fine in the core.
4. **Policies are pure.** A policy never keeps counters. Counters live in `RunState`.
5. **Events are append-only history.** Never edit a shipped payload. Add a new `schema_version` and an upcaster.
6. **Expected failures are values.** A failed validation at the boundary returns a typed result. A `(state, input)` pair with no transition returns the same state and no commands. Exceptions are for bugs and infrastructure failures.
7. **Adapters live at the edge.** Provider SDKs and database drivers are in `runtime` and implement ports defined in `core`.
8. **Never block the event loop.** No synchronous I/O inside `async def`. Wrap synchronous libraries with `asyncio.to_thread`.
9. **Strict typing.** pyright strict, no `Any` in public APIs, and a reason in a comment next to every `type: ignore`.
10. **Naming.** Events in the past tense (`ToolExecuted`), commands in the imperative (`CallTool`).

Public functions get a one-line docstring. Comments explain why, not what.

## Testing guide

The strategy and the invariants are in the Technical Design. This section is about how to work with them.

- **Layout.** Tests sit next to their package under `packages/<name>/tests`. Mark slow tests `slow` and tests that need Postgres `postgres`. The default run is `make test`, which skips `slow`.
- **State-machine tests.** Use Hypothesis `RuleBasedStateMachine` for the run state machine: rules generate inputs, and invariants check `seq`, terminal states, and `fold(events)` against the live state. Use a larger example count in CI than locally by registering a profile.
- **Fakes.** Pure fakes (`FakeClock`, the sequential `IdGenerator`) live in `core`'s testing module. `FakeLLM` and `FakeTool` live in `runtime`'s. Build the runaway fake agent once and reuse it in the kill-switch tests and the demo.
- **Contract tests.** Write the `EventStore` suite once and parametrize it over SQLite and Postgres. Postgres tests use the Compose service (or a container started by a fixture).
- **Recorded responses.** Provider adapter tests replay recordings made with `pytest-recording`. Scrub authorization headers from every recording. A recording contains the prompts you sent, so review it before committing it.
- **Golden snapshots.** Use `syrupy` to keep one JSON snapshot per event type. A diff in a snapshot means the schema changed, so bump `schema_version` on purpose or fix the change.
- **Load and crash tests.** Mark them `slow` and run them locally or on a schedule, not on every push. The crash test starts a worker as a subprocess, kills it mid-run, and checks that a second worker finishes the run.
- **Determinism.** Tests never read the real clock or real randomness. A flaky test is a bug in the design, not noise. Every recorded run is also a regression test: running \`step\` over its inputs must reproduce its final state. Store tests cover duplicate inputs, stale lease epochs and outbox redelivery.

## Local environment

Docker Compose provides Postgres and a trace backend. Pin the image tags to specific versions when you create the file, instead of `latest`.

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: gumshoe
      POSTGRES_PASSWORD: gumshoe
      POSTGRES_DB: gumshoe
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    ports:
      - "16686:16686"   # UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP

volumes:
  pgdata: {}
```

**Setup.** Copy `.env.example` to `.env`, run `make up`, then `uv sync --all-packages`. The Jaeger UI is at `http://localhost:16686`. Never commit `.env`.

**Environment variables (proposed names)**

| Variable | Purpose |
| --- | --- |
| `GUMSHOE_DATABASE_URL` | Postgres connection string (SQLite path for development) |
| `GUMSHOE_OTLP_ENDPOINT` | Where to export traces, for example `http://localhost:4318` |
| `GUMSHOE_API_KEY` | Bearer token for the HTTP API |
| `GUMSHOE_MAX_CONCURRENT_RUNS` | Concurrency limit per worker (default 100) |
| `GUMSHOE_CAPTURE_MODE` | `full` or `metadata_only` |
| The provider's own key variable | Credentials for the first LLM provider |

## Continuous integration

One workflow runs the same checks as `make check`, with a Postgres service for the contract tests.

```yaml
name: ci
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: gumshoe
          POSTGRES_PASSWORD: gumshoe
          POSTGRES_DB: gumshoe
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U gumshoe"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      GUMSHOE_DATABASE_URL: postgresql://gumshoe:gumshoe@localhost:5432/gumshoe
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5   # pin to the current major version
      - run: uv sync --all-packages
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run pyright
      - run: uv run pytest -m "not slow"
```

Check the action versions against their current releases when you create the file.

**Rules.** Make this workflow a required check on the main branch. Include the functional-core lint (the nested ruff.toml) for `core` in the test run. Run the `slow` tests (load and crash) on a schedule or manually, not on every push. Add a Python version matrix only if you decide to support more than one version.

## Working with coding agents

The design depends on a few rules that are easy to break by accident, so write them down where a coding agent reads them. Put this file at the repository root as `AGENTS.md`. Claude Code reads `CLAUDE.md`; keep one source of truth by making `CLAUDE.md` a symlink to `AGENTS.md` or a short file that points to it.

```markdown
# AGENTS.md

Gumshoe is an open-source Python runtime that runs LLM agents in production with guardrails and replay. Before changing behavior, read the PRD, the Technical Design and the Phase 1 plan in `docs/`.

## Architecture rules (never break these)
- `packages/core` and `packages/guard` are pure: no I/O, no clock, no randomness, no environment access, no async, no globals, no Pydantic. Time and ids arrive as data in the input. A delay is a `ScheduleInput` command.
- `step(state, input)` returns a new state and commands. A pair with no transition returns the same state and no commands. Results carry `call_id`, `step_id` and `attempt`, and a stale result is ignored.
- `core` depends on the standard library only. It never imports from `runtime`, `guard`, `lab` or `extras`.
- States, inputs, commands, events and verdicts are frozen dataclasses. Unions end in `assert_never`. Policies are pure functions and hold no counters.
- Shipped event payloads are never edited. A breaking change means a new `schema_version` plus an upcaster.
- Everything with side effects (database, LLM providers, tools, HTTP, OpenTelemetry export) lives in `runtime`, behind ports defined in `core`. Pydantic wire models live there too and map onto core types.
- Persistence is one transaction: inbox, events, projection, outbox. A worker drains the outbox.
- Never block the event loop. Wrap synchronous libraries with `asyncio.to_thread`.

## Commands
- `uv sync --all-packages`  install everything
- `make check`  Ruff, pyright and pytest
- `make up` / `make down`  start and stop Postgres and Jaeger

## Working style
- Work on one milestone of the Phase 1 plan at a time.
- Write the failing test first. For logic in `core`, prefer a Hypothesis property over examples.
- Keep diffs small and say which design section a change implements.
- Run the functional-effect review before finishing a change to a core file.

## Definition of done
Typed (pyright strict), Ruff clean, functional-core check clean, tests added, design doc updated if behavior changed, CI green.

## Do not
- Do not add a dependency to `core`.
- Do not weaken typing (`Any`, `type: ignore`) to make checks pass.
- Do not commit secrets, or recorded provider responses that contain them.
- Do not start Phase 2 features (approval gate, sandbox, memory) before Phase 1 passes its exit criteria.
```

**Task prompt template.** Give the agent one task at a time, with the design section it implements:

```text
Implement <task> from milestone <M#> of the Phase 1 plan.
Follow the Technical Design section <section name>.
Write the failing tests first (property tests for core logic), then the code.
Change nothing outside <package>. Run make check before finishing.
```

## Release process

- **Versioning.** Semantic versioning. Stay on 0.x until the event schema has been stable through at least one release, and say so in the README.
- **Changelog.** One entry per release, grouped by package. Any event schema change gets its own line.
- **Publishing.** Use PyPI Trusted Publishing from GitHub Actions, so no long-lived token is stored. Do a dry run on TestPyPI first. Publish in dependency order: `core`, then `guard`, then `runtime`.
- **Names.** Confirm the distribution and import names, the GitHub repository and the domain before the first publish (milestone M0). Consider publishing a minimal first version early to hold the names.
- **Tags.** A version tag triggers the release workflow. Create a GitHub release with the changelog entry.
- **Dependencies.** Keep the lock file committed, and enable automated dependency update pull requests.
- **Security.** Include a security policy with a contact address, and decide how vulnerability reports are handled before publishing.
- **Before the first release.** Complete the release checklist in the Phase 1 plan, including the employment-contract check from the PRD risks.

## First-day checklist

- [ ] Read the PRD overview and design principles, then the Technical Design
- [ ] Settle the names (distribution, import, GitHub, domain) and check trademarks
- [ ] Create the repository with the Apache 2.0 license and a README stub
- [ ] Set up the uv workspace with the five package skeletons
- [ ] Add the Ruff, pyright and pytest configuration, the Makefile, and .functional-effect.json with the plugin installed
- [ ] Add `compose.yaml` and `.env.example`, then run `make up`
- [ ] Add the CI workflow and confirm it passes on the empty skeleton
- [ ] Add the nested ruff.toml for `core` and prove that Ruff and the plugin's hook both fail on a deliberate I/O import, Pydantic import or datetime.now
- [ ] Commit `AGENTS.md`
- [ ] Write the first failing test: the transition table for the run state machine
- [ ] Open a branch for milestone M1
