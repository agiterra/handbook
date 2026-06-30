# Setup — Claude Code

Setup the Agiterra Multi-Agent Toolkit on Claude Code, from zero.

## Quick path (recommended)

If you have a Claude Code agent open right now, just say:

> "Install the Agiterra marketplace and set up the minimum-useful Agiterra Multi-Agent Toolkit on my machine."

Your agent should be able to follow [the manual path below](#manual-path) and ask only what it genuinely needs (a pastry name for your first agent, whether you want ngrok, which optional plugins).

One thing your agent **cannot** do for you: register the *first* permanent agent. That step needs **operator** auth (you, the dashboard owner). See [step 4](#4-bootstrap-your-first-agent-operator).

## Manual path

### 1. Install the marketplace

```
/plugin marketplace add agiterra/claude-marketplace
```

This adds Agiterra's plugin registry to your Claude Code. Plugins then appear in `/plugin install`.

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
/plugin install bridge@agiterra
/plugin install crew@agiterra
/plugin install knowledge-indexer@agiterra
```

> `bridge` is the orchestrator's plugin — it collapses the register → assemble-env → launch → place-pane → attach → kick-off dance into single composite calls (`spawn`, `handoff`, `close`, …) and builds on `crew`.
> `operator-relay@agiterra` is **worker-side**, not an orchestrator plugin — it runs on the ephemeral agents you spawn (a UserPromptSubmit hook that relays operator prompts up to their manager). It loads automatically from your workers' spawn-root plugin set; you don't install it into your own stack. See the [plugin map](./CORE.md#4-the-plugin-map).

**Optional, situational** — add as needed:
```
/plugin install crew-themes@agiterra
/plugin install crew-fleet@agiterra
/plugin install github@agiterra
/plugin install slack@agiterra
/plugin install wallet-claude-code@agiterra
```

### 3. Run The Wire

The Wire ships as a standalone, checksum-verified binary with a one-command installer (no clone, no `bun install`):

```bash
curl -fsSL https://raw.githubusercontent.com/agiterra/wire/main/scripts/install.sh | bash
```

This detects your platform, fetches and verifies the matching release binary, installs it to `~/.wire/bin/wire`, writes a default config at `~/.wire/.env`, and installs a service that keeps Wire running across reboots:

- **macOS**: `~/Library/LaunchAgents/com.wire.gateway.plist` (launchd)
- **Linux**: `~/.config/systemd/user/wire.service` (systemd --user)

Config lives at `~/.wire/.env`, data at `~/.wire/wire.db`, logs at `~/.wire/wire.log`. (Full details, version pinning, and manual install are in the Wire repo's `INSTALL.md`.)

Verify it's up:

```bash
curl -fsS http://localhost:9800/health
# {"status":"ok","ts":...}
```

Wire is now on `http://localhost:9800`. Open it in a browser — you'll land on the login screen (the first passkey claims ownership; see the next step).

### 4. Bootstrap your first agent (operator)

This is the step that trips people up. A **new permanent agent** (a personai) can only be registered with **operator** authority — the dashboard owner. An agent's own `register_agent` tool signs with *its own* key, so it cannot conjure agent #1 out of nothing. The chicken needs an egg.

So you, the operator, bootstrap the first personai. Two ways:

**A. Via the dashboard (recommended).**
1. Open `http://localhost:9800`. The first time, the login page says "Claim this instance" — register a passkey (WebAuthn). The first passkey to claim becomes the owner.
2. Once signed in, use the dashboard's **register form** to add your first agent: an id (your pastry name, lowercase), a display name, and its Ed25519 public key. The dashboard registers it as a **permanent** agent.

You'll need a keypair for that pubkey. Generate one and stash the private key where your spawn script will read it (see [PERSONAI.md](./PERSONAI.md), which wires `AGENT_ID` / `AGENT_PRIVATE_KEY` into the agent's environment).

**B. Via the dashboard token (headless / scripted).**
Set `WIRE_DASHBOARD_TOKEN=<secret>` in `~/.wire/.env` and reload the service, then:

```bash
curl -X POST "http://localhost:9800/agents/register?token=$WIRE_DASHBOARD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"id":"<your-pastry-name>","display_name":"<Your Display Name>","pubkey":"<base64-ed25519-pubkey>","permanent":true}'
```

The operator token (or an operator session) is what satisfies the server's `new-permanent` auth gate.

Once that first personai is registered and its spawn script exports `AGENT_PRIVATE_KEY`, **it** can sponsor any further agents — including ephemeral workers — using its own `register_agent` tool. (Only permanent agents may sponsor; an ephemeral cannot spawn another ephemeral on the Wire — ephemerals use the runtime's own subagents for parallel work, which never touch the Wire.)

> The wire and wire-ipc plugins read `AGENT_ID` and `AGENT_PRIVATE_KEY` from the session environment to sign and connect. The keypair from this step must be exported when you launch the agent — the personai spawn script does this for you (see [PERSONAI.md](./PERSONAI.md)).

### 5. (Optional) Expose Wire over ngrok

If you want cross-machine or remote access:

```bash
ngrok http 9800
```

- For **agents connecting in** from elsewhere, set `WIRE_URL=https://your-tunnel.ngrok-free.app` in their environment (e.g. their spawn script or `~/.wire/.env`). `WIRE_URL` is a *client* setting — it tells a connecting agent where the broker is; it has no effect on the running server.
- For the **dashboard + WebAuthn login** to work over the tunnel, the *server* needs to know its public hostname: set `WIRE_RP_ID` (and, if needed, `WIRE_ORIGIN`) to your tunnel host in `~/.wire/.env` and reload the service. A passkey is bound to one domain, so the RP ID must match the host you actually log in from.

### 6. Verify

Open the dashboard. You should see your agent registered, connected (not grey), and ready to receive messages.

Send a test message:

> "Send yourself a Wire IPC message to confirm everything works."

You should see the message appear in the message log.

## Next

- [PERSONAI.md](./PERSONAI.md) — make your agent a full persistent personai with vault + spawn scripts (and the keypair wiring referenced in step 4)
- [PROJECTS.md](./PROJECTS.md) — the parent-repo + submodules + worktrees pattern

## Troubleshooting

| Symptom | Fix |
|---|---|
| Dashboard shows agent as grey | Wire-MCP's SSE has dropped. Restart your Claude Code session (or `/plugin` reload the wire plugin), or read the wire-tools logs at `~/.wire/wire-connection.jsonl`. |
| Registering the first agent fails with "operator authentication required" (401) | A new **permanent** agent needs operator auth. Claim the dashboard with a passkey, or set `WIRE_DASHBOARD_TOKEN` and pass `?token=` (see [step 4](#4-bootstrap-your-first-agent-operator)). `register_agent` from an unregistered agent will not work for agent #1. |
| `register_agent` says "sponsor not initialized" | The wire MCP has no `AGENT_PRIVATE_KEY` in its environment, so it has no identity to sign with. For an existing sponsor, ensure the spawn env exports it. For the very first agent, you can't sponsor at all yet — bootstrap via the operator path in step 4. |
| `agent_exists_pubkey_mismatch` (HTTP 409) on register | The id already exists with a different key. Don't silently rotate a live agent's keypair. Only pass `force_rotate: true` once you've confirmed no live process still holds the old key. |
| Messages don't arrive | Check the Wire dashboard's message log. If messages land but aren't routed, check the destination agent's id. If they don't land at all, check `WIRE_URL` matches the running Wire. |
| ngrok shows "1 simultaneous session limit" | Could be a real conflict OR an ngrok-side outage. Check status.ngrok.com first — ngrok shows the same error for both. |
