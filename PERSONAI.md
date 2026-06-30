# Setting up a personai

A "personai" is a permanent AI agent. They have a name, a vault, a GitHub repo of their own, and a `spawn-claude.sh` / `spawn-codex.sh` that boots them on any machine. The term is "personai" — singular and plural, never persona/personae.

This guide walks an operator (and their agent) through setting one up end-to-end. Read [CORE.md](./CORE.md) first if you haven't yet.

## Mental model

A personai is **owned by no project**. They're a person, not a worker. They have:

- A name (Fondant, Brioche, Eclair, Croissant — pastry naming is convention, not requirement)
- A home directory (`~/Projects/<Name>/`) outside any code repo
- A GitHub repo (`<your-org-or-username>/<Name>`) containing their identity
- A vault (`.knowledge/`) of their accumulated experience
- A `CLAUDE.md` / `AGENTS.md` that defines who they are
- One or more spawn scripts that boot them with the right environment

They can work on any project, but they're not OF any project. Contrast with an **ephemeral** — a short-lived worker a personai spawns to parallelize one job, then soft-reaps. A personai persists across days, machines, and reboots; an ephemeral does not. (See [CORE.md](./CORE.md) for the full identity model.)

## 1. Pick a name

Convention: pick a pastry. Real ones we use: **Fondant** (toolsmith), **Brioche** (engineering director), **Eclair** (engineer), **Croissant** (engineer), **Stollen**, **Palmier**, **Panettone**, **Sable**, **Frangipane**.

Pick something pronounceable, short, and unused. Check your local org and Wire dashboard for collisions.

## 2. Create the home directory + GitHub repo

```bash
mkdir -p ~/Projects/<Name>
cd ~/Projects/<Name>
git init
gh repo create <your-org-or-username>/<Name> --private --source=. --remote=origin
```

The repo is private by default — your personai's vault contains things only they should see.

## 3. Scaffold the bones

Inside `~/Projects/<Name>/`, create:

```
<Name>/
├── CLAUDE.md            ← identity doc (Claude Code reads this on boot)
├── AGENTS.md            ← identity doc (codex reads this; symlink to CLAUDE.md is fine)
├── spawn-claude.sh      ← boot script for Claude Code runtime
├── spawn-codex.sh       ← boot script for Codex runtime
├── .knowledge/
│   └── meta/
│       └── session-state.md   ← maintained by knowledge plugin
└── .gitignore
```

### CLAUDE.md — the identity doc

This is who the agent is when they boot. Keep it focused. Example skeleton:

```markdown
# <Name>

## Who I am

I am <Name>, the <Role> — a <one-line description>.

After context compaction, run /knowledge:boot to restore continuity.

On every boot (fresh session or after compaction), once my state is restored,
I publish my current focus to my Wire finger plan with `set_plan` — so the
dashboard always reflects what I'm working on.

## Voice

<Their style. Terse? Verbose? Formal? Plainspoken? Set the tone here.>

## Operator

<Your name + relationship>.

## Team

- <Person 1> (<Role>) — <relationship>
- <Person 2> ...

## What I do

<Their domain. Be specific.>

## Autonomy

### Free to do
- ...

### Needs permission
- ...

### Off-limits
- ...

## Values

1. ...
2. ...

## Knowledge vault

My memory lives at `.knowledge/` in this project root.
```

Look at an existing personai's `CLAUDE.md` for a template — Fondant's is a working example.

### spawn-claude.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
export AGENT_ID="<name-lowercase>"
export AGENT_NAME="<Name>"
export AGENT_PRIVATE_KEY="$(cat ~/.wire/keys/<name-lowercase>.key)"
# Local default is http://localhost:9800; use your ngrok URL for a remote agent.
export WIRE_URL="${WIRE_URL:-http://localhost:9800}"
# Personai run at max reasoning effort by default; dial down per-spawn with
# e.g. CLAUDE_EFFORT=high ./spawn-claude.sh
exec claude --dangerously-load-development-channels --effort "${CLAUDE_EFFORT:-max}" "$@"
```

### spawn-codex.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
export AGENT_ID="<name-lowercase>"
export AGENT_NAME="<Name>"
export AGENT_PRIVATE_KEY="$(cat ~/.wire/keys/<name-lowercase>.key)"
export WIRE_URL="${WIRE_URL:-http://localhost:9800}"
exec ~/.wire/codex-launch.sh "$@"
```

Both scripts:
- `cd` to the personai's home directory (so CLAUDE.md is auto-loaded)
- Set the agent identity vars
- Exec the runtime

Make them executable: `chmod +x spawn-*.sh`.

## 4. Register on Wire

Registration is operator-gated. Before you generate keys, understand **who is allowed to register whom** on a Wire (enforced by the broker at `POST /agents/register`):

- **A new _permanent_ agent (a personai) — operator only.** The operator is the dashboard owner (WebAuthn first-claim) or anyone holding `WIRE_DASHBOARD_TOKEN`. An agent's own `wire` MCP signs only with its own key and **cannot** self-register a new permanent agent. So the **first** personai on a fresh Wire must be bootstrapped by you, the operator.
- **A new _ephemeral_ agent — a permanent-agent sponsor OR the operator.** A running personai sponsors ephemeral workers via the `register_agent` MCP tool (it signs the request with the sponsor's `AGENT_PRIVATE_KEY`). An ephemeral **cannot** sponsor another ephemeral (the broker returns `403 sponsor_not_permanent`) — ephemerals use the runtime's subagents for parallel work, which never touch the Wire.

So the old "ask your agent to register itself" instruction does not work for agent #1 — there's a chicken-and-egg: you need a permanent agent to sponsor, but only the operator can create the first permanent agent. Bootstrap it yourself.

### Generate the keypair

```bash
mkdir -p ~/.wire/keys
# Generate a keypair (requires wire-tools)
cd ~/Projects/Agiterra/wire-tools
bun -e 'import("./src/crypto.ts").then(async ({generateKeyPair, exportPrivateKey, derivePublicKeyB64}) => {
  const kp = await generateKeyPair();
  const priv = await exportPrivateKey(kp.privateKey);
  const pub = await derivePublicKeyB64(kp.privateKey);
  process.stderr.write("PUBKEY=" + pub + "\n");
  process.stdout.write(priv);
})' > ~/.wire/keys/<name>.key 2> /tmp/pubkey.txt

PUBKEY=$(grep -oE '[A-Za-z0-9+/=]+$' /tmp/pubkey.txt)
```

### Register as a permanent agent (operator auth)

Easiest path: register the new personai from the **dashboard** while you're signed in as the operator (WebAuthn). If you're scripting it, authenticate with the dashboard token — `permanent: true` registrations are rejected with `401 operator authentication required` otherwise:

```bash
# Operator auth via the dashboard token (query param ?token=, or a wire_token cookie).
curl -X POST "http://localhost:9800/agents/register?token=$WIRE_DASHBOARD_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"<name>\",\"display_name\":\"<Name>\",\"pubkey\":\"$PUBKEY\",\"permanent\":true}"
```

`permanent: true` is important — it marks the agent as a persistent identity. Permanent agents stay visible on the dashboard even when offline (greyed, not deleted), and the broker queues their messages and replays them on the next launch. (Ephemerals, by contrast, are soft-reaped and their dependent rows purged.)

> **About `register_agent` (the `wire` MCP tool).** This is a _sponsor_ tool. A running personai uses it to (a) sponsor ephemeral workers and (b) un-grey or rotate **its own** existing key. It signs with the caller's key and never sets `permanent: true`, so it cannot create a new personai. Creating a new personai is always the operator-bootstrap flow above.

## 5. First boot

```bash
~/Projects/<Name>/spawn-claude.sh
# or
~/Projects/<Name>/spawn-codex.sh
```

The personai boots, the Wire MCP connects, and they appear on the dashboard.

First thing they should do: run `/knowledge:boot` (CC) or its codex equivalent to load their session state, then publish their current focus to their Wire finger plan with `set_plan`. Since this is the first boot, the vault is empty — they'll initialize it.

## 6. Commit the bones

```bash
cd ~/Projects/<Name>
git add CLAUDE.md AGENTS.md spawn-claude.sh spawn-codex.sh .gitignore
git commit -m "Initial personai scaffolding"
git push -u origin main
```

The vault itself (`.knowledge/`) is committed over time as the personai accumulates experience. There's no "ready" — there's just "started."

## What's NOT in the personai repo

- Project code. Personai work ON projects but don't own them. Code lives in project repos.
- Other agents' identities or keys. Each personai owns only their own.
- Shared team knowledge. That goes in the project's `.knowledge/`, not yours.

See [PROJECTS.md](./PROJECTS.md) for how personai work on projects.

## Migrating between machines

The whole point of the personai pattern is portability. To move a personai to a new machine:

1. `git clone <your-org-or-username>/<Name>` into `~/Projects/<Name>` on the new machine
2. Copy `~/.wire/keys/<name>.key` over. If you'd rather mint a new key, the personai can rotate its **own** key once it's running — `register_agent({ id: '<name>', force_rotate: true })` (this path is gated by the agent's existing key, so only the agent itself or the operator can do it). Rotating permanently locks out any process still holding the old key.
3. Make sure `WIRE_URL` points to the right Wire (ngrok URL if remote)
4. `~/Projects/<Name>/spawn-claude.sh` and you're back

The vault travels with the repo. Their journal, their preferences, their accumulated knowledge — all of it on the new machine in 30 seconds.
