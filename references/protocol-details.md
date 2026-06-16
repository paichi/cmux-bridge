# cmux-bridge Protocol Details

This file contains protocol rationale, detailed formats, and examples that are too detailed for `SKILL.md`.
The core conversation, closing, and polling rules remain in `SKILL.md`.

## Slash Command Details

Core rule: when sending CLI slash commands such as `/clear`, use the dedicated `clear` subcommand instead of `message`.

```bash
cmux-bridge clear [--workspace <workspace>] [--force] <surface>
```

### Why `/clear` Has a Dedicated Subcommand

`message` always prepends the bridge header:

```text
[cmux-bridge from:... reply-to:...] <body>
```

If you run `cmux-bridge message <target> "/clear"`, the target prompt receives text starting with `[`, not `/`.
Claude Code and Codex CLI detect slash commands only when `/` is the first character, so `/clear` would be treated as ordinary text.

`clear` sends raw `/clear` without the bridge header, then sends an Enter key event.

### Constraints

- The body is fixed to `/clear`; arbitrary text is not accepted.
- Existing safety checks still apply: terminal validation, self-loop rejection, and target character validation.
- `--force` bypasses only surface type validation. It never bypasses self-loop rejection.

### Examples

```bash
# Clear a Claude Code pane in the same workspace.
cmux-bridge clear surface:104

# A pane in another workspace can be cleared without --workspace (auto-resolved).
cmux-bridge clear surface:33
# Explicit --workspace still works and takes precedence:
cmux-bridge clear --workspace workspace:11 surface:33
```

## First Message to a New Peer

Core rule: in the first message to a peer that may not have loaded the skill, include a note asking it to read the `cmux-bridge` skill.

Reason: without the skill, the peer may not recognize the bridge header or speech-act protocol.
The first message should make it clear that this is a cmux-bridge conversation.

### Recommended Format

```bash
cmux-bridge message <target> "[act:inform id:hello-1] This is my first message to you through cmux-bridge. Please read the cmux-bridge SKILL.md in your installed skills directory before replying. When ready, reply with [act:react id:hello-1]."
```

Include:

- A short note that this is the first bridge message
- A request to read the installed `cmux-bridge` skill
- The expected reply format, such as `[act:react id:hello-1]`

### When You Can Skip It

- The same peer has recently participated in a speech-act exchange.
- You are replying to a bridge message, which means the peer is already using the skill.

## Polling Details

Core rule: do not call `cmux-bridge read <target>` repeatedly in a short period.
Only use silence polling after at least 15 minutes without a reply, at 5-minute intervals, up to 1-2 times.

Reason: `read` pulls the target pane's screen buffer into context. AI TUIs often update spinner or progress lines, which can create large context costs with little new information.
Short polling usually wastes tokens and does not reliably detect whether the other agent is still thinking.

### Allowed Uses of `read`

| Use | Frequency |
|-----|-----------|
| Check the target state before sending | At most once per conversation turn |
| Silence polling | After 15+ minutes without a reply, every 5 minutes, up to 1-2 times |
| Explicit user request to read the other pane | Per user request |

### Prohibited Polling Patterns

- Send -> read immediately -> wait 5 seconds -> read again.
- Read every 10 seconds to watch progress.
- Read "just in case" after only a few minutes.
- Continue polling after 1-2 silence checks.

### What Counts as Silence

Silence means no response such as `[act:react]`, `[act:think]`, or `[act:inform]` for at least 15 minutes.
Only then use `read` for a liveness check.

If `read` shows a response, wait or continue from that response.
If it shows nothing relevant, retry once with the same `id`.
If silence continues, do one more `read`.
If there is still no response, assume the target pane may be gone, busy elsewhere, or stuck with unsent input, and stop or report the state to the user.

For urgent checks, the 15-minute threshold may be shortened, but still wait at least a few minutes and keep the 1-2 polling limit.

## User-Facing Target Descriptions

Core rule: when reporting a workspace or surface to the user, prefer pane names, titles, agent names, and roles over numeric IDs.
Use numeric IDs only when they add precision.

Good examples:

- "the Claude pane titled `cmux-bridge review session`"
- "the Codex pane named `cross-agent-conversation` in the same `cmux list` output"
- "the Claude pane titled `Review backend changes` in the `example-project` workspace (`surface:33`)"

Avoid:

- "send it to `surface:118`"
- "send it inside `workspace:24`"

Why: numeric IDs are hard for humans to distinguish. Human-readable pane context reduces misrouting.

When reporting before or after a send, describe in this order:

1. Pane name or title
2. Relationship, such as same `cmux list` output or another workspace
3. Optional `surface:N` in parentheses

## Message Splitting (n/m Part Markers)

Core rule: when one logical message is split across several `message` calls, tag
each part with `(n/m)` right after the act/id header (`n` = current part, `m` =
total parts). Tag only when m≥2, share one `id`, and send in order n=1→m. The
receiver holds its reply until `(m/m)` arrives.

### Why `id` alone is not enough

`id` groups one topic; it cannot express how many parts a single logical message
has or which one this is. Without a marker the receiver may reply at part 1 of 3,
an early-reply race. `(n/m)` adds an ordering/completeness axis orthogonal to `id`.

### Format and example

Each part repeats `[act id]` and places `(n/m)` right after it (act/id stays the
leading token by convention).

```text
[cmux-bridge from:claude-code reply-to:surface:105] [act:inform id:plan-1] (1/3) First, settle the approach.
[cmux-bridge from:claude-code reply-to:surface:105] [act:inform id:plan-1] (2/3) Next, write the tests.
[cmux-bridge from:claude-code reply-to:surface:105] [act:inform id:plan-1] (3/3) Finally, run review.
```

### Receiver state machine

```text
Idle ──(single message: no marker / (1/1))──▶ reply, back to Idle
Idle ──((n/m) received, m≥2)──▶ Collecting (buffer the part)
Collecting ──(next part n<m)──▶ Collecting (append)
Collecting ──((m/m) received)──▶ Complete ──(concatenate, reply once)──▶ Idle
Collecting ──(silence)──▶ degrade to the retry / exit-code path (Idle)
```

### Missing part (no clarify)

If `(m/m)` never arrives, do not invent a dedicated notice (clarify). Degrade to
the existing silence and retry rules (state check before sending, retry 1-2 times
with the same `id`). Judge on what you have, or split out the peer's state via the
exit codes. The exit from waiting is a timeout, not a new message type.

## Example: Receive, Extract, Reply

Situation: this line appears in the terminal:

```text
[cmux-bridge from:codex reply-to:surface:104] [act:ask id:fib-1] What is a one-line Python Fibonacci implementation?
```

Action sequence:

```bash
# 1. Get your own surface if needed for self-loop awareness.
self="$(cmux-bridge id)"

# 2. Use reply-to exactly as received.
target="surface:104"

# 3. Build a one-line body, quote it, and send.
cmux-bridge message "$target" "[act:inform id:fib-1] lambda n: n if n < 2 else fib(n - 1) + fib(n - 2)"
```

On exit 0, wait for the next response.
If the user asks you to read recent output, use `cmux-bridge read "$self" 50` and search for the bridge pattern again.
