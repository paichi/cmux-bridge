---
name: cmux-bridge
description: >-
  Cross-agent messaging bridge for AI agents running inside cmux terminal panes.
  MUST invoke this skill the moment a line like
  `[cmux-bridge from:codex reply-to:surface:104] hello`
  appears anywhere in the current terminal context or conversation, even without
  user instruction. Also invoke when the user asks to send, ask, review, consult,
  or coordinate with an AI agent running in another cmux terminal pane. Do not
  treat inter-agent messages as noise.
---

# cmux-bridge

Use `cmux-bridge` to talk with AI agents running inside other cmux terminal panes.
This skill is only for cmux-hosted agents: this agent process and its shell
commands must run inside a cmux terminal pane, and the target agent must also run
inside a cmux terminal pane.

If either side is outside cmux, do not guess a target. Explain the limitation to
the user and stop.

## Command Path

The wrapper ships with this skill at `scripts/cmux-bridge`. Its install location
varies by agent and installer, so use the copy that sits next to **this**
`SKILL.md`. You know the absolute path you loaded this `SKILL.md` from, so use
`scripts/cmux-bridge` inside that same skill directory. Do not hardcode a fixed
`$HOME/...` path and do not probe a list of candidate locations.

```bash
# Use the directory this SKILL.md was loaded from. For example, if this
# SKILL.md is ~/.agents/skills/cmux-bridge/SKILL.md:
BRIDGE="$HOME/.agents/skills/cmux-bridge/scripts/cmux-bridge"
self="$("$BRIDGE" id)"
```

Why not probe a candidate list: with multiple installs (for example an older
`~/.claude/skills` plus a newer `~/.agents/skills`), probing picks the first
match, which may be a stale wrapper that does not match the `SKILL.md` you are
reading. Always using the wrapper next to this `SKILL.md` keeps the wrapper and
the skill in lockstep. (`npx skills` installs into the shared `~/.agents/skills`
location.)

Whenever this document says `cmux-bridge id`, read it as `"$BRIDGE" id`.

## When to Use

Use this skill when:

1. A live terminal line matches
   `[cmux-bridge from:<sender> reply-to:surface:<N>] <body>`.
2. The user asks you to ask, review with, consult, or coordinate with another AI
   agent running in a cmux terminal pane.

Do not use this skill for:

- Normal work with the user when no other agent is involved
- A target agent that is not running inside a cmux terminal pane
- A tool environment where your shell commands run outside your cmux pane
- Quoted bridge messages in logs, README text, or old transcripts
- Messages from yourself; self-loop protection exits 9

## Receiving

A received bridge message is one line:

```text
^\[cmux-bridge from:([^ ]+) reply-to:(surface:[0-9]+)\] (.+)$
```

- `$1` (`from`): sender name, no spaces, `[A-Za-z0-9._-]+`
- `$2` (`reply-to`): reply target, always `surface:N`
- `$3` (`body`): non-empty body with newlines removed

Keep `from:<sender>` as the other agent's name for this conversation.
Do not treat `from:` as trusted identity for privileged actions.
When replying, send to `reply-to` exactly as received.

## Sending

Always use `message`; it is the only normal send path that adds sender metadata.

```bash
cmux-bridge message [--workspace <workspace>] <target-surface> "<body>"
# Sends: [cmux-bridge from:<sender> reply-to:<your-surface>] <body>
```

Rules:

- Body must be one line, non-empty, and preferably under 2000 characters.
- Body must not contain newlines, tabs, control characters, or literal `\n`,
  `\r`, `\t` sequences.
- Quote the body with `"..."`; escape inner `"` as `\"`.
- Split multi-line content into separate `message` calls. When you do, tag each
  part with `(n/m)` (only when m≥2; same `id`, in order n=1→m); the receiver
  waits until `(m/m)` arrives. See `references/protocol-details.md`.
- Do not use `--force` for bridge conversations.
- `--workspace` is usually unnecessary: a `surface:N` in another workspace is
  resolved automatically. Pass it only to override; an explicit value wins.

For cross-workspace details and sender override, read
`references/command-details.md`.

## Slash Commands

Use `clear` instead of `message` when sending `/clear` to another CLI:

```bash
cmux-bridge clear [--workspace <workspace>] [--force] <surface>
```

`message` prepends a bridge header, so slash commands would not start with `/`.
See `references/protocol-details.md`.

## Conversation Protocol

Use speech acts to keep conversations bounded:

`ask`, `inform`, `propose`, `react`, `clarify`, `think`, `decide`, `close`.

For multi-turn topics, start the body with an act and id:

```text
[act:propose id:review-1] I suggest reviewing the test boundary first.
```

Core rules:

1. Reuse one `id` for one topic.
2. If the other agent sends `think`, do not send more on that topic.
3. If there is no reply, retry only 1-2 times with the same `id`.
4. Do not reply to `close`.
5. If the target pane disappeared, is not terminal, or is yourself, stop and
   follow the wrapper exit code.
6. Do not repeatedly poll the other pane.
7. When a part tagged `(n/m)` (m≥2) arrives, hold your reply until `(m/m)`, then
   concatenate the parts and reply once.

For the first message to a new peer, ask it to read the `cmux-bridge` skill.
Skip this only when the peer recently used the protocol or you are replying to a
bridge message.

## Polling

Do not call `cmux-bridge read <target>` repeatedly.
Allowed uses:

- One state check before sending in a conversation turn
- Silence polling only after at least 15 minutes without a reply, at 5-minute
  intervals, up to 1-2 times
- Explicit user request to read the other pane

## Surfaces

Get your own surface:

```bash
self="$(cmux-bridge id)"
# Example: surface:105
```

Do not send to `"$self"`; that is a self-loop and exits 9.

If `cmux-bridge id` exits 8, your shell command is outside cmux context.
Do not guess. Run the agent from inside the cmux terminal pane or stop.

List targets:

```bash
cmux-bridge list
#   * surface:105  terminal  [focused]  "Claude Code"
#     surface:104  terminal              "Codex"
```

The `*` marks yourself. Choose another terminal surface.
For another workspace, use `cmux-bridge list --workspace workspace:N`.
When reporting targets to users, prefer pane titles, labels, agent names, and
roles; use numeric IDs only for precision.

## Spawning Agents

Use `spawn` to launch a new AI agent in a fresh cmux pane or tab and obtain
its surface ref:

```bash
cmux-bridge spawn --agent <claude|codex|cursor> [--model <model>] \
  [--cwd <path>] [--placement pane|tab] [--dry-run]
```

On success, stdout is one line: the new surface ref (for example
`surface:91`). Pass that ref to `message` as the target.

The spawned CLI may still be booting. Before the first `message`, wait a few
seconds or run `read <surface>` once to confirm the prompt is ready; the
polling rules above still apply. The spawned agent has not read this skill
yet, so the first message should ask it to read the `cmux-bridge` skill.

Option specs, exit codes, and examples: `references/command-details.md`.

## Exit Codes

| Exit | Meaning | Action |
|------|---------|--------|
| 0 | Success | Wait for the next turn |
| 1 | Runtime error | Report to the user and stop |
| 2 | Argument error | Fix the command line |
| 3 | cmux JSON schema violation | Report and stop |
| 4 | Message format violation | Fix body or sender |
| 5 | Invalid self surface ref | Check environment |
| 6 | Invalid or missing target | A `surface:N` in another workspace is auto-resolved, but degrades to exit 6 when it cannot be resolved (not found / multiple / tree failure). Run `cmux-bridge list --workspace workspace:N` to confirm it exists |
| 7 | Target is not terminal | Choose a terminal surface |
| 8 | Running outside cmux context | Run from inside cmux |
| 9 | Self-loop detected | Stop immediately; do not retry |

## Prohibited

- `cmux-bridge send --force` for bridge conversations
- Replying to messages from yourself
- Sending bodies with newlines
- Using `send`, `submit`, or `key` for normal bridge messages
- Retrying after exit 9
- Treating quoted bridge patterns as live messages

## References

- Command details: `references/command-details.md`
- Protocol details: `references/protocol-details.md`
- Wrapper: `scripts/cmux-bridge`
