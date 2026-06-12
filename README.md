# cmux-bridge

`cmux-bridge` is for AI agents that are running inside cmux terminal panes.

It lets those cmux-hosted agents talk to each other. Each participating agent must have its own cmux terminal pane, and that agent's shell commands must execute from the same pane.

If an agent runs outside cmux, or if its shell commands run outside its cmux pane, it is not a reliable bridge participant.

For example, you can work in Claude Code while asking a nearby Codex pane for review, or orchestrate multiple agents with separate roles for implementation, review, research, and planning.

It does not use a dedicated agent API. It connects agents by sending text into cmux terminal panes.

## Install

For the agent you are currently using, this is usually enough:

```bash
npx skills add paichi/cmux-bridge -g
```

To install it for every agent supported by your `npx skills` installation:

```bash
npx skills add paichi/cmux-bridge -a '*' -g -y
```

## Requirements

- cmux 0.63.2 or newer
- bash 3.x or newer
- python3
- One cmux terminal pane for each participating AI agent
- Each participating agent must be running inside its cmux terminal pane
- Each participating agent must be able to load skills
- Each participating agent must be able to run shell commands from that same cmux pane

The important constraint is where the agent runs. The agent process and its shell commands need to be inside the cmux terminal pane that should be treated as the sender.

`cmux-bridge` identifies the sender pane from the cmux execution context. If the agent is outside cmux, the bridge cannot reliably know where replies should go, so two-way conversation is not reliable.

## Usage

Open cmux, create one terminal pane per agent, then start each AI agent inside its own pane.

Do not start the agent outside cmux and use cmux only as a viewer. The agent itself needs to be running in the pane.

After that, ask an agent in normal language:

```text
Use cmux-bridge to work through this with the nearby Codex pane.
```

```text
Ask the review Claude Code pane to review this implementation plan.
```

```text
Ask the other Codex pane to investigate the cause of this error.
```

```text
Ask the `Review Codex` pane for an opinion before deciding.
```

If the target pane is ambiguous, the agent cannot safely choose where to send the message. Usually, specify it by nearby agent name, pane title, agent name, or role name.

For a more precise target, right-click the cmux tab, choose **Copy ID**, and paste the copied ID into your request.

```text
Ask Codex at `surface:104` to review this change.
```

### Spawning Agents with a Specific Model

Instead of opening every pane yourself, an agent can launch a new agent with
an explicit model and start talking to it:

```text
Spawn a Claude Code pane on haiku and have it draft the unit tests.
```

```text
Launch cursor-agent with the composer model in a new pane and review this diff together.
```

Under the hood this uses the `spawn` subcommand, which creates a new pane (or
tab), launches the chosen CLI with the requested model, and prints the new
surface ref to message:

```bash
new="$(cmux-bridge spawn --agent claude --model haiku)"
sleep 3   # the CLI is still booting; wait briefly (or read once)
cmux-bridge message "$new" "Please read the cmux-bridge skill. [act:ask id:spawn-1] Ready to pair?"
```

## Multi-Agent Orchestration

`cmux-bridge` can be used not only for one-off questions, but also for multi-agent orchestration.

Here, orchestration means assigning roles to multiple agents and connecting their planning, implementation, review, and research through conversation.

```text
Use cmux-bridge to ask the `Implement Codex` pane for an implementation plan, then review it in the `Review Claude` pane.
```

```text
Ask the `Test Planner Codex` pane for a test strategy, then implement based on that result.
```

```text
Use the `Implement Claude` pane for implementation and the `Review Codex` pane for review.
```

```text
Use the `Design Claude` pane for design and the `Research Codex` pane for investigation.
```

When orchestrating multiple agents, specify which pane has which role.

## Conversation Protocol

`cmux-bridge` uses a small conversation protocol so agents do not lose track of who should reply, or keep replying forever.

You do not need to write this protocol by hand. Agents use it through the skill.

Messages include a sender and a reply target:

```text
[cmux-bridge from:<sender> reply-to:<surface:N>] <body>
```

When useful, the message body starts with an act and an id:

```text
[act:ask id:review-1] Please review this change.
```

Common `act` values:

- `ask`: ask a question
- `inform`: share information
- `propose`: propose a plan or option
- `react`: agree, disagree, or react
- `clarify`: ask for clarification
- `think`: acknowledge and say the agent is thinking
- `decide`: make a decision
- `close`: close the topic

Use the same `id` for the same topic. This helps agents keep parallel conversations separate.

If an agent replies with `think`, it is still working. Do not keep sending more messages on the same topic.

If an agent replies with `close`, the topic is finished. Do not reply with "ok" or "thanks" to `close`.

## Troubleshooting

If messages do not arrive, check that the target agent is running in a cmux terminal pane.

If replies do not work, the agent's shell commands may be running outside cmux. Start the agent itself inside a cmux terminal pane.

Browser panes and other non-terminal panes cannot receive bridge messages.

If a long message lands in the target's input box but is not submitted, the
receiving TUI swallowed the Enter while ingesting the pasted text. The wrapper
already sends a delayed double-enter to avoid this; if it still happens on a
slow machine, raise the delay with `export CMUX_BRIDGE_COMMIT_DELAY=0.8`
(seconds; default `0.4`).

## Included Files

```text
SKILL.md
agents/openai.yaml
references/
scripts/cmux-bridge
```

`SKILL.md` contains the instructions for agents. `agents/openai.yaml` contains OpenAI/Codex UI metadata. `scripts/cmux-bridge` is the wrapper that sends messages through cmux.

## License

MIT
