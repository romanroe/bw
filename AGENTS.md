# Repository Guidelines

## Project Overview

`bw` is a Bash wrapper around [`bwrap`](https://github.com/containers/bubblewrap) (bubblewrap) that runs coding-agent CLIs (opencode, omp, codex, claude, junie) inside an unprivileged Linux sandbox. It default-denies access to all host data, exposes only an explicit allowlist of paths, hides secrets, and controls environment and network. The entire project is three shell scripts — there is no application source, build system, or package.

## Architecture & Data Flow

Layering: `bwoc`/`bwomp` → `bw` → `bwrap`.

`scripts/bw` is a linear, fail-closed pipeline (`scripts/bw:1` sets `set -euo pipefail`):

1. **Parse options** — `--workdir`, `--mask`, `--offline`, `--allow-env-file`, `--docker`, `--help`; the rest is `command [args...]`.
2. **Validate** — canonicalize every path; refuse `workdir == HOME` or `workdir` inside `sandbox_home`; reject masks that would hide the workdir/sandbox home; require every read-only/writable HOME path and file to exist. With `--docker`, reject `--offline`, resolve Docker CLI endpoint precedence, and accept only an existing local Unix socket. A default `.env.sandbox` and its `.env` destination must both be regular non-symlink files.
3. **Assemble `bwrap` args** — per-user state dir for the deny mount sources; private `/proc`, `/dev`, tmpfs `/tmp`; `--clearenv`; read-only system + tool mounts; writable allowlist + workdir; masks; `.env` hiding or `.env.sandbox` mapping; optional Wayland and Docker sockets; network toggle.
4. **Resolve command** — process name symlink; alias vs. host binary vs. path; augment `PATH`.
5. **Exec** `exec -a "$BW_ARGV0" "${BW_EXEC[@]}" "${BWRAP_ARGS[@]}" -- "$@"`.

**Security model:** `SANDBOX_HOME` (`$HOME/sandbox_home`, mode `0700`) is bind-mounted over `$HOME` as a persistent fallback; specific HOME mounts overlay it. `--unshare-all` drops all namespaces; `--share-net` opts networking back in only when not `--offline`. Masks and hidden environment files overlay an empty read-only dir/file (`EMPTY_DENY_DIR`/`EMPTY_DENY_FILE`). Those two objects plus the process-name symlinks live in a per-user state dir (`$XDG_RUNTIME_DIR/bw`, falling back to `${TMPDIR:-/tmp}/bw-$UID`), guarded by symlink/ownership/emptiness checks and never mounted into the sandbox; there is no cleanup trap because `bw` `exec`s `bwrap`.

By default, a regular `.env.sandbox` at the workdir root is mounted read-only over an existing regular `.env`; its original name and all other regular `.env` files remain hidden. Missing destinations, symlinks, and non-regular files are rejected to prevent bind setup from mutating the host project. `--allow-env-file` instead exposes the real `.env` files. `--docker` is a separate opt-in: it bind-mounts only the resolved local Docker Unix socket at `/tmp/runtime/docker.sock`, sets a fixed internal `DOCKER_HOST`, rejects `--offline`, and never exposes `~/.docker`.

**Process naming:** `bw` `exec`s `bwrap` through a symlink named after the requested program (`$BW_NAME_DIR/<name>` → the real `bwrap`) with `exec -a "<name>"`, so the pty's foreground process-group leader has both `comm` and `argv[0]` set to e.g. `opencode`. tmux window names (`#{pane_current_command}`) and Konsole tab titles (`%n`) therefore show the sandboxed program instead of `bash`; `ps` shows it too. Symlink creation failures fall back to the plain binary — naming never blocks a run.

## Key Directories

- `scripts/` — all executables. `bw` (main, 708 lines), `bwoc`, `bwomp`.
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
scripts/bw --allow-env-file -- opencode         # expose the real .env files
scripts/bw --docker -- omp                      # expose the local Docker daemon
scripts/bw --workdir /path/to/proj -- omp       # mount a specific RW workdir
scripts/bw --help
```

Recommended (not present) checks for changes: `shellcheck scripts/bw` and `shfmt -d scripts/bw`.

## Code Conventions & Common Patterns

- **Bash strict mode**: `set -euo pipefail` at the top of every non-trivial script.
- **Helpers**: `require_cmd` (dependency preflight), `canon_path` (expands `~`, `readlink -m`), `is_subpath_or_same` (path containment), `ensure_dir` (mode-exact `mkdir`, tolerates a concurrent run) — reuse these; do not reimplement path logic inline.
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
- **Required host commands** (preflighted): `bwrap`, `readlink`, `mkdir`, `ln`. `docker` is required only for `--docker`. The real `bwrap` binary is resolved once into `BWRAP_BIN` because the name symlink must point at the executable, not at `bwrap` on `PATH`.
- **Node runtime**: a mise-managed Node at `~/.local/share/mise/installs/node/latest/bin/node` is mandatory and causes exit 2 if missing. It is prepended to the sandbox `PATH`.
- **Read-only tool mounts** assumed present: `~/.pyenv`, `~/.local/bin`, `~/.local/share/{mise,pipx}`, `~/.config/mise`, `~/.config/pypoetry/config.toml`.
- **Env passthrough**: LLM keys forwarded only if set (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `OPENROUTER_API_KEY`, `OPENCODE_API_KEY`), plus terminal hints, `EDITOR`, `LC_ALL`. Everything else is stripped by `--clearenv`. Add new passthrough variables to the existing environment loops.
- No package manager; nothing to install for the repo itself.

## Testing & QA

No test suite, CI, or linters exist. Verify changes by **running** the script against real invocations:

- Smoke test: `scripts/bw --help`, then `scripts/bw --offline bash` and inspect the sandbox (`mount`, `env`, `ls "$HOME"`, network reachability).
- For `.env.sandbox`, confirm that it appears read-only as `.env`, its source name stays hidden, unrelated `.env.*` files stay hidden, and symlinks or missing destinations fail closed without creating host files.
- For `--docker`, confirm the default sandbox has no Docker socket, invalid/offline endpoints fail before `bwrap`, and both `docker ps` and a temporary Unix echo socket work through the mounted path.
- When adding an allowlist entry or mask, confirm the path is visible/hidden inside the sandbox and that validation still rejects conflicting configs.
- Before committing, run `shellcheck` and `shfmt` locally even though they are not wired into the repo.
