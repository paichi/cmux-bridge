# cmux-bridge Command Details

This file contains command rationale and extra examples that are too detailed for `SKILL.md`.
The core runtime rules remain in `SKILL.md`, so agents can follow the safety rules without loading this file.

## Why the Wrapper Path Is Explicit

`npx skills` installs a skill into each agent's skill directory. It does not register bundled scripts on `PATH`.
That is why the skill instructs agents to resolve the bundled `scripts/cmux-bridge` wrapper by absolute path.

During local development, the repository root is also the skill directory, so resolving `$BRIDGE` next to this `SKILL.md` points at the local `scripts/cmux-bridge`.

## Extracting a Received Message in Bash

```bash
line='[cmux-bridge from:codex reply-to:surface:104] The memoized Fibonacci version is this'
if [[ "$line" =~ ^\[cmux-bridge\ from:([^\ ]+)\ reply-to:(surface:[0-9]+)\]\ (.+)$ ]]; then
    from="${BASH_REMATCH[1]}"       # codex
    reply_to="${BASH_REMATCH[2]}"   # surface:104
    body="${BASH_REMATCH[3]}"       # The memoized Fibonacci version is this
fi
```

When replying, use `reply-to` as the target exactly as received.

## Sending Across Workspaces

A `surface:N` in another workspace can be targeted **without** `--workspace`; the wrapper reverse-looks-up its owning workspace automatically. A reply to a received `reply-to:surface:N` therefore reaches the other workspace as-is.

```bash
# No --workspace: the owning workspace is auto-resolved.
cmux-bridge message surface:33 "[act:think id:review-1] Received. I will review it."

# Explicit --workspace still works and takes precedence:
cmux-bridge message --workspace workspace:11 surface:33 "[act:think id:review-1] Received. I will review it."
```

Why: `surface:N` is globally unique across workspaces, so the wrapper resolves the owning workspace from `cmux tree --all --json` and aligns target validation and delivery to it. If the lookup is not unique (not found / multiple / tree failure), it does not guess: it degrades to the previous exit 6 instead of misdelivering. An explicit `--workspace` always wins.

## If cmux Context Is Missing

If `cmux-bridge id` exits 8, the wrapper cannot identify the sender pane from cmux context.
Do not guess a `reply-to` target. Run the agent from inside the cmux terminal pane that should receive replies.

## Sender Override

```bash
export CMUX_AGENT_NAME=claude
# Subsequent `message` calls use this value in the `from:` field.
```

Sender detection order:

1. `$CMUX_AGENT_NAME`
2. `$CLAUDECODE == "1"` -> `claude-code`
3. Non-empty `$CODEX_THREAD_ID` -> `codex`
4. `$(whoami)`

## Enter Commit Delay (CMUX_BRIDGE_COMMIT_DELAY)

`message` / `submit` / `clear` inject an Enter key event after the body to
commit the submit. If the receiving TUI is still ingesting the pasted text,
the Enter can be swallowed, so the wrapper sends **delay -> enter -> delay ->
backup enter**. Each delay (in seconds) is controlled by
`CMUX_BRIDGE_COMMIT_DELAY`.

| env | Meaning | Default |
|-----|---------|---------|
| `CMUX_BRIDGE_COMMIT_DELAY` | `sleep` seconds around the Enter commit (non-negative number) | `0.4` |

Rules:

- Applies to `message` (except `--dry-run`) / `submit` / `clear` only. `send`
  and `key` have no Enter commit and ignore it.
- Unset defaults to `0.4`. `0` is allowed (send the two Enters with no wait).
- Only a non-negative number is accepted (e.g. `0.25`, `1`). Empty value,
  multiple dots, or non-numeric input exits 2 (no fallback).
- Validation runs before any cmux call; an invalid value emits no identify /
  send / send-key.

```bash
# Shorten the delay to reduce latency when scripting many sends
export CMUX_BRIDGE_COMMIT_DELAY=0.1
cmux-bridge message surface:104 "[act:ask id:ping-1] Please respond"
```

## spawn — Launch an Agent in a New Pane or Tab

```bash
cmux-bridge spawn --agent <claude|codex|cursor> [--model <model>] [--cwd <path>] [--placement pane|tab] [--dry-run]
```

On success, stdout is a single line: the new surface ref (for example
`surface:91`). Use it as the `message` target. Warnings (such as rename-tab
failure) go to stderr and are non-fatal.

### Agent to Launch Command Mapping

| `--agent` | binary | model flag when `--model` is set |
|-----------|--------|----------------------------------|
| `claude` | `claude` | `--model <model>` |
| `codex` | `codex` | `-m <model>` |
| `cursor` | `cursor-agent` | `--model <model>` |

If `--model` is omitted, no model flag is appended. Model names are not
whitelisted; only character validation applies (`^[A-Za-z0-9._-]+$`, exit 2
on mismatch).

### Options

| Option | Behavior |
|--------|----------|
| `--agent` | Required. `claude`, `codex`, or `cursor` only (exit 2 otherwise). |
| `--model` | Optional. Omitted = no model flag on the launch command. |
| `--cwd` | Optional. Directory must exist (exit 2 if not). Injects `cd <path> && ...` with `printf '%q'` quoting. |
| `--placement` | Optional. `pane` (default, split right) or `tab` (`new-surface`). Other values exit 2. |
| `--dry-run` | After argument validation, prints the launch command on stdout and exits 0 without calling cmux. |

### Workspace Resolution (non-dry-run)

1. `CMUX_BRIDGE_SELF_WORKSPACE` and `CMUX_BRIDGE_SELF_SURFACE` both non-empty:
   use the workspace side (partial pair exits 2, same rule as elsewhere).
2. Else non-empty `CMUX_WORKSPACE_ID`.
3. Else `cmux identify --json` `caller.workspace_ref`.
4. All fail: exit 8 (running outside cmux context).

Self env values follow the ref/UUID-only rule: bare index is rejected with
exit 2; there is no fallback to plain identify.

### Exit Codes (spawn)

| Exit | Meaning |
|------|---------|
| 0 | Success (stdout = new surface ref) |
| 1 | cmux runtime error (pane creation, send, etc.) |
| 2 | Argument error / partial self env pair / invalid self env value |
| 3 | cmux output schema violation (cannot parse the surface ref) |
| 8 | cmux context unavailable (workspace cannot be resolved) |

### Example

```bash
new="$(cmux-bridge spawn --agent claude --model haiku)"
# -> surface:91
sleep 3   # or: cmux-bridge read "$new" 20   (once, not polled)
cmux-bridge message "$new" "Please read the cmux-bridge skill. [act:ask id:spawn-1] Ready to pair?"
```

### Failure Note

If command injection fails after the pane or tab was created, the empty
surface may remain open. Close it manually in cmux if needed.
