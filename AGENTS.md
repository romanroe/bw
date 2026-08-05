# Repository Guidelines

## Project Overview

`bw` is a Bash wrapper around [`bwrap`](https://github.com/containers/bubblewrap) (bubblewrap) that runs coding-agent CLIs (opencode, omp, codex, claude, junie) inside an unprivileged Linux sandbox. It default-denies access to all host data, exposes only an explicit allowlist of paths, hides secrets, and controls environment and network. The entire project is three shell scripts — there is no application source, build system, or package.

## Architecture & Data Flow

Layering: `bwoc`/`bwomp` → `bw` → `bwrap`.

`scripts/bw` is a linear, fail-closed pipeline (`scripts/bw:1` sets `set -euo pipefail`):

1. **Parse options** (`scripts/bw:122-154`) — `--workdir`, `--mask`, `--offline`, `--allow-env-file`, `--help`; the rest is `command [args...]`.
2. **Validate** (`scripts/bw:156-354`) — canonicalize every path; refuse `workdir == HOME` or `workdir` inside `sandbox_home`; reject masks that would hide the workdir/sandbox home; require every read-only/writable HOME path and file to exist.
3. **Assemble `bwrap` args** (`scripts/bw:362-515`) — private `/proc`, `/dev`, tmpfs `/tmp`; `--clearenv`; read-only system + tool mounts; writable allowlist + workdir; masks; `.env` hiding; Wayland socket; network toggle.
4. **Resolve command** (`scripts/bw:517-550`) — alias vs. host binary vs. path; augment `PATH`.
5. **Exec** `bwrap "${BWRAP_ARGS[@]}" -- "$@"` (`scripts/bw:552`).

**Security model:** `SANDBOX_HOME` (`$HOME/sandbox_home`, mode `0700`) is bind-mounted over `$HOME` as a persistent fallback (`scripts/bw:376`); specific HOME mounts overlay it. `--unshare-all` drops all namespaces; `--share-net` opts networking back in only when not `--offline` (`scripts/bw:456-460`). Masks and default `.env` hiding overlay an empty read-only dir/file (`EMPTY_DENY_DIR`/`EMPTY_DENY_FILE`, `scripts/bw:356-360`), cleaned via `trap ... EXIT`.

## Key Directories

- `scripts/` — all executables. `bw` (main, 553 lines), `bwoc`, `bwomp`.
- `.idea/` — PyCharm/IntelliJ metadata, gitignored. Note: declares a `PYTHON_MODULE` (`.idea/bw.iml`) despite being a pure Bash project — ignore this mismatch.

## Development Commands

There is no build/test/lint tooling in the repo. Work directly on the scripts.

```bash
# Run a sandboxed agent CLI (network on by default)
scripts/bw opencode
scripts/bwoc            # == bw opencode
scripts/bwomp           # == bw omp

# Common flags
scripts/bw --offline bash                       # no network
scripts/bw --mask "$HOME/Nextcloud/Privat" -- opencode
scripts/bw --allow-env-file -- opencode         # expose .env files
scripts/bw --workdir /path/to/proj -- omp       # mount a specific RW workdir
scripts/bw --help
```

Recommended (not present) checks for changes: `shellcheck scripts/bw` and `shfmt -d scripts/bw`.

## Code Conventions & Common Patterns

- **Bash strict mode**: `set -euo pipefail` at the top of every non-trivial script.
- **Helpers**: `require_cmd` (dependency preflight), `canon_path` (expands `~`, `readlink -m`), `is_subpath_or_same` (path containment) — reuse these; do not reimplement path logic inline.
- **Fail closed**: validate before acting; on any invalid/conflicting config, print a multi-line diagnostic to `stderr` and `exit 2`. Missing command → `exit 127`. Follow this exact convention for new checks.
- **Config as arrays**: allowlists are top-of-file arrays — `SYSTEM_RO_PATHS`, `READONLY_HOME_PATHS`, `READONLY_HOME_FILES`, `WRITABLE_HOME_PATHS`, `WRITABLE_HOME_FILES`, `MASKED_PATHS`. Add/remove entries here; every existing entry is asserted to exist at runtime.
- **Quoting**: quote all expansions; use `"$@"`, `IFS=`/`read -r`, and null-delimited `find ... -print0` loops.
- **Comments** explain *why* (security intent), not *what*.

## Important Files

- `scripts/bw` — the entire program; entry point and all logic.
- `scripts/bwoc`, `scripts/bwomp` — thin `exec bw <subcommand> "$@"` launchers; no logic.
- `LICENSE` — MIT, © 2026 Roman Roelofsen.
- `.gitignore` — ignores only `.idea`.

## Runtime/Tooling Preferences

- **OS**: Linux only (requires user namespaces / bubblewrap).
- **Required host commands** (preflighted, `scripts/bw:52-55`): `bwrap`, `readlink`, `mktemp`, `mkdir`.
- **Node runtime**: a mise-managed Node at `~/.local/share/mise/installs/node/latest/bin/node` is mandatory (`scripts/bw:253-257`; exits 2 if missing). It is prepended to the sandbox `PATH`.
- **Read-only tool mounts** assumed present: `~/.pyenv`, `~/.local/bin`, `~/.local/share/{mise,pipx}`, `~/.config/mise`, `~/.config/pypoetry/config.toml`.
- **Env passthrough**: LLM keys forwarded only if set (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `OPENROUTER_API_KEY`, `OPENCODE_API_KEY`), plus terminal hints, `EDITOR`, `LC_ALL`. Everything else is stripped by `--clearenv`. Add new passthrough vars to the loops at `scripts/bw:429-474`.
- No package manager; nothing to install for the repo itself.

## Testing & QA

No test suite, CI, or linters exist. Verify changes by **running** the script against real invocations:

- Smoke test: `scripts/bw --help`, then `scripts/bw --offline bash` and inspect the sandbox (`mount`, `env`, `ls "$HOME"`, network reachability).
- When adding an allowlist entry or mask, confirm the path is visible/hidden inside the sandbox and that validation still rejects conflicting configs.
- Before committing, run `shellcheck` and `shfmt` locally even though they are not wired into the repo.
