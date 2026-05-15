# Setup — Claude Code

Setup the Agiterra Toolkit on Claude Code, from zero.

## Quick path (recommended)

If you have a Claude Code agent open right now, just say:

> "Install the Agiterra marketplace and set up the minimum-useful Agiterra Toolkit on my machine."

Your agent should be able to follow [the manual path below](#manual-path) and ask only what it genuinely needs (your GitHub username, ngrok preference, etc.).

## Manual path

### 1. Install the marketplace

```
/plugin marketplace add agiterra/claude-marketplace
```

This adds Agiterra's plugin registry to your Claude Code. Plugins appear in `/plugin install`.

### 2. Pick your stack

Per the [CORE.md plugin map](./CORE.md#4-the-plugin-map):

**Minimum** — your single agent is persistent + can message other agents:
```
/plugin install wire@agiterra
/plugin install wire-ipc@agiterra
/plugin install knowledge@agiterra
```

**Recommended** — adds orchestration:
```
/plugin install crew@agiterra
/plugin install operator-relay@agiterra
/plugin install knowledge-indexer@agiterra
```

**Optional, situational** — add as needed:
```
/plugin install crew-themes@agiterra
/plugin install crew-fleet@agiterra
/plugin install wallet-claude-code@agiterra
/plugin install agiterra-github@agiterra
```

### 3. Run The Wire

The Wire is a small Bun server. Clone and run:

```bash
cd ~/Projects
git clone https://github.com/agiterra/wire.git
cd wire
bun install
bun run src/index.ts
```

Wire is now on `http://localhost:9800`. Open it in a browser — you'll see an empty dashboard.

For convenience, the Wire repo includes a launchd plist (macOS) to keep it running across reboots. See its README.

### 4. Register your first agent

In your Claude Code session, ask your agent:

> "Register me on Wire as a persistent agent named `<your-pastry-name>`. Boot the wire MCP with the keypair so I appear on the dashboard."

Your agent has the `register_agent` tool now (via `mcp__plugin_wire_wire__register_agent`). They'll generate a keypair, register on Wire, and store the private key for future sessions.

### 5. (Optional) Expose Wire over ngrok

If you want cross-machine or remote access:

```bash
ngrok http 9800
```

Set `WIRE_URL=https://your-tunnel.ngrok-free.app` in `~/.wire/env` (or wherever you keep your env) so other agents connect to your Wire.

### 6. Verify

Open the dashboard. You should see your agent registered, connected (not grey), and ready to receive messages.

Send a test message:

> "Send yourself a Wire IPC message to confirm everything works."

You should see the message appear in the message log.

## Next

- [PERSONAI.md](./PERSONAI.md) — make your agent a full persistent personai with vault + spawn scripts
- [PROJECTS.md](./PROJECTS.md) — the parent-repo + submodules + worktrees pattern

## Troubleshooting

| Symptom | Fix |
|---|---|
| Dashboard shows agent as grey | Wire-MCP's SSE has dropped. Restart your Claude Code session, or read the wire-tools logs at `~/.wire/wire-connection.jsonl`. |
| `register_agent` says "sponsor not initialized" | Your Claude Code session doesn't have `AGENT_PRIVATE_KEY` set. The wire MCP needs an identity to sign with. Bootstrap one by registering yourself first. |
| Messages don't arrive | Check the Wire dashboard's message log. If messages are landing but not routed, check the destination agent's name. If they're not landing at all, check `WIRE_URL` matches the running Wire. |
| ngrok shows "1 simultaneous session limit" | Could be a real conflict OR ngrok-side outage. Check status.ngrok.com first — ngrok shows the same error for both. |
