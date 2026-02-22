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

Commit a `.env.example` with every required variable stubbed out. The setup script generates or copies `.env` if it's missing, pulling values from a vault, CI environment, or local defaults for development.

Secrets are never hardcoded and never committed. Where possible, local development and CI should source secrets the same way — same variable names, same structure, different backends. This keeps the gap between environments small and predictable.

### Local ↔ CI Parity

If it works locally, it works in CI. The CI pipeline should use the same setup entry point — not a parallel universe of bespoke scripts that happen to do similar things.

When local and CI diverge, bugs hide in the gap. Treat "it passes in CI but fails locally" (or vice versa) as a setup defect, not a flaky test.

### Task Running

Every project needs a uniform interface — a set of predictable verbs that work the same regardless of whether the project uses Python, Node, Rust, or anything else. This idea was formalized by GitHub in 2015 as **"Scripts to Rule Them All" (STRTA)**: a `script/` directory at the project root with consistently named entry points.

**The standard STRTA scripts:**

| Script | Responsibility |
|---|---|
| `script/bootstrap` | Install/update all dependencies |
| `script/setup` | First-time setup (calls bootstrap, then initializes state) |
| `script/update` | Update after a pull (calls bootstrap, then migrates etc.) |
| `script/server` | Start the app |
| `script/test` | Run the test suite |
| `script/cibuild` | What CI calls (setup + test in a clean environment) |
| `script/console` | Open an interactive REPL |

The scripts are composable — `setup` calls `bootstrap`, `cibuild` calls `test`. Each does one unit of work.

**In practice, most projects implement this pattern through a task runner rather than individual scripts.** A `Makefile` or `justfile` at the project root serves the same purpose: uniform targets with discoverable names. The STRTA naming convention (setup, test, server, etc.) is what matters — the mechanism is flexible.

**The two layers:**

There are two levels of task running in a well-structured project, and it's worth understanding how they relate:

1. **Language-native runners** — `uv run`, `npm scripts`, `cargo` — know how to execute things *within* their ecosystem. They handle dependency resolution, virtualenvs, and toolchain specifics. They're the implementation layer.

2. **Project-level runners** — `Makefile`, `justfile`, or `script/` — sit *above* the language tooling and provide a **uniform interface**. A `make test` might call `uv run pytest` in a Python project or `bun test` in a JS project. The developer doesn't need to know which.

The project-level runner is the one a new contributor interacts with. The language-native runner is what it delegates to. Example:

```makefile
# Makefile
.PHONY: setup test server lint

setup:
	uv sync
	cp -n .env.example .env || true

test:
	uv run pytest

server:
	uv run uvicorn app:main --reload

lint:
	uvx ruff check .
	uvx ruff format --check .
```

```just
# justfile
setup:
    uv sync
    cp -n .env.example .env || true

test:
    uv run pytest

server:
    uv run uvicorn app:main --reload

lint:
    uvx ruff check .
    uvx ruff format --check .
```

**For simple projects** where the language tooling already provides good ergonomics (e.g., `cargo test` is already obvious), a dedicated task runner may be unnecessary overhead. The two-layer model earns its keep when the project has setup steps, env vars to manage, services to coordinate, or when CI needs a single entry point.

**Makefile vs justfile:**

| | Makefile | justfile |
|---|---|---|
| Availability | Pre-installed on virtually all Unix systems | Requires installation (`cargo install just`, or via package manager) |
| Syntax | Tab-sensitive, cryptic variable assignment, `.PHONY` boilerplate | Spaces or tabs, clean variable syntax, no `.PHONY` needed |
| Discovery | `cat Makefile` or `make help` (if you wrote one) | `just --list` shows all recipes with descriptions |
| Industry adoption | De facto standard, universally recognized | Growing fast, especially in Rust/modern tooling communities |
| Best for | Teams, open source, enterprise (zero-dependency) | Solo/greenfield, projects already using Nix (just is usually available) |

Either works. Makefile is the safer default for shared projects; justfile is nicer to write and maintain. Pick one per project and be consistent.

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
│       └── main.py
├── tests/
│   └── test_main.py
├── .env.example
├── .python-version
├── pyproject.toml
├── uv.lock
├── Makefile / justfile
└── README.md
```

**Key tools:**

| Concern | Tool |
|---|---|
| Package & project management | `uv` |
| Linting & formatting | `ruff` (via `uvx ruff check` / `uvx ruff format`) |
| Type checking | `mypy` or `pyright` |
| Testing | `pytest` (via `uv run pytest`) |

**How it maps to the principles:**

- **Environment as code:** `uv run` auto-syncs the virtualenv against `uv.lock` before every command. Deleting `.venv` and running `uv run` recreates it instantly. No manual activate/deactivate cycle.
- **Dependency management:** `uv.lock` is the lockfile — commit it. `uv add <package>` updates both `pyproject.toml` and the lockfile. `uv sync` restores the exact environment.
- **CI parity:** CI runs `uv sync && uv run pytest`. Same commands as local development.

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
│   └── index.ts
├── tests/
│   └── index.test.ts
├── .env.example
├── package.json
├── bun.lock / pnpm-lock.yaml
├── tsconfig.json
├── biome.json
├── Makefile / justfile
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

**How it maps to the principles:**

- **Environment as code:** Pin the runtime version via `.node-version` (Node) or via Nix/mise. With Bun, `bun install` is near-instant so zero-command setup via direnv is practical — just `bun install && bun run dev` in the `.envrc`.
- **Dependency management:** Lockfiles (`bun.lock` or `pnpm-lock.yaml`) are committed. `pnpm` is preferred over `npm` for Node projects because of its content-addressable store — faster installs and no phantom dependencies.
- **CI parity:** `bun install --frozen-lockfile && bun test` or `pnpm install --frozen-lockfile && pnpm test`. The `--frozen-lockfile` flag ensures CI fails if the lockfile is out of date rather than silently updating it.

**Monorepo note:** For multi-package projects, both Bun and pnpm support workspaces natively. Define workspace members in `package.json` and keep a single lockfile at the root.

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
│   ├── core/
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── cli/
│   │   ├── Cargo.toml
│   │   └── src/
│   └── server/
│       ├── Cargo.toml
│       └── src/
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
- [ ] Is there a project-level task runner with standard verbs (setup, test, server, lint)?
- [ ] Do task runner targets delegate to language-native tooling (not duplicate it)?
- [ ] Are all dependencies pinned with a lock file?
- [ ] Does the setup step skip work when nothing has changed?
- [ ] Is `.env.example` committed with all required variables?
- [ ] Are secrets sourced from a vault or environment — never hardcoded?
- [ ] Does CI use the same entry points as local development?
- [ ] Is the runtime/compiler version pinned (`.python-version`, `rust-toolchain.toml`, `.node-version`)?
- [ ] Can two developers on different machines get identical results?
