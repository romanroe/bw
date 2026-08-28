# bw

`bw` is an opinionated Bash wrapper around [bubblewrap](https://github.com/containers/bubblewrap) (`bwrap`). It runs coding-agent CLIs such as opencode, OMP, Codex, Claude, and Junie inside an unprivileged Linux sandbox.

The default policy is fail-closed:

- Host data is invisible unless it is explicitly mounted.
- The selected project is writable; configured tool paths are read-only or writable according to fixed allowlists.
- Project environment files are hidden by default.
- The process environment is cleared and rebuilt from a small allowlist.
- Network access can be disabled per run.
- Docker daemon access is disabled unless `--docker` is supplied.

This is not a general-purpose sandbox builder. The path allowlists and runtime assumptions match one workstation setup. Fork the repository and edit `scripts/bw` before using it on a different machine.

## Features

- Unprivileged sandboxing through bubblewrap and Linux user namespaces.
- Private `/proc`, `/dev`, `/tmp`, `/var/tmp`, and runtime directory.
- Read-only system and tool mounts.
- Writable project directory and explicit writable agent-state allowlist.
- Persistent sandbox home for writes outside explicitly mounted HOME paths.
- Repeatable path masking with `--mask`.
- Default project `.env` hiding.
- Optional `.env.sandbox` to read-only `.env` mapping.
- Explicit access to real environment files with `--allow-env-file`.
- Network isolation with `--offline`.
- Opt-in local Docker daemon access with `--docker`.
- Optional Wayland socket forwarding for clipboard support.
- Fixed environment-variable passthrough for terminal integration and LLM API keys.
- Host command, explicit path, and shell alias resolution.
- Process naming that keeps terminal and tmux titles on the sandboxed command instead of `bash` or `bwrap`.
- Thin `bwoc` and `bwomp` launchers.

## Requirements

- Linux with unprivileged user namespaces enabled.
- Bash.
- Host commands `bwrap`, `readlink`, `mkdir`, and `ln`. These are checked at startup; a missing command produces exit code `127`.
- `docker` on the host only when `--docker` is used.
- A mise-managed Node executable at:

  ```text
  ~/.local/share/mise/installs/node/latest/bin/node
  ```

  This runtime is mandatory and is prepended to the sandbox `PATH`.

The default read-only HOME allowlist requires these paths to exist:

```text
~/.pyenv
~/.local/bin
~/.local/share/mise
~/.local/share/pipx
~/.config/mise
~/.cargo/bin
~/.rustup
~/.config/pypoetry/config.toml
```

The writable allowlist is also workstation-specific and every entry must already exist:

```text
~/.claude
~/.codex
~/.junie
~/.local/share/kwrite
~/.local/share/claude
~/.local/share/junie
~/.local/state
~/.config/opencode
~/.cache/opencode
~/.local/share/opencode
~/.local/state/opencode
~/.opencode
~/.omp
~/.claude.json
~/.config/kwriterc
~/.local/bin/junie
```

Poetry's `auth.toml`, `~/.docker`, and the rest of HOME are deliberately not mounted.

## Installation

The launchers call the bare command `bw`, so `scripts/bw` must be resolvable through `PATH`.

```bash
# Option 1: add the repository scripts directory to PATH
export PATH="/path/to/bw/scripts:$PATH"

# Option 2: create command symlinks
ln -s /path/to/bw/scripts/bw ~/.local/bin/bw
ln -s /path/to/bw/scripts/bwoc ~/.local/bin/bwoc
ln -s /path/to/bw/scripts/bwomp ~/.local/bin/bwomp
```

## Usage

```text
bw [options] -- command [args...]
bw [options] command [args...]
```

`--` ends wrapper option parsing. It is recommended when the target command or its arguments begin with `-`.

### Options

| Option | Description |
| --- | --- |
| `--workdir DIR` | Select the directory mounted read-write and used as the sandbox working directory. Default: the current directory. |
| `--mask PATH` | Hide an existing host path inside the sandbox. Repeat the option for multiple paths. |
| `--offline` | Keep the network namespace isolated. Without this flag, the sandbox shares the host network namespace. |
| `--allow-env-file` | Expose the real regular `.env` and `.env.*` files instead of applying the default hiding and `.env.sandbox` mapping. |
| `--docker` | Expose the selected local Docker Unix socket. This breaks normal sandbox isolation and cannot be combined with `--offline`. |
| `--help`, `-h` | Print command help and exit. |

### Examples

```bash
bw opencode
bw omp
bw codex
bw claude
bw junie

bw --offline -- bash
bw --workdir /path/to/project -- omp
bw --mask "$HOME/Nextcloud/Private" -- opencode
bw --allow-env-file -- opencode
bw --docker -- omp
bw --docker --workdir /path/to/project -- omp
```

`bwoc` is equivalent to `bw opencode`, and `bwomp` is equivalent to `bw omp`.

Wrapper options cannot be passed through these launchers because the fixed agent command comes first. Use `bw --docker -- omp`, not `bwomp --docker`.

## Project environment files

### Default behavior

Without `--allow-env-file`, `bw` recursively finds regular `.env` and `.env.*` files in the workdir and overlays them with an empty read-only file. These well-known templates remain visible:

```text
.env.example
.env.sample
.env.template
.env.dist
```

Other environment-file symlinks are not followed or masked. Do not use symlinks as secret environment files.

### `.env.sandbox`

A regular `.env.sandbox` at the workdir root can provide a restricted environment file for agents and tests. `bw` mounts it read-only as the root `.env`, then hides its original `.env.sandbox` name.

Example:

```dotenv
# .env.sandbox
DB_HOST=localhost
DB_PORT=5432
DB_NAME=project_test
DB_USER=project_test
DB_PASSWORD=replace-me
```

Run the agent normally:

```bash
bw --docker -- omp
```

The mapping has strict preconditions:

- `.env.sandbox` must be a regular file, not a symlink, directory, FIFO, or device.
- A regular non-symlink `.env` destination must already exist at the workdir root.
- Both files should be ignored by version control when they contain credentials.

The existing `.env` requirement prevents bubblewrap from creating a mount destination in the writable host project. An empty placeholder is sufficient:

```bash
: > .env
```

Inside the sandbox:

- `.env` contains the `.env.sandbox` data and is read-only.
- `.env.sandbox` is replaced by an empty read-only file.
- Other regular `.env` files remain hidden.

With `--allow-env-file`, no automatic mapping occurs and the real environment files are exposed unchanged.

## Docker access

`--docker` gives processes inside the sandbox control of the selected local Docker daemon. Docker daemon access can usually start privileged containers, mount arbitrary host paths, and bypass filesystem or network restrictions. Treat it as a deliberate reduction of isolation.

Endpoint resolution follows Docker CLI environment precedence:

1. A non-empty `DOCKER_HOST`.
2. A non-empty `DOCKER_CONTEXT`, resolved with `docker context inspect`.
3. The active Docker context.

Only absolute local `unix:///...` endpoints are accepted. TCP and SSH endpoints are rejected. The socket must exist and be a Unix socket before bubblewrap starts.

The host socket is mounted read-write at:

```text
/tmp/runtime/docker.sock
```

The sandbox receives this fixed value:

```text
DOCKER_HOST=unix:///tmp/runtime/docker.sock
```

The host `DOCKER_HOST` and `DOCKER_CONTEXT` values are removed by `--clearenv`; `~/.docker` is not mounted. Private registry configuration and credentials therefore remain unavailable unless separately exposed by changing the wrapper.

`--docker` and `--offline` are mutually exclusive because daemon access can bypass network isolation.

## Filesystem policy

| Path or resource | Sandbox access |
| --- | --- |
| Selected workdir | Read-write |
| `~/sandbox_home` | Read-write fallback mounted over HOME |
| `SYSTEM_RO_PATHS` | Read-only when present |
| `READONLY_HOME_PATHS`, `READONLY_HOME_FILES` | Read-only; every entry must exist |
| `WRITABLE_HOME_PATHS`, `WRITABLE_HOME_FILES` | Read-write; every entry must exist |
| `/proc`, `/dev` | Private instances |
| `/tmp`, `/var/tmp` | Private tmpfs |
| `/tmp/runtime` | Private, mode `0700` |
| Wayland socket | Read-write when a valid host Wayland socket is detected |
| Docker socket | Read-write only with `--docker` |
| Other host paths | Not mounted |

The workdir cannot equal HOME or be located inside `~/sandbox_home`. A mask cannot contain the workdir, sandbox home, or a required allowlist entry. Conflicting configurations fail before bubblewrap starts.

Masks replace existing paths with persistent empty read-only mount sources from a per-user state directory:

```text
$XDG_RUNTIME_DIR/bw
```

If `XDG_RUNTIME_DIR` is unsuitable, the fallback is `${TMPDIR:-/tmp}/bw-$UID`. Ownership, symlinks, and deny-source contents are validated before use. The state directory is not mounted into the sandbox.

## Environment policy

`--clearenv` removes the host environment. `bw` then sets core values including:

```text
HOME USER LOGNAME PATH BROWSER TERM LANG
XDG_CACHE_HOME XDG_CONFIG_HOME XDG_RUNTIME_DIR
PIP_CACHE_DIR NPM_CONFIG_CACHE
```

`LC_ALL` and `EDITOR` are forwarded only when set.

Terminal integration variables are forwarded when set:

```text
COLORTERM TERM_PROGRAM TERM_PROGRAM_VERSION
KITTY_WINDOW_ID KITTY_PID
WEZTERM_PANE WEZTERM_UNIX_SOCKET
GHOSTTY_RESOURCES_DIR GHOSTTY_BIN_DIR
VTE_VERSION KONSOLE_VERSION ALACRITTY_SOCKET
TMUX STY COLUMNS LINES
```

LLM credentials are forwarded when set:

```text
OPENAI_API_KEY
ANTHROPIC_API_KEY
GEMINI_API_KEY
GOOGLE_API_KEY
OPENROUTER_API_KEY
OPENCODE_API_KEY
```

Other variables, including project-specific database credentials, are not forwarded. Put restricted project values in `.env.sandbox` or use `--allow-env-file` explicitly.

The sandbox `PATH` starts with the configured Cargo and Node bin directories, followed by standard system directories. Existing host `PATH` entries are retained only when they are directories below a configured read-only HOME path. The resolved target command's directory is also prepended.

## Network and desktop integration

Networking is enabled by default by adding `--share-net` after `--unshare-all`. `--offline` omits `--share-net`, leaving the sandbox in its isolated network namespace.

If `WAYLAND_DISPLAY` names a socket directly below a valid owned `XDG_RUNTIME_DIR`, that socket is mounted into the private runtime directory and `WAYLAND_DISPLAY` is forwarded. X11 `DISPLAY` and X11 sockets are not exposed.

The resolved `/etc/resolv.conf` target is mounted read-only when available so DNS works without mounting all of `/run`.

## Command resolution and process names

For a command without `/`, `bw` resolves in this order:

1. A simple interactive Bash alias.
2. A host command found through `PATH`.

Explicit paths are passed directly. Absolute command symlinks are resolved to their final target before execution. A missing command produces exit code `127`.

Before executing bubblewrap, `bw` creates a command-named symlink to the real `bwrap` executable in its private state directory and uses `exec -a`. This makes `comm`, `argv[0]`, tmux window names, and compatible terminal tab titles show names such as `omp` or `opencode`. Symlink creation failure is cosmetic and falls back to the plain `bwrap` executable.

## Configuration

There is no configuration file or general override system. Edit the arrays near the top of `scripts/bw`:

- `SYSTEM_RO_PATHS`: read-only system paths, skipped when absent.
- `READONLY_HOME_PATHS`: required read-only HOME directories.
- `READONLY_HOME_FILES`: required read-only HOME files.
- `WRITABLE_HOME_PATHS`: required writable HOME directories.
- `WRITABLE_HOME_FILES`: required writable HOME files.
- `MASKED_PATHS`: initial masks; `--mask` appends per-run entries.

Keep mounts narrow. In particular, do not mount all of `~/.cargo`, `~/.docker`, `/run`, or HOME merely to make one tool work.

## Exit codes

| Code | Meaning |
| --- | --- |
| `0` | Help completed or the sandboxed command succeeded. |
| `2` | Invalid or conflicting wrapper configuration, missing required path, invalid `.env.sandbox`, or rejected Docker endpoint. |
| `127` | A required host command or the target command was not found. |
| Other | Exit status from bubblewrap or the sandboxed command. |

## Development and verification

There is no project test suite or CI configuration. Useful checks after changing `scripts/bw`:

```bash
bash -n scripts/bw
scripts/bw --help
shellcheck scripts/bw
shfmt -d scripts/bw
```

Behavioral checks should exercise the changed path:

```bash
# Default Docker isolation
scripts/bw -- bash -c 'test -z "${DOCKER_HOST:-}" && test ! -S /tmp/runtime/docker.sock'

# Docker access
scripts/bw --docker -- docker ps

# Network isolation
scripts/bw --offline -- bash -c 'ip route'
```

For `.env.sandbox`, verify the mapped `.env` contents, read-only mount, hidden source name, hidden unrelated `.env.*` files, and fail-closed handling for symlinks and missing destinations.

## License

MIT © 2026 Roman Roelofsen. See [LICENSE](LICENSE).
