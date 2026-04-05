# Project Bootstrap Standard

A new clone should be productive in one command or less. Environment setup is a project responsibility, not a developer task. If onboarding requires a wiki page with manual steps, the project has a bug.

---

## Principles

### Environment as Code

The ideal: a developer runs `cd` into the project and everything is ready. Nix + direnv achieves this — a `flake.nix` or `shell.nix` declares the full toolchain, and direnv loads it automatically on directory entry. No install instructions, no version matrix, no "works on my machine."

When Nix isn't viable (enterprise tooling, proprietary SDKs, team familiarity), the fallback is an idempotent setup script that detects what's missing and acts only when needed. The principle is the same either way: **declare, don't document.**

### Dependency Management

Lock files are non-negotiable. Every dependency must be pinned to an exact version and resolved deterministically. The setup entry point should handle installation automatically — and be smart enough to skip it when nothing has changed (e.g., by comparing lockfile hashes).

Language-agnostic, the rule is simple: a fresh clone with the correct runtime should produce an identical dependency tree every time, on every machine.

### Secrets & Environment Variables

Commit a `.env.example` with every required variable stubbed out and commented — each variable should explain what it's for and where to get a value. The setup script generates or copies `.env` if it's missing, pulling values from a vault, CI environment, or local defaults for development.

```bash
# .env.example
# PostgreSQL connection string. For local dev, use the docker compose instance.
DATABASE_URL=postgresql://localhost:5432/myapp_dev

# Stripe API key. Get a test key from https://dashboard.stripe.com/test/apikeys
STRIPE_SECRET_KEY=sk_test_...

# Log level: debug | info | warn | error
LOG_LEVEL=info
```

Secrets are never hardcoded and never committed. Where possible, local development and CI should source secrets the same way — same variable names, same structure, different backends. This keeps the gap between environments small and predictable.

### Local ↔ CI Parity

If it works locally, it works in CI. The CI pipeline should use the same setup entry point — not a parallel universe of bespoke scripts that happen to do similar things.

When local and CI diverge, bugs hide in the gap. Treat "it passes in CI but fails locally" (or vice versa) as a setup defect, not a flaky test.

### Task Running

Every project needs a uniform interface — a set of predictable verbs that work the same regardless of whether the project uses Python, Node, Rust, or anything else. This idea was formalized by GitHub in 2015 as **"Scripts to Rule Them All" (STRTA)**: a `script/` directory at the project root with consistently named entry points.

In practice, most projects implement this pattern through a task runner rather than individual scripts. A `justfile` or `Makefile` at the project root serves the same purpose: uniform targets with discoverable names. The STRTA naming convention (setup, test, server, etc.) is what matters — the mechanism is flexible.

**justfile is the recommended default.** It has cleaner syntax, built-in discoverability via `just --list`, and no `.PHONY` boilerplate. Use a Makefile when you need zero-dependency portability (open source, enterprise environments where you can't assume `just` is installed).

**The standard verbs:**

| Verb | Responsibility |
|---|---|
| `setup` | First-time setup (install deps, generate `.env`, seed data) |
| `dev` | Start the app in development mode |
| `test` | Run the test suite |
| `lint` | Run linter |
| `fmt` | Run formatter |
| `typecheck` | Run type checker |
| `check` | Run lint + typecheck + test — **mirrors CI exactly** |
| `ci` | What CI calls (setup + check in a clean environment) |

**The `default` recipe should print all available commands:**

```just
# Show all available commands
default:
    @just --list
```

The justfile *is* the project's API. If an action isn't in `just --list`, it doesn't exist. If a contributor needs to read a README to use the project, the justfile isn't complete enough.

**The two layers:**

There are two levels of task running in a well-structured project:

1. **Language-native runners** — `uv run`, `bun`, `cargo` — know how to execute things *within* their ecosystem. They handle dependency resolution, virtualenvs, and toolchain specifics. They're the implementation layer.

2. **Project-level runners** — `justfile` or `Makefile` — sit *above* the language tooling and provide a **uniform interface**. A `just test` might call `uv run pytest` in a Python project or `bun test` in a JS project. The developer doesn't need to know which.

The project-level runner is the one a new contributor interacts with. The language-native runner is what it delegates to.

**For simple projects** where the language tooling already provides good ergonomics (e.g., `cargo test` is already obvious), a dedicated task runner may be unnecessary overhead. The two-layer model earns its keep when the project has setup steps, env vars to manage, services to coordinate, or when CI needs a single entry point.

### Local Dev Data

A project should be usable without access to production or staging environments. This means providing a way to run with realistic data locally.

**Seed scripts:** A `just seed` recipe that populates a local database with a small but representative dataset — enough to exercise the main workflows and edge cases you care about. This takes 30 minutes to write once and saves hours of "let me point at staging."

**Recorded API responses:** For external APIs your project depends on, use response recording tools (`vcrpy` for Python, `polly.js` or `msw` for TypeScript) to capture real responses once, then replay them locally. You get realistic data without hitting live services, and your tests work offline.

**The standard:**

```just
# Set up local development environment with data
dev-setup:
    docker compose up -d
    just db-migrate
    just seed

# Seed local database with representative test data
seed:
    uv run python scripts/seed.py
```

Testing against production or staging APIs works until it doesn't — you can't test edge cases, you can't work offline, you can't run tests in CI without credentials, and you're one bad request away from corrupting real data.

---

## Code Organization

Good project structure makes the question "where does this go?" have an obvious answer. This section describes a sensible default — not a mandate, but a starting point that works for most projects.

### The Layering Principle

Separate your code into three layers based on what it does:

1. **Domain** — Pure business logic. Functions that take data in and return data out. No database calls, no HTTP requests, no file IO. This code is trivially testable — no mocks, no setup, just input → output.

2. **Services** — The IO boundary. Code that talks to databases, external APIs, the filesystem. Services implement interfaces and orchestrate domain logic with real-world side effects.

3. **API / Entrypoints** — The thin outer shell. HTTP handlers, CLI commands, queue consumers. These parse input, delegate to services, and format output. As little logic here as possible.

**The rule:** Domain imports nothing from Services or API. Services import from Domain. API imports from Services. Dependencies flow inward.

This isn't about ceremony — it's about testability and clarity. When your domain layer is pure, you can test your business logic with simple assertions. When your services are behind interfaces, you can swap real databases for test containers. When your API layer is thin, there's nothing interesting to test there except integration.

### Recommended Structures

**Python (uv):**

```
src/
  app/
    domain/           # Pure logic — no IO imports
      models.py       # Data types, value objects
      pricing.py      # Business rules as pure functions
      validation.py
    services/         # IO boundary — DB, HTTP, filesystem
      user_repo.py    # Data access (interface + implementation)
      email.py
      payment.py
    api/              # Thin HTTP layer — parse, delegate, respond
      routes.py
      middleware.py
      deps.py         # Dependency wiring (FastAPI Depends, etc.)
    config.py         # Typed config, validated at startup
    main.py           # Entrypoint — wiring only, no logic
tests/
  domain/             # Fast, no mocks needed
  services/           # Use testcontainers for real deps
  api/                # Integration tests against real HTTP
```

**TypeScript (Bun / Effect):**

```
src/
  domain/             # Pure logic — zero imports from services
    models.ts         # Types, branded types, discriminated unions
    pricing.ts        # Business rules as pure functions
    validation.ts
  services/           # Effect services — the IO boundary
    user-repo.ts      # Service interface + live implementation
    email-client.ts
    payment.ts
  api/                # HTTP handlers — thin, delegate to services
    routes.ts
    middleware.ts
  config.ts           # Effect Config — validated at startup
  main.ts             # Entrypoint — layer wiring only
test/
  domain/             # Fast, pure function tests
  services/           # Testcontainers or test implementations
  api/                # Integration tests
```

**Rust:**

```
src/
  domain/             # Pure types and logic
    mod.rs
    models.rs
    pricing.rs
  services/           # Trait definitions + implementations
    mod.rs
    user_repo.rs
    email.rs
  api/                # Handlers (Axum, Actix, etc.)
    mod.rs
    routes.rs
  config.rs
  main.rs             # Wiring
tests/
  domain/
  services/
  api/
```

### When the Domain Is Thin

Not every project has complex business logic. Data pipelines, CRUD APIs, and internal tools often have thin domains — and that's fine. Don't force a thick domain layer when the interesting complexity lives in orchestration, data flow, or infrastructure.

If your domain module is 50 lines of straightforward data transformations, that's the correct answer. The layering principle still helps because it tells you where things go, but don't manufacture abstraction to fill a folder.

### Explicit Public APIs

Each module should export a clear public interface. In Python, use `__init__.py` to control what's importable. In TypeScript, use barrel files (`index.ts`) or explicit exports. This prevents consumers from reaching into internal implementation details.

```python
# src/app/domain/__init__.py
from .models import User, Order
from .pricing import calculate_discount

# Everything else is private to the module
```

---

## Language Guides

### Python (uv)

**Why uv:** It replaces pip, pip-tools, pipx, venv, and pyenv with a single tool. It's fast (10–100x faster than pip), handles Python version management, and produces deterministic lockfiles out of the box.

**Project init:**

```bash
uv init my-project --python 3.13
cd my-project
```

This scaffolds `pyproject.toml`, `.python-version`, `.gitignore`, `README.md`, and a starter `main.py`.

**Recommended structure:**

```
my-project/
├── src/
│   └── my_project/
│       ├── __init__.py
│       ├── domain/
│       ├── services/
│       ├── api/
│       ├── config.py
│       └── main.py
├── tests/
│   ├── domain/
│   ├── services/
│   └── api/
├── scripts/
│   └── seed.py
├── .env.example
├── .python-version
├── pyproject.toml
├── uv.lock
├── justfile
└── README.md
```

**Key tools:**

| Concern | Tool |
|---|---|
| Package & project management | `uv` |
| Linting & formatting | `ruff` (via `uvx ruff check` / `uvx ruff format`) |
| Type checking | `mypy` or `pyright` |
| Testing | `pytest` (via `uv run pytest`) |

**Standard justfile:**

```just
default:
    @just --list

setup:
    uv sync
    cp -n .env.example .env || true

dev:
    uv run fastapi dev src/my_project/main.py

test:
    uv run pytest

lint:
    uvx ruff check .

fmt:
    uvx ruff format .

typecheck:
    uv run pyright

check: lint typecheck test
    @echo "✓ All checks passed"

seed:
    uv run python scripts/seed.py
```

**How it maps to the principles:**

- **Environment as code:** `uv run` auto-syncs the virtualenv against `uv.lock` before every command. Deleting `.venv` and running `uv run` recreates it instantly. No manual activate/deactivate cycle.
- **Dependency management:** `uv.lock` is the lockfile — commit it. `uv add <package>` updates both `pyproject.toml` and the lockfile. `uv sync --locked` in CI ensures the lockfile is up to date.
- **CI parity:** CI runs `just check`. Same command as local development.

**Nix + direnv integration:** Add a `.envrc` with `use flake` or `use nix` and include `uv` in the Nix shell. When Nix manages the Python version, set `python-preference = "only-system"` in `pyproject.toml` under `[tool.uv]` to prevent uv from downloading its own Python.

---

### Node / Bun

**The landscape:** Node.js remains the default runtime for most teams and existing projects. Bun is a compelling alternative for greenfield work — it's an all-in-one runtime, bundler, test runner, and package manager with significantly faster install and startup times. Both use the same `package.json` ecosystem.

**Recommended approach:** Use Bun for new projects where you control the stack. Use Node + pnpm for projects where ecosystem compatibility matters or the team is already established.

**Project init:**

```bash
# Bun
bun init my-project
cd my-project

# Node (pnpm)
mkdir my-project && cd my-project
pnpm init
```

**Recommended structure:**

```
my-project/
├── src/
│   ├── domain/
│   ├── services/
│   ├── api/
│   ├── config.ts
│   └── index.ts
├── test/
│   ├── domain/
│   ├── services/
│   └── api/
├── scripts/
│   └── seed.ts
├── .env.example
├── package.json
├── bun.lock / pnpm-lock.yaml
├── tsconfig.json
├── biome.json
├── justfile
└── README.md
```

**Key tools:**

| Concern | Tool |
|---|---|
| Runtime | Bun or Node |
| Package manager | `bun` (built-in) or `pnpm` |
| Linting & formatting | Biome (fast, single tool) or ESLint + Prettier |
| Testing | `bun test` (built-in) or Vitest |
| TypeScript | Built-in (Bun) or `tsc` |

**Standard justfile:**

```just
default:
    @just --list

setup:
    bun install
    cp -n .env.example .env || true

dev:
    bun run --watch src/index.ts

test:
    bun test

lint:
    bunx biome check .

fmt:
    bunx biome check --write .

typecheck:
    tsc --noEmit

check: lint typecheck test
    @echo "✓ All checks passed"

seed:
    bun run scripts/seed.ts
```

**How it maps to the principles:**

- **Environment as code:** Pin the runtime version via `.node-version` (Node) or via Nix/mise. With Bun, `bun install` is near-instant so zero-command setup via direnv is practical.
- **Dependency management:** Lockfiles (`bun.lock` or `pnpm-lock.yaml`) are committed. `pnpm` is preferred over `npm` for Node projects because of its content-addressable store — faster installs and no phantom dependencies.
- **CI parity:** `bun install --frozen-lockfile && just check` or `pnpm install --frozen-lockfile && just check`. The `--frozen-lockfile` flag ensures CI fails if the lockfile is out of date rather than silently updating it.

**Monorepo note:** For multi-package projects, both Bun and pnpm support workspaces natively. Define workspace members in `package.json` and keep a single lockfile at the root. For multi-language monorepos (e.g., TypeScript + Python), use Turborepo for cross-language task orchestration — it understands the package dependency graph and only rebuilds what changed.

---

### Rust

**Why Rust is easy here:** Cargo is one of the best-designed build tools in any ecosystem. It handles project scaffolding, dependency management, building, testing, benchmarking, and publishing — all with strong conventions and zero configuration for the common case. A project-level task runner is often unnecessary since Cargo's commands are already the uniform interface.

**Project init:**

```bash
cargo new my-project      # binary
cargo new my-lib --lib     # library
cd my-project
```

**Recommended structure (single crate):**

```
my-project/
├── src/
│   ├── domain/
│   │   ├── mod.rs
│   │   └── models.rs
│   ├── services/
│   │   ├── mod.rs
│   │   └── user_repo.rs
│   ├── api/
│   │   ├── mod.rs
│   │   └── routes.rs
│   ├── config.rs
│   ├── main.rs
│   └── lib.rs
├── tests/
│   └── integration.rs
├── benches/
│   └── bench.rs
├── .env.example
├── Cargo.toml
├── Cargo.lock
└── rust-toolchain.toml
```

**For larger projects, use a workspace:**

```
my-workspace/
├── Cargo.toml            # [workspace] members
├── Cargo.lock            # shared across all crates
├── crates/
│   ├── core/             # domain types and logic
│   ├── cli/
│   └── server/
└── rust-toolchain.toml
```

**Key tools:**

| Concern | Tool |
|---|---|
| Build & dependency management | `cargo` |
| Linting | `clippy` (`cargo clippy`) |
| Formatting | `rustfmt` (`cargo fmt`) |
| Testing | Built-in (`cargo test`) |
| Benchmarking | Built-in (`cargo bench`) or `criterion` |
| Audit | `cargo-audit` for vulnerability scanning |

**How it maps to the principles:**

- **Environment as code:** Pin the Rust toolchain via `rust-toolchain.toml` at the project root. Cargo respects this automatically — anyone who clones the project gets the exact compiler version. Combine with Nix for full reproducibility, or rely on `rustup` as the fallback.
- **Dependency management:** `Cargo.lock` is the lockfile. Commit it for binaries and applications (deterministic builds). For libraries, the convention is to not commit it — consumers resolve their own dependency tree.
- **CI parity:** `cargo test` and `cargo clippy` locally are the same commands CI runs. The `rust-toolchain.toml` ensures the CI runner uses the identical compiler version.

**`rust-toolchain.toml` example:**

```toml
[toolchain]
channel = "1.84"
components = ["rustfmt", "clippy"]
```

---

## Checklist

- [ ] Can a new clone go from zero to running in one command?
- [ ] Is the full toolchain declared in code (Nix flake, setup script, or equivalent)?
- [ ] Is there a justfile with standard verbs (setup, dev, test, lint, fmt, check)?
- [ ] Does `just check` mirror CI exactly?
- [ ] Do justfile recipes delegate to language-native tooling (not duplicate it)?
- [ ] Are all dependencies pinned with a lock file?
- [ ] Does the setup step skip work when nothing has changed?
- [ ] Is `.env.example` committed with commented explanations for every variable?
- [ ] Are secrets sourced from a vault or environment — never hardcoded?
- [ ] Does CI use the same entry points as local development?
- [ ] Is the runtime/compiler version pinned (`.python-version`, `rust-toolchain.toml`, `.node-version`)?
- [ ] Can two developers on different machines get identical results?
- [ ] Is there seed data or fixture setup for local development (`just seed`)?
- [ ] Is the code organized with a clear separation of domain, services, and API?
- [ ] Does each module export an explicit public API?
