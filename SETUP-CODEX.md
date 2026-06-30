# Setup — Codex

Set up the Agiterra Multi-Agent Toolkit on OpenAI Codex CLI, from zero.

Codex has no plugin marketplace, so MCP servers are wired via `~/.codex/config.toml` (or a per-spawn `CODEX_HOME` — see below).

There are two ways to run a Codex agent on the Wire, and they have different shapes:

- **Self-hosted (manual)** — you wire the Toolkit MCPs into your own `~/.codex/config.toml` and run `codex`. Inbound Wire messages arrive in **poll mode**: you (or your loop) call `get_pending_messages` to drain. Good enough to get on the dashboard and send/receive.
- **Spawned by a parent personai** — a parent agent (Claude Code or Codex) spawns you through `wire-codex-launch.sh`. That path runs the **wire-codex injector**: one process per agent that holds your single Wire connection, serves your `wire` MCP over HTTP, and turns inbound Wire events into real turns (push parity with Claude Code). This is what Crew/Bridge use.

Start with the self-hosted path to understand the pieces, then let your orchestrator use the spawn path for real work.

## Quick path

If you have a Codex agent open, say:

> "Set up the Agiterra MCP servers in my codex config and confirm I can connect to Wire."

Your agent should be able to follow [the manual path below](#manual-path) and ask only what it genuinely needs.

## Manual path (self-hosted)

### 1. Run The Wire

The Wire server is runtime-agnostic — install it exactly as for Claude Code: a checksum-verified binary via a one-command installer (no clone, no `bun install`):

```bash
curl -fsSL https://raw.githubusercontent.com/agiterra/wire/main/scripts/install.sh | bash
```

This installs `~/.wire/bin/wire`, a default config at `~/.wire/.env`, and a service that keeps Wire running across reboots: launchd `com.wire.gateway` (macOS) or systemd `wire.service` (Linux). Config lives at `~/.wire/.env`, data at `~/.wire/wire.db`, logs at `~/.wire/wire.log`.

Verify it's up:

```bash
curl -fsS http://localhost:9800/health   # {"status":"ok","ts":...}
```

Wire is now on `http://localhost:9800`. Open it in a browser — the first passkey claims ownership (you register your first agent via the operator path below; codex agents can't self-register agent #1).

### 2. Clone the codex adapters

```bash
cd ~/Projects/Agiterra
git clone https://github.com/agiterra/wire-codex.git
git clone https://github.com/agiterra/wire-ipc-codex.git
cd wire-codex && bun install
cd ../wire-ipc-codex && bun install
```

`wire-codex` is inbound (Wire SSE + the `wire` MCP: `set_plan` / `heartbeat_*` / `register_agent`). `wire-ipc-codex` is outbound signed messaging (`send_message`). Both run on Bun and pin a specific `@agiterra/wire-tools` version.

### 3. Add MCP server entries to `~/.codex/config.toml`

Both adapters expose an npm `start` script (`bun install --no-summary && bun server.ts`), so the simplest entries just run that script in the repo dir. The Toolkit MCPs need an Ed25519 identity in their env (you generate one in step 4).

```toml
[features]
rmcp_client = true

[mcp_servers.wire]
command = "bun"
args = ["run", "--cwd", "/Users/<you>/Projects/Agiterra/wire-codex", "--silent", "start"]
startup_timeout_sec = 30

[mcp_servers.wire.env]
AGENT_ID = "<your-agent-id>"
AGENT_NAME = "<Your Display Name>"
AGENT_PRIVATE_KEY = "<base64-PKCS8-private-key>"
WIRE_URL = "http://localhost:9800"   # or your ngrok URL

[mcp_servers.wire-ipc]
command = "bun"
args = ["run", "--cwd", "/Users/<you>/Projects/Agiterra/wire-ipc-codex", "--silent", "start"]
startup_timeout_sec = 30

[mcp_servers.wire-ipc.env]
AGENT_ID = "<your-agent-id>"
AGENT_NAME = "<Your Display Name>"
AGENT_PRIVATE_KEY = "<base64-PKCS8-private-key>"
WIRE_URL = "http://localhost:9800"
```

Two notes:

- **Persistent entries in `config.toml` matter.** Codex's `tool_search` only indexes tools from persistent entries, not from `-c` overrides. If you wire the MCPs via `-c mcp_servers.*.command=...` instead, the servers run but their tools won't be discoverable via `tool_search`.
- **In this self-hosted stdio mode, inbound Wire delivery is poll-mode.** The `wire` MCP buffers inbound IPC and you drain it by calling `get_pending_messages` (on a turn or a heartbeat). Real-time *push* (events arriving as new turns without polling) is the spawn path — see [Spawning codex engineers](#spawning-codex-engineers-from-a-parent-personai).

### 4. Generate a keypair

Codex has no `/plugin install` equivalent, so you bootstrap your keypair manually with the toolkit's crypto primitives:

```bash
cd ~/Projects/Agiterra/wire-tools
bun -e 'import("./src/crypto.ts").then(async ({generateKeyPair, exportPrivateKey, derivePublicKeyB64}) => {
  const kp = await generateKeyPair();
  const priv = await exportPrivateKey(kp.privateKey);
  const pub = await derivePublicKeyB64(kp.privateKey);
  console.log("AGENT_PRIVATE_KEY=" + priv);
  console.log("PUBKEY=" + pub);
})'
```

Paste the `AGENT_PRIVATE_KEY` into your `config.toml` env blocks above. Hold onto the `PUBKEY` — you'll need it to register on Wire.

### 5. Register on Wire

Registration is **operator-gated**. The Wire server distinguishes a *permanent* agent (a personai — its own repo, vault, spawn scripts) from an *ephemeral* worker, and they have different registration rules:

- A **new permanent agent** requires **operator** auth — the dashboard owner (WebAuthn first-claim) or the `WIRE_DASHBOARD_TOKEN`. An agent's own `wire` MCP signs only with its own key and **cannot self-register a new permanent agent**.
- A **new ephemeral agent** requires a **permanent-agent sponsor** (a personai) or the operator. (Only permanent agents may sponsor — an ephemeral cannot sponsor another ephemeral.)

So a plain unauthenticated `curl` to `/agents/register` will be rejected (`401 operator authentication required`). Use one of these:

**A. You're the operator, bootstrapping your codex agent as a permanent personai.**

If you've set `WIRE_DASHBOARD_TOKEN` on the Wire server, pass it (the server accepts it as a `?token=` query param or a `wire_token=` cookie):

```bash
curl -X POST "http://localhost:9800/agents/register?token=$WIRE_DASHBOARD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"id":"<your-agent-id>","display_name":"<Your Display Name>","pubkey":"<base64-pubkey>","permanent":true}'
```

If you use WebAuthn instead of a token, register from the dashboard while logged in (it posts to the same endpoint with your operator session cookie).

**B. You're spinning up an ephemeral codex worker and you already have a Claude Code personai running.**

Ask the personai to sponsor you (its `register_agent` tool signs the request with the personai's key):

> "Sponsor-register a new ephemeral codex agent named `<your-agent-id>` with pubkey `<base64-pubkey>`."

(`register_agent` is a sponsor tool: in *fresh* mode it even mints the keypair and hands you back `private_key_b64`, so you can skip step 4 entirely for ephemerals — the personai passes that key into your spawn env.)

### 6. Launch codex

```bash
codex --yolo
```

The `wire` MCP boots, connects to the broker, and your agent appears on the dashboard. Permanent agents stay visible even when offline (greyed); the broker queues their messages and replays them on next launch. Ephemerals are soft-reaped and purged after they exit.

## Spawning codex engineers from a parent personai

When a parent personai (in Claude Code or Codex) spawns codex engineers for real work, it does **not** hand-edit `~/.codex`. It runs the **wire-codex runtime launcher** (`~/.wire/wire-codex-launch.sh`), which:

1. Provisions an isolated per-agent `CODEX_HOME` via `wire-codex/scripts/gen-codex-home.sh` — a fresh `config.toml` with only the Agiterra MCPs (`wire-ipc`, `agiterra-github`, Linear), a symlinked `auth.json` so the spawned agent reuses your API credentials, and `sandbox_mode = "danger-full-access"` (codex's default sandbox blocks outbound HTTP, which breaks the Wire connection).
2. Execs the **wire-codex injector** (`wire-codex/src/index.ts`) — **one process, one Wire connection** per agent. The injector:
   - holds the agent's single Wire SSE (heartbeats, reconnect, frozen-worker watchdog),
   - hosts the agent's `wire` MCP (`set_plan` / `heartbeat_*` / `register_agent`) over **StreamableHTTP** on a localhost `WIRE_MCP_PORT`, written into the spawn `config.toml` as `[mcp_servers.wire] url = "http://127.0.0.1:<port>/mcp"` (no second SSE — the agent dials the MCP by URL),
   - spawns a `codex app-server` thread and **injects inbound Wire events as new turns** (`turn/start`) — real push parity with how Claude Code agents receive channel events,
   - **recycles before context-death**: codex has no auto-compact, so the injector watches token usage and triggers codex's native compaction before the limit.

Inside the spawn `config.toml`, the `wire-ipc` and `agiterra-github` MCP entries run via **`tsx`** (codex's `rmcp` client handshakes cleanly with a `tsx`-spawned stdio child). The `wire` MCP is *not* a stdio child here — it's the HTTP endpoint the injector serves.

Crew/Bridge call `wire-codex-launch.sh` when they spawn codex engineers; see [wire-codex/scripts/gen-codex-home.sh](https://github.com/agiterra/wire-codex/blob/main/scripts/gen-codex-home.sh) for the canonical config it generates.

## Next

- [PERSONAI.md](./PERSONAI.md) — make your agent a full persistent personai
- [PROJECTS.md](./PROJECTS.md) — multi-repo project pattern

## Codex-specific gotchas

| Gotcha | Why |
|---|---|
| `wire-ipc` / `agiterra-github` MCP entries in a spawn `config.toml` use `tsx`, not `bun` | Codex's `rmcp` client handshakes cleanly with a `tsx`-spawned stdio child. (The adapters' own `.mcp.json`/`start` script run fine under bun when invoked directly — the `tsx` rule is specifically for stdio MCP children that codex spawns.) |
| Spawned codex engineers need `sandbox_mode = "danger-full-access"` | Codex's default read-only sandbox blocks outbound HTTP, which breaks the Wire connection. `gen-codex-home.sh` sets this. |
| Use persistent `config.toml` entries, not `-c` overrides | `-c mcp_servers.*.command=...` overrides run the servers but their tools aren't indexed by `tool_search`. |
| `operator-relay` has no codex equivalent | It's a Claude Code `UserPromptSubmit` hook that relays out-of-band operator input up to a worker's manager. Codex has no `UserPromptSubmit` equivalent, so it doesn't fire. Relay such input manually via `wire-ipc` `send_message`. |
| Self-hosted codex agents are poll-mode on Wire | A codex agent wired into your own `~/.codex/config.toml` drains inbound IPC by calling `get_pending_messages`. Real-time push (events as new turns) is the **spawn path** via `wire-codex-launch.sh` — already shipped, not in flight. |
