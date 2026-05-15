# Setup — Codex

Setup the Agiterra Multi-Agent Toolkit on OpenAI Codex CLI, from zero.

Codex has no plugin marketplace, so MCP servers are wired via `~/.codex/config.toml` (or a per-spawn config — see below).

## Quick path

If you have a codex agent open, say:

> "Set up the Agiterra MCP servers in my codex config and confirm I can connect to Wire."

Your agent should be able to follow [the manual path below](#manual-path) and ask only what it genuinely needs.

## Manual path

### 1. Run The Wire

The Wire server itself is runtime-agnostic — same setup as Claude Code:

```bash
cd ~/Projects
git clone https://github.com/agiterra/wire.git
cd wire
bun install
bun run src/index.ts
```

Wire is now on `http://localhost:9800`.

### 2. Add MCP server entries to `~/.codex/config.toml`

Codex MCP entries reference repo paths and run via `tsx` (codex's `rmcp` client hangs on `bun`-spawned stdio children, but `tsx` handshakes cleanly).

Add the Toolkit MCPs to `~/.codex/config.toml`:

```toml
[features]
rmcp_client = true

[mcp_servers.wire]
command = "tsx"
args = ["/Users/<you>/Projects/Agiterra/wire-codex/server.ts"]
startup_timeout_sec = 30

[mcp_servers.wire.env]
AGENT_ID = "<your-agent-id>"
AGENT_NAME = "<Your Display Name>"
AGENT_PRIVATE_KEY = "<base64-PKCS8-private-key>"
WIRE_URL = "https://the-wire.ngrok.io"   # or http://localhost:9800
# Forward terminal-multiplexer env vars so wire-tools' push-delivery
# transport can detect the multiplexer it's running inside. Codex
# sanitizes the MCP child env to only what's listed here — $STY etc.
# from the spawning shell will NOT propagate without explicit forwarding.
# Without these, codex agents fall back to poll-mode
# (`get_pending_messages`) instead of getting Wire messages pushed into
# their screen as new turns.
STY = "${STY:-}"
TMUX = "${TMUX:-}"
TMUX_PANE = "${TMUX_PANE:-}"
CMUX_SURFACE_ID = "${CMUX_SURFACE_ID:-}"
TERM_PROGRAM = "${TERM_PROGRAM:-}"
ITERM_SESSION_ID = "${ITERM_SESSION_ID:-}"

[mcp_servers.wire-ipc]
command = "tsx"
args = ["/Users/<you>/Projects/Agiterra/wire-ipc-codex/server.ts"]
startup_timeout_sec = 30

[mcp_servers.wire-ipc.env]
AGENT_ID = "<your-agent-id>"
AGENT_NAME = "<Your Display Name>"
AGENT_PRIVATE_KEY = "<base64-PKCS8-private-key>"
WIRE_URL = "https://the-wire.ngrok.io"
```

Persistent entries in `config.toml` matter — codex's `tool_search` only indexes tools from persistent entries, not from `-c` overrides.

### 3. Clone the codex adapters

```bash
cd ~/Projects/Agiterra
git clone https://github.com/agiterra/wire-codex.git
git clone https://github.com/agiterra/wire-ipc-codex.git
cd wire-codex && bun install
cd ../wire-ipc-codex && bun install
```

### 4. Generate a keypair

Codex doesn't have a CC-equivalent of `/plugin install` — you bootstrap your keypair manually:

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

Paste the `AGENT_PRIVATE_KEY` into your `config.toml` env block above. Hold onto the pubkey — you'll need it to register on Wire.

### 5. Register on Wire

A one-time `curl` to register your keypair on Wire:

```bash
curl -X POST http://localhost:9800/agents/register \
  -H "Content-Type: application/json" \
  -d '{"id":"<your-agent-id>","display_name":"<Your Display Name>","pubkey":"<base64-pubkey>"}'
```

Or, if you already have a Claude Code personai running, ask them to register you:

> "Register a new codex agent named `<your-agent-id>` with pubkey `<base64-pubkey>`."

### 6. Launch codex

```bash
codex --yolo
```

Wire MCP will boot, connect to the broker, and your agent will appear on the dashboard.

## Spawning codex engineers from a parent agent

If you're running a parent personai (in Claude Code or codex) that spawns codex engineers, use `~/.wire/codex-launch.sh` — a per-spawn launcher that:

- Creates an isolated `CODEX_HOME` so your personal `~/.codex` isn't polluted
- Generates a per-spawn `config.toml` with only the Agiterra MCPs enabled
- Symlinks your auth.json so the spawned agent reuses your API credentials

See [agiterra/wire/scripts/codex-launch.sh](https://github.com/agiterra/wire) for the canonical version. Crew calls this script when spawning codex engineers.

## Next

- [PERSONAI.md](./PERSONAI.md) — make your agent a full persistent personai
- [PROJECTS.md](./PROJECTS.md) — multi-repo project pattern

## Codex-specific gotchas

| Gotcha | Why |
|---|---|
| Use `tsx`, not `bun`, in MCP `command` | Codex's `rmcp` client hangs on `bun`-spawned stdio children |
| Use `--sandbox danger-full-access` for spawned codex engineers | Default sandbox blocks outbound HTTP, breaks SSE to Wire |
| Persistent `config.toml` only for tool_search | `-c mcp_servers.*.command=...` overrides work but tools won't be discoverable via `tool_search` |
| `operator-relay` MCP loads but its tools are a no-op | CC's `UserPromptSubmit` hook has no codex equivalent. Document directives manually via `wire-ipc.send_message`. |
| Codex agents are poll-mode on Wire | They must call `get_pending_messages` to drain (a real-time push delivery is in flight — see [wire-tools#9](https://github.com/agiterra/wire-tools/issues/9)) |
