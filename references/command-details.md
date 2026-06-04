# cmux-bridge Command Details

This file contains command rationale and extra examples that are too detailed for `SKILL.md`.
The core runtime rules remain in `SKILL.md`, so agents can follow the safety rules without loading this file.

## Why the Wrapper Path Is Explicit

`npx skills` installs a skill into each agent's skill directory. It does not register bundled scripts on `PATH`.
That is why the skill instructs agents to resolve the bundled `scripts/cmux-bridge` wrapper by absolute path.

During local development, the repository root is also the skill directory, so the same `$BRIDGE` pattern works against the local `scripts/cmux-bridge`.

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

```bash
# Send to a Claude Code pane in the `example-project` workspace.
cmux-bridge message --workspace workspace:11 surface:33 "[act:think id:review-1] Received. I will review it."
```

Why: `surface:N` can look unique in the UI, but cmux validates it in the caller workspace context.
When sending to a pane in another workspace, specify `--workspace` so target validation and delivery use the same context.

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
