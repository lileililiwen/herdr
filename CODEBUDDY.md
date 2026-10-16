# CodeBuddy Code support in this fork

This fork (`lileililiwen/herdr`, branch `master`) adds native **CodeBuddy Code** support to
herdr on top of upstream `herdrdev/herdr`.

- Feature commit: `ed44fc57 feat: add codebuddy integration`
- Docs commit: `e3421823 docs: document the codebuddy integration`
- Base: upstream `master` at `143bf5fd`
- Upstream status: not supported. `grep -ri codebuddy` on upstream `master` returns nothing,
  the remote detection catalog (`https://herdr.dev/agent-detection/index.toml`) has no
  `codebuddy` entry, and upstream issue
  [#567 "Feature Request: Add CodeBuddy Code agent support"](https://github.com/herdrdev/herdr/issues/567)
  was closed `NOT_PLANNED` because feature requests belong in Discussions. Issue #567 did
  document the four missing pieces (process identification, `Agent` variant, manifest, hook
  authority) and a manual workaround; this fork implements those pieces properly.

## TL;DR

```bash
# build and install from this checkout (needs Zig 0.16.0, see "Building")
cd ~/code/herdr
ZIG=~/.local/share/zig-0.16.0/zig CARGO_PROFILE_RELEASE_DEBUG=0 cargo install --path . --locked

# install the hook into ~/.codebuddy (or $CODEBUDDY_CONFIG_DIR)
herdr integration install codebuddy
herdr integration status | grep codebuddy      # codebuddy (experimental): current (v1)

# then run `codebuddy` (not a wrapper) inside a herdr pane
```

After that, a CodeBuddy Code pane shows `idle` / `working` / `blocked` from the hook instead
of screen scraping, and herdr can restore the same session with `codebuddy --resume <id>`.

## Table of contents

- [What CodeBuddy Code is](#what-codebuddy-code-is)
- [What this fork adds](#what-this-fork-adds)
- [How state and sessions work](#how-state-and-sessions-work)
- [Install, use, verify, remove](#install-use-verify-remove)
- [Why the install is "experimental and CLI-only"](#why-the-install-is-experimental-and-cli-only)
- [Files changed](#files-changed)
- [Building from source](#building-from-source)
- [Local workstation notes](#local-workstation-notes)
- [Verification performed](#verification-performed)
- [Known limits](#known-limits)
- [Troubleshooting](#troubleshooting)
- [Reverting everything](#reverting-everything)

## What CodeBuddy Code is

CodeBuddy Code is Tencent's terminal coding agent (`npm i -g @tencent-ai/codebuddy-code`).
It is a Claude Code derivative, so its extension surface is deliberately Claude-compatible.

Facts below were read out of the published npm package
(`@tencent-ai/codebuddy-code@2.168.2`, unpacked from the registry tarball), not from
documentation prose:

| Property | Value |
| --- | --- |
| Package / version inspected | `@tencent-ai/codebuddy-code@2.168.2` (Node.js ≥ 18.20.8) |
| Launchers in `bin/` | `codebuddy`, `codebuddy-code`, `cbc`, `cbc-prewarm`, `codebuddy-lowmem` (all exec the same bundle) |
| Config home | `$CODEBUDDY_CONFIG_DIR` when set, otherwise `~/.codebuddy` |
| Settings file | `~/.codebuddy/settings.json` (`settings.local.json`, project `.codebuddy/` also exist) |
| Hook schema | identical to Claude Code: top-level `hooks`, keyed by event name, each entry `{ matcher, hooks: [{ type: "command", command, timeout }] }` |
| Hook `timeout` unit | **seconds**, default 60 (`(entry.timeout ?? 60) * 1e3` in the shipped bundle) |
| Hook stdin payload | `session_id`, `hook_event_name`, `transcript_path`, `cwd`, `source` (SessionStart), `agent_id` (subagents), `tool_name`/`tool_input`/`tool_response`, `reason` (SessionEnd) |
| Hook events | `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `UserPromptSubmit`, `Stop`, `StopFailure`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PostCompact`, `SessionStart`, `SessionEnd`, `PermissionRequest`, `PermissionDenied`, `Elicitation`, `ElicitationResult`, `ConfigChange`, `InstructionsLoaded`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`, `Setup`, `WorktreeCreate`, `WorktreeRemove`, `FinalStop`, `UserPromptExpansion` |
| Resume | `codebuddy --resume <session-id>` (`-r`), also `--continue`, `--session-id`, `--fork-session` |

## What this fork adds

### 1. Agent detection

`src/detect/mod.rs`:

- `Agent::Codebuddy`, label `codebuddy`, canonical interactive executable `codebuddy`
  (so `herdr agent start --kind codebuddy` works).
- Alias table: `codebuddy`, `codebuddy-code`, `codebuddy code` (case/`.exe`/`.js` suffixes are
  normalized by the existing lookup helpers).
- Process detection for the npm layout. Running the launcher produces
  `node .../node_modules/@tencent-ai/codebuddy-code/bin/codebuddy`, so:
  - the `codebuddy` / `codebuddy-code` basenames match the alias table directly,
  - `cbc` and `codebuddy-lowmem` (whose bare names mean nothing) are recognized **only** by
    their position inside `node_modules/@tencent-ai/codebuddy-code/bin/`,
  - a non-generic process name (for example `MainThread` on platforms that do not report
    `node`) still resolves through the wrapped-runtime path, where `Codebuddy` is allowlisted
    next to `Qwen`, `Cline`, and `Letta`.
- Wrapper coverage that is intentionally *not* added: a bare `cbc` binary outside the package,
  and `codebuddy-helper`-style names, stay unrecognized (`identify_agent_in_job_rejects_unrelated_codebuddy_launchers`).

### 2. Hook integration

- `src/integration/assets/codebuddy/herdr-agent-state.sh` and `.ps1`: the installed hook.
  It reads the CodeBuddy payload from stdin and reports over herdr's UNIX socket with
  `source = "herdr:codebuddy"`, `agent = "codebuddy"`.
- `herdr integration install codebuddy` writes
  `~/.codebuddy/hooks/herdr-agent-state.sh` (`.ps1` on Windows, 0755/executable set by the
  installer), creates `~/.codebuddy/hooks/` when missing, and registers 14 events in
  `~/.codebuddy/settings.json`, preserving every unrelated key. Uninstall removes only herdr's
  entries and the managed script.
- Hook entries are written as `{ "matcher": "*", "hooks": [ { "type": "command",
  "command": "bash '<hook>' <action>", "timeout": 10 } ] }` (10 s, Claude-compatible seconds).

### 3. Session identity and resume

- The `session` report stores the pane's session id under source `herdr:codebuddy`, so herdr
  persists it and restores the pane after a server restart with `codebuddy --resume <id>`
  (`src/agent_resume.rs`).
- Session-start sources `startup`, `clear`, `resume`, and `compact` may replace the pane's
  stored session reference; other/unknown sources keep the existing one
  (`src/terminal/state.rs`).

### 4. State authority

CodeBuddy has **no screen manifest**. State comes from the hook, exactly like MastraCode
(`full_lifecycle_hook_authority` in `src/detect/mod.rs`,
`SCREEN_MANIFEST_AGENTS` deliberately unchanged).

## How state and sessions work

### Hook event to reported state

| CodeBuddy event | herdr report |
| --- | --- |
| `SessionStart` | `pane.report_agent_session` → `idle`, stores `session_id`, forwards `source` (startup/resume/clear/compact) |
| `UserPromptSubmit` | `working` |
| `PreToolUse` | `working` |
| `PostToolUse` | `working` |
| `PostToolUseFailure` | `working` |
| `PermissionRequest` | `blocked` |
| `PermissionDenied` | `working` |
| `Elicitation` | `blocked` |
| `ElicitationResult` | `working` |
| `SubagentStart` | `working` |
| `PreCompact` | `working` |
| `StopFailure` | `idle` |
| `Stop` | `idle` |
| `SessionEnd` | `idle` |

`Notification` is deliberately not registered: it fires for non-blocking notices
(`agent_completed`, `auth_success`) as well as `permission_prompt`, so it cannot express
"blocked" safely.

### Subagent filtering

Subagent hooks carry `agent_id`. The hook:

- **drops** `session` and `idle` reports from subagents (they must not claim the pane's session
  identity or clear a state the main agent still owns),
- **keeps** subagent `working` reports (that is real work happening in the pane).

### Ordering

Every report carries a monotonic `seq` (`time.time_ns()`), the same arbitration herdr uses for
other hook-owned agents. Late reports from a released run cannot resurrect a pane.

## Install, use, verify, remove

```bash
# install (writes hook + registers 14 events; requires ~/.codebuddy to exist)
herdr integration install codebuddy

# status
herdr integration status            # lists every target, plus "codebuddy (experimental): current (v1)"
herdr integration status --outdated-only

# remove (only herdr's entries and the managed hook script)
herdr integration uninstall codebuddy

# start CodeBuddy through herdr's agent automation
herdr agent start mysession --kind codebuddy --pane <pane_id>
```

The config directory must exist first: herdr refuses to create an agent's config home and says
`codebuddy config directory not found at … install codebuddy code first`.

## Why the install is "experimental and CLI-only"

`IntegrationTarget` is a **frozen v1 client-endpoint enum**: `tests/fixtures/endpoint-method-shapes-v1.json`
stores a SHA-256 digest of every advertised client-shell method's wire shape, and
`advertised_client_shell_method_shapes_stay_at_the_v1_contract` asserts it is unchanged. Adding
a variant changes the `integration.install` digest, and that test fails with
*"an existing endpoint method changed shape; add load-bearing behavior as a new advertised
method or explicitly gate new fields"*.

The project's own answer to this is `letta`: an experimental, CLI-only target listed in
`EXPERIMENTAL_INTEGRATION_TARGET_LABELS` and kept out of the client enum until an agent
registry lands. This fork registers CodeBuddy the same way:

- `herdr integration install codebuddy` works, `herdr integration status` reports
  `codebuddy (experimental)`,
- it does **not** appear in the Settings integration screen or in the JSON `integration.list`
  API response,
- agent detection and `herdr agent start --kind codebuddy` work independently of that
  restriction.

The upside is a clean, additive fork: no client-facing contract changes.

## Files changed

| File | Change |
| --- | --- |
| `src/detect/mod.rs` | `Agent::Codebuddy`, labels/executable, aliases, npm launcher paths, wrapped-runtime allowlist, full-lifecycle hook authority, tests |
| `src/integration/assets/codebuddy/herdr-agent-state.sh` / `.ps1` | New hook assets (POSIX shell + PowerShell) |
| `src/integration/mod.rs` | `CODEBUDDY_*` constants (install name, assets, version `1`, timeout `10` s, 14-event table), `codebuddy` added to `EXPERIMENTAL_INTEGRATION_TARGET_LABELS`, exports |
| `src/integration/targets.rs` | `install_codebuddy` / `uninstall_codebuddy` |
| `src/integration/actions.rs` | `install_experimental_codebuddy` / `uninstall_experimental_codebuddy` wrappers |
| `src/integration/registry.rs` | `experimental_codebuddy_integration_status` |
| `src/integration/env.rs` | `CODEBUDDY_CONFIG_DIR_ENV_VAR`, `codebuddy_dir()` |
| `src/integration/types.rs` | `CodebuddyInstallPaths`, `CodebuddyUninstallResult` |
| `src/integration/tests.rs` | Install/idempotence/uninstall/status/missing-dir tests, asset list updates |
| `src/cli/integration.rs` | `codebuddy` (and `codebuddy-code`) target parsing, dispatch, status line, help/usage text |
| `src/agent_resume.rs` | `herdr:codebuddy` official source, `codebuddy --resume <id>` plan, test |
| `src/terminal/state.rs` | Session-reference replacement rules, tests |
| `src/config/sound.rs` | Sound mapping (default, like MastraCode) |
| `docs/next/website/src/content/docs/{agents,integrations,session-state,cli-reference,agent-automation}.mdx` (+ `ja/`, `zh-cn/`) | Agent row, integration section, resume row, `--kind` lists, CLI examples |
| `CHANGELOG.md`, `docs/next/CHANGELOG.md` | Unreleased entry |

## Building from source

herdr embeds Ghostty's terminal core. `vendor/libghostty-vt` is Ghostty 1.3.2 (587 `.zig` files
and 296 C/C++ files) and `crates/ghostty-vt/build.rs` shells out to `zig build -Demit-lib-vt`,
then links the resulting static library. So building herdr requires **Zig 0.16.0**; the
repository pins it in `flake.nix` (`zig_0_16`) and in CI, and the build panics with
*"Building Herdr requires Zig 0.16.0"* when it is missing or wrong.

```bash
# one-time: Zig 0.16.0
curl -Ls -o /tmp/zig.tar.xz https://ziglang.org/download/0.16.0/zig-x86_64-linux-0.16.0.tar.xz
mkdir -p ~/.local/share/zig-0.16.0 && tar xJf /tmp/zig.tar.xz -C ~/.local/share/zig-0.16.0 --strip-components=1

# build and install into ~/.cargo/bin
cd ~/code/herdr
ZIG=~/.local/share/zig-0.16.0/zig CARGO_PROFILE_RELEASE_DEBUG=0 cargo install --path . --locked
```

- `CARGO_PROFILE_RELEASE_DEBUG=0` keeps the release artifacts small (it is not a project
  default; it was used because the volume was at 100 % capacity).
- Zig is a **build-time** dependency only. Prebuilt herdr releases need no Zig; the installed
  binary has no runtime dependency on it.
- Checks (mirroring CI): `cargo fmt --check`,
  `cargo clippy --all-targets --locked -- -D warnings`,
  `cargo nextest run --locked`, plus the Python maintenance suites in `just maintenance-test`.

## Local workstation notes

State of this machine after the fork was installed (paths are specific to `paul@paul-desk`):

| Item | Path / value |
| --- | --- |
| Checkout | `~/code/herdr` (moved from `~/tmp/herdr`), branch `master`, remotes `origin` = fork, `upstream` = `herdrdev/herdr` |
| Installed binary | `~/.cargo/bin/herdr` — `herdr 0.9.5`, built from this checkout |
| `herdr` on `PATH` | `~/.local/bin/herdr` → symlink to `~/.cargo/bin/herdr` (`~/.local/bin` precedes `~/.cargo/bin` in `PATH`) |
| Previous release | backed up at `~/.local/bin/herdr.bak-0.9.3` (0.9.3, 29 MB) |
| Zig | `~/.local/share/zig-0.16.0/zig` (396 MB) |
| CodeBuddy hook | `~/.codebuddy/hooks/herdr-agent-state.sh` + 14 entries in `~/.codebuddy/settings.json` |
| Release artifacts | `~/code/herdr/target` (553 MB; the 5.7 GB `target/debug` tree was deleted to fit the release build) |

### Switching the running server

CodeBuddy state is produced by the **server**, so the running server must run this build. On
this machine a 0.9.3 server was left running deliberately (`herdr status` shows
`server_binary_stale: yes`). To switch it:

```bash
# option A: keep live panes (the documented update path; the 0.9.3 server advertises live_handoff)
herdr server live-handoff --import-exe /home/paul/.cargo/bin/herdr

# option B: plain restart — exits the pane processes
herdr server stop
herdr
```

Both builds use `PROTOCOL_VERSION = 22` and `ENDPOINT_PROTOCOL_GENERATION = 1`, so a new client
talks to the old server (`endpoint_compatible: yes`) — but the old server has neither CodeBuddy
detection nor the hook authority, which is why the pane stays a plain terminal until the switch.

### Do not run `herdr update`

`herdr update` installs an official release and would overwrite this fork's binary. Rebuild
with the `cargo install` line above instead. The app's "new release available" notice can be
ignored.

## Verification performed

| Check | Command | Result |
| --- | --- | --- |
| Full Rust suite | `cargo nextest run --locked` | 4108 passed, 0 failed, 16 skipped |
| Lint / format | `cargo clippy --all-targets --locked`, `cargo fmt --check` | clean |
| Maintenance suites | `python3 -m unittest scripts.test_*` | 150 passed |
| Docs heading parity | `scripts/docs_translation_parity.py` | OK (en/ja/zh-cn outlines match) |
| API schema artifact | `generated_protocol_schema_artifact_is_current` | unchanged (frozen enum untouched) |
| Hook contract | fake socket server + 6 CodeBuddy payloads | correct methods/params; subagent `session`/`idle` dropped, subagent `working` kept |
| End-to-end | isolated `HOME`, real server + real pane + installed hook | `codebuddy` detected; `idle → working → blocked → idle`; session ref `herdr:codebuddy/cb-e2e-session`; uninstall removes only herdr's entries |
| CodeBuddy parses the config | `CODEBUDDY_CONFIG_DIR=<dir> codebuddy config get hooks` | exits 0, returns the herdr entries (an isolated config dir confirmed exactly one entry per event) |

## Known limits

- **No screen manifest by design.** Without the hook installed, a CodeBuddy pane is detected as
  the `codebuddy` agent but its state stays `unknown`. Adding a manifest would need real
  terminal captures of CodeBuddy's TUI, which this fork does not claim to have.
- **Windows hook is unverified on Linux.** `herdr-agent-state.ps1` mirrors the shipped
  MastraCode/Claude PowerShell hooks and passes the repository's UTF-8-stdin asset test, but it
  has not been executed on Windows here. CodeBuddy's Windows shell behavior for hook commands
  (the installer writes `powershell -NoProfile -ExecutionPolicy Bypass -File … <action>`) is
  likewise untested on this machine.
- **`cbc` alone is not recognized.** Only the package-relative launcher path is; a native
  install whose binary is literally named `cbc` will not be detected.
- **Not in the Settings integration tab or `integration.list`** — deliberate, see above.
- **Untested locally:** `bun test src/integration/assets/*.test.ts` and `just docs-contract-test`
  (bun is not installed). Those suites cover the pi/omp/opencode/kilo JavaScript assets, not the
  new shell asset.
- **Hook timings are inferred from the package, not from a live CodeBuddy session.**
  `timeout: 10` (seconds) matches the other `settings.json` integrations; the package default is
  60 s.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| CodeBuddy pane stays a plain terminal with no state | The running server is an older build. Check `herdr status` for `server_binary_stale: yes`, then live-handoff or restart the server. |
| Pane is detected as `codebuddy` but state stays `unknown` | The hook never reported: run `herdr integration status` (expect `current (v1)`), make sure CodeBuddy runs **inside a herdr pane** (`HERDR_ENV=1`, `HERDR_PANE_ID`, `HERDR_SOCKET_PATH` are set by herdr), and that `python3` exists on `PATH` for the POSIX hook. |
| `herdr integration install codebuddy` fails with `codebuddy config directory not found` | CodeBuddy Code is not installed or `~/.codebuddy` does not exist. Install it (`npm i -g @tencent-ai/codebuddy-code`) or create the directory. |
| `failed to parse …/settings.json` | herdr requires strict JSON; CodeBuddy's own file is strict JSON, but a hand-edited file with comments will be rejected. Fix the JSON first (herdr never rewrites a file it cannot parse). |
| Hook script looks replaced after an upgrade | Expected: the asset is managed by herdr and says so in its header. Put custom hooks in separate files. |
| `codebuddy config get hooks` prints every entry twice | CodeBuddy's CLI prints the compact and pretty form of the value; the settings file itself holds one entry per event. |
| Old wrapper `herdr-codebuddy` / `herdr-agent-shim` | No longer needed for CodeBuddy: native detection identifies the `codebuddy` process. Keep the shim for other agents (for example `herdr-mcode`), or run plain `codebuddy` inside a pane. |

## Reverting everything

```bash
# 1. remove the hook from CodeBuddy
herdr integration uninstall codebuddy

# 2. restore the previous herdr client
rm ~/.local/bin/herdr
mv ~/.local/bin/herdr.bak-0.9.3 ~/.local/bin/herdr

# 3. optionally drop the source build + toolchain
rm -rf ~/code/herdr/target ~/.local/share/zig-0.16.0

# 4. (only if you had also switched the server) restart it on the restored binary
herdr server stop && herdr
```
