# bw

`bw` is a small Bash wrapper around [bubblewrap](https://github.com/containers/bubblewrap) (`bwrap`) that runs coding-agent CLIs — such as **opencode**, **omp**, **codex**, **claude**, and **junie** — inside an unprivileged Linux sandbox.

It is **fail-closed**: the sandbox default-denies access to all host data and exposes only an explicit allowlist of paths. Secrets are hidden, the environment is stripped to a small predictable set, and network access is controllable per run.

## Why

Coding agents execute arbitrary commands and read whatever they can reach. `bw` confines them so a run can only see:

- The current project directory (read-write).
- A fixed allowlist of tool directories and agent config/state under `$HOME`.
- A minimal, read-only system layout (`/usr`, `/bin`, `/etc`, …).

Everything else in your home directory is invisible. Project `.env` files are hidden by default, and LLM API keys are forwarded only when already set in your environment.

## Requirements

- **Linux only** — requires unprivileged user namespaces (bubblewrap).
- Host commands on `PATH`: `bwrap`, `readlink`, `mktemp`, `mkdir` (checked at startup; missing → exit `127`).
- A [mise](https://mise.jdx.dev/)-managed Node runtime at `~/.local/share/mise/installs/node/latest/bin/node` (required; missing → exit `2`). It is prepended to the sandbox `PATH`.
- The following read-only tool locations are expected to exist and are mounted into the sandbox:
  - `~/.pyenv`, `~/.local/bin`, `~/.local/share/mise`, `~/.local/share/pipx`, `~/.config/mise`
  - `~/.config/pypoetry/config.toml`

Agent config/state directories in the writable allowlist (e.g. `~/.claude`, `~/.codex`, `~/.config/opencode`) must already exist; the script aborts if a listed path is missing. Edit the allowlist arrays at the top of `scripts/bw` to match your setup.

## Install

The `bwoc` and `bwomp` launchers call the bare command `bw`, so `bw` must be resolvable on your `PATH`. Put the `scripts/` directory on `PATH`, or symlink the three scripts into a directory already on it:

```bash
# Option A: add scripts/ to PATH (e.g. in ~/.bashrc)
export PATH="/path/to/bw/scripts:$PATH"

# Option B: symlink into an existing PATH directory
ln -s /path/to/bw/scripts/bw   ~/.local/bin/bw
ln -s /path/to/bw/scripts/bwoc ~/.local/bin/bwoc
ln -s /path/to/bw/scripts/bwomp ~/.local/bin/bwomp
```

## Usage

```
bw [options] -- command [args...]
bw [options] command [args...]
```

### Options

| Option | Description |
| --- | --- |
| `--workdir DIR` | Directory mounted read-write. Default: current directory. |
| `--mask PATH` | Hide this host path inside the sandbox. Repeatable. |
| `--offline` | Disable network access. |
| `--allow-env-file` | Expose `.env` / `.env.*` files in the workdir (hidden by default). |
| `--help`, `-h` | Show usage. |

Everything after the options (or after `--`) is the command and its arguments. The command is resolved as a shell alias, a host binary on `PATH`, or an explicit path.

### Examples

```bash
bw opencode                                  # run opencode in the sandbox (network on)
bwoc                                         # shorthand for: bw opencode
bwomp                                        # shorthand for: bw omp

bw --offline bash                            # a shell with no network
bw --mask "$HOME/Nextcloud/Privat" -- opencode   # hide an extra path
bw --allow-env-file -- opencode              # expose project .env files
bw --workdir /path/to/proj -- omp            # mount a specific RW workdir
bw --help
```

## How it works

Layering: `bwoc` / `bwomp` → `bw` → `bwrap`.

`scripts/bw` is a linear, fail-closed pipeline:

1. **Parse options** — `--workdir`, `--mask`, `--offline`, `--allow-env-file`, `--help`; the rest is `command [args...]`.
2. **Validate** — canonicalize every path; refuse `workdir == HOME` or a workdir inside the sandbox home; reject masks that would hide the workdir or sandbox home; require every read-only/writable path and file in the allowlist to exist.
3. **Assemble the sandbox** — private `/proc`, `/dev`, and tmpfs `/tmp`; `--clearenv`; read-only system and tool mounts; the writable allowlist plus the workdir; masks; `.env` hiding; the Wayland socket; and the network toggle.
4. **Resolve the command** — shell alias vs. host binary vs. explicit path; augment `PATH` accordingly.
5. **Exec** `bwrap … -- <command>`.

### Security model

- `~/sandbox_home` (mode `0700`, created on first run) is bind-mounted over `$HOME` as a persistent fallback; specific HOME mounts overlay it, so writes outside the allowlist land in `sandbox_home` instead of your real home.
- `--unshare-all` drops all namespaces; `--share-net` opts networking back in only when `--offline` is not set.
- Masked paths and hidden `.env` files are overlaid with an empty read-only directory/file, cleaned up on exit.
- After `--clearenv`, only a small set of variables is forwarded: `HOME`, `USER`, `PATH`, terminal hints (`TERM`, `COLORTERM`, `TMUX`, …), `EDITOR`, `LANG`/`LC_ALL`, XDG cache/config dirs, and LLM keys **if already set** (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `OPENROUTER_API_KEY`, `OPENCODE_API_KEY`).

## Configuration

The exposed paths are declared as arrays at the top of `scripts/bw`. Edit these to fit your machine; every listed entry is asserted to exist at runtime:

- `SYSTEM_RO_PATHS` — read-only system mounts.
- `READONLY_HOME_PATHS`, `READONLY_HOME_FILES` — read-only tool dirs/files under `$HOME`.
- `WRITABLE_HOME_PATHS`, `WRITABLE_HOME_FILES` — writable agent config/state under `$HOME`.
- `MASKED_PATHS` — paths to hide (usually set per run with `--mask`).

## Exit codes

- `2` — invalid/conflicting configuration or a required path missing.
- `127` — a required host command or the target command was not found.

## License

MIT © 2026 Roman Roelofsen. See [LICENSE](LICENSE).
