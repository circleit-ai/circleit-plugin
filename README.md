# CircleIt plugin for Claude Code and Codex

Circle, box or pin anything on a web page in Chrome, type what you want changed, and press **Send**. The feedback lands in your running Claude Code session or Codex thread. The agent wakes up, looks at the annotated screenshots and the exact DOM elements you marked (selector, text, styles and source-file hints), makes the change, and reports back. You see its message in the browser and on the dashboard.

This repository is the whole agent side: one plugin that installs in both Claude Code and Codex. You need a CircleIt account at https://circleit.ai and the CircleIt Chrome extension.

## Install

### Claude Code (terminal and desktop app)

Install once. The plugin goes in at user scope, so every project and every new session has it, in the terminal and in the Claude desktop app:

```text
/plugin marketplace add circleit-ai/circleit-plugin
/plugin install circleit@circleit
```

From a shell, the same is `claude plugin marketplace add circleit-ai/circleit-plugin` and then `claude plugin install circleit@circleit`.

**Sessions you start after installing** load CircleIt as they start. **Sessions that were already open:**

- **Terminal:** run `/reload-plugins`. If Claude Code says the reload changes MCP tools, run `/reload-plugins --force` (adding CircleIt's tools makes the next message re-read the conversation once). The reload starts CircleIt's monitor too, so feedback wakes the session as it does in a new one. If `circleit_status` still says "Delivery: on your next message", the monitor didn't start: feedback then reaches Claude with your next message, or when it finishes what it is doing, until you restart the session (`claude --continue` picks the conversation back up).
- **Claude desktop app:** open sessions pick CircleIt up by themselves within a few seconds of installing. The picker shows whether each session wakes automatically or picks up on your next message.

### Codex

```sh
codex plugin marketplace add circleit-ai/circleit-plugin
codex plugin add circleit@circleit
```

Then open Codex, run `/hooks` and approve CircleIt's `SessionStart` hook. You only do this once. The hook tells CircleIt which thread and folder it is working in, so feedback can wake that thread.

### Connect once per machine

In either agent, run `/circleit:connect` or say "connect CircleIt". Your browser opens, you approve the connection, and from then on every session on that machine is connected. You don't need to restart anything.

### Allow CircleIt's tools

**Claude Code** asks before each CircleIt tool call, so feedback that arrives while you're away waits at the prompt. Allow them once, in `~/.claude/settings.json`:

```json
{
    "permissions": {
        "allow": [
            "mcp__plugin_circleit_circleit"
        ]
    }
}
```

If you already have a `permissions.allow` list, add `"mcp__plugin_circleit_circleit"` to it. Or choose "Yes, and don't ask again" the first time Claude Code asks; that covers one tool in one project. This covers CircleIt's own tools only: file edits and commands still follow your Claude Code permission mode.

**Codex**: nothing to allow. CircleIt's tools are marked non-destructive, so Codex runs them without asking.

## How feedback reaches the agent

| | Claude Code | Codex |
|---|---|---|
| Wake-up | A terminal session runs a plugin monitor, which starts with the session and on `/reload-plugins`. It prints one line per item into the session, which wakes it even when it is idle. | The MCP server queues a message on the thread with `codex queue`. An idle thread picks it up within about 10 seconds. |
| Next message | The desktop app runs no monitors, nor does a terminal session whose monitor didn't start. There the MCP server registers the session itself, about 6 seconds after it starts, and CircleIt's `UserPromptSubmit` and `Stop` hooks give new feedback to Claude with your next message, or when it finishes its current turn. `circleit_status` then says "Delivery: on your next message". | — |
| Watch loop | `/circleit:watch` loops on `circleit_wait_for_feedback`. | Ask Codex to "watch for CircleIt feedback", or use the `circleit:source-command-watch` skill. Both run the same loop. |

`claude -p` and Agent SDK apps take no part: CircleIt registers no session for them and its hooks stay silent there, so feedback waits for an interactive session. Each item reaches the agent once. When a terminal session with a monitor starts in the same folder, the monitor takes over and the next-message session ends; the extension's agent picker shows which sessions wake up automatically and which pick feedback up on your next message.

Each message leads with the project and the exact page address. The agent confirms that the address belongs to the project it is working in before it changes any code. If it doesn't, the agent replies with `needs_info` instead of guessing.

## What's inside

| Path | Purpose |
|---|---|
| `skills/circleit/SKILL.md` | How the agent handles feedback: confirm the address, study the screenshots, find the code, make the change, verify it and report back. |
| `commands/watch.md`, `commands/connect.md` | `/circleit:watch` and `/circleit:connect` in Claude Code. Codex imports them as the skills `circleit:source-command-watch` and `circleit:source-command-connect`. |
| `server/circleit.mjs` | The connector: one bundled Node file (Node 18 or newer, no `node_modules`) that serves the MCP tools, the Claude Code monitor and the Claude Code hooks (`hook prompt`, `hook stop`). |
| `.claude-plugin/` | The Claude Code manifest (MCP server, `feedback` monitor, and the `UserPromptSubmit` and `Stop` hooks) and its marketplace. |
| `.codex-plugin/plugin.json`, `codex.mcp.json` | The Codex manifest (skills, MCP server, `SessionStart` hook) and its MCP config. |
| `.agents/plugins/marketplace.json` | The Codex marketplace. |

MCP tools: `circleit_get_feedback`, `circleit_list_feedback`, `circleit_set_status`, `circleit_wait_for_feedback`, `circleit_connect` and `circleit_status`, plus `circleit_session_started`, which only the Codex hook calls.

## Configuration

Credentials live in `~/.circleit/credentials.json` (mode 0600) and every session on the machine shares them. You can override them with environment variables:

| Variable | Effect |
|---|---|
| `CIRCLEIT_API_URL` | CircleIt server URL |
| `CIRCLEIT_TOKEN` | Agent token, which replaces the credentials file |
| `CIRCLEIT_HOME` | Directory for credentials, session files and the inbox (default `~/.circleit`) |

Codex passes only an allowlist of environment variables to MCP servers. `codex.mcp.json` adds these three, plus `CODEX_HOME` and `CODEX_CLI_PATH` so that `codex queue` reaches the same Codex installation.

In a Claude Code session without a monitor, feedback the session has picked up waits in `~/.circleit/inbox/<sha1 of the folder>.json` until a hook hands it to Claude. The hooks only read and update that file: they never use the network and finish in well under a second.

Codex starts the MCP server inside the installed plugin folder, so CircleIt learns your project folder from the `SessionStart` hook, or failing that from the next tool call. Until then, `circleit_status` reports "Workspace: not known yet". `circleit_wait_for_feedback` can block for up to an hour, because `codex.mcp.json` sets `tool_timeout_sec` to 3700.

## Troubleshooting

- **Nothing arrives.** Ask the agent to run `circleit_status`. It shows the connection, the session, the project, the page addresses CircleIt associates with it, and how feedback is delivered: "Live delivery: on" (the monitor wakes the session) or "Delivery: on your next message".
- **Feedback only shows up when I type something.** That is next-message delivery: the session has no monitor (the desktop app, or a terminal session whose monitor didn't start). Restart a terminal session for automatic wake-ups, or run `/circleit:watch` to have Claude wait for feedback.
- **`node` not found.** The connector runs on the `node` found on the agent's `PATH`. If you launch Codex or Claude from the Dock, make sure Node 18 or newer is on the `PATH` that GUI apps see.
- **Claude Code wakes up but waits at a permission prompt.** Allow CircleIt's tools as above. File edits and shell commands the agent runs to make the change still follow your permission mode, so they may still ask.
- **"CircleIt can't receive feedback for this workspace" / "the server refused this machine's sign-in".** The server turned the session down: the plan's limit (402, with an upgrade link), a revoked or expired sign-in (401), or no access to the team (403). CircleIt stops retrying until you reconnect with `/circleit:connect` and checks again every 5 minutes; `circleit_status` shows the same message.
- **Sign out a machine.** `node <plugin>/server/circleit.mjs logout` revokes its token on the server and removes `~/.circleit/credentials.json`; or revoke it from the dashboard's Setup page.
- **Claude Code with `-p`, the Agent SDK, or CI.** CircleIt stays out of runs nobody watches: no session is registered and the hooks stay silent, so feedback waits for an interactive session. To have such a run pick feedback up, have it call `circleit_wait_for_feedback` (or run `/circleit:watch`).
- **Codex thread doesn't wake.** Check that the `SessionStart` hook is approved in `/hooks`. Until it is, CircleIt can't wake the thread. It starts listening only after the agent first calls a CircleIt tool, and feedback waits in the dashboard until then.
