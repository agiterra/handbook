# Setting up a personai

A "personai" is a persistent AI agent — a persona-AI. They have a name, a vault, a GitHub repo of their own, and a `spawn-claude.sh` / `spawn-codex.sh` that boots them on any machine.

This guide walks an operator (and their agent) through setting one up end-to-end. Read [CORE.md](./CORE.md) first if you haven't yet.

## Mental model

A personai is **owned by no project**. They're a person, not a worker. They have:

- A name (Fondant, Brioche, Eclair, Croissant — pastry naming is convention, not requirement)
- A home directory (`~/Projects/<Name>/`) outside any code repo
- A GitHub repo (`<your-username>/<Name>`) containing their identity
- A vault (`.knowledge/`) of their accumulated experience
- A `CLAUDE.md` / `AGENTS.md` that defines who they are
- One or more spawn scripts that boot them with the right environment

They can work on any project, but they're not OF any project.

## 1. Pick a name

Convention: pick a pastry. Real ones we use: **Fondant** (toolsmith), **Brioche** (engineering director), **Eclair** (engineer), **Croissant** (engineer), **Stollen**, **Palmier**, **Panettone**, **Sable**, **Frangipane**.

Pick something pronounceable, short, and unused. Check your local org and Wire dashboard for collisions.

## 2. Create the home directory + GitHub repo

```bash
mkdir -p ~/Projects/<Name>
cd ~/Projects/<Name>
git init
gh repo create <your-org>/<Name> --private --source=. --remote=origin
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

Look at an existing personai's CLAUDE.md for a template — [Fondant's](https://github.com/agiterra/Fondant/blob/main/CLAUDE.md) is a working example.

### spawn-claude.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
export AGENT_ID="<name-lowercase>"
export AGENT_NAME="<Name>"
export AGENT_PRIVATE_KEY="$(cat ~/.wire/keys/<name-lowercase>.key)"
export WIRE_URL="${WIRE_URL:-https://the-wire.ngrok.io}"
exec claude --dangerously-load-development-channels "$@"
```

### spawn-codex.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
export AGENT_ID="<name-lowercase>"
export AGENT_NAME="<Name>"
export AGENT_PRIVATE_KEY="$(cat ~/.wire/keys/<name-lowercase>.key)"
export WIRE_URL="${WIRE_URL:-https://the-wire.ngrok.io}"
exec ~/.wire/codex-launch.sh "$@"
```

Both scripts:
- `cd` to the personai's home directory (so CLAUDE.md is auto-loaded)
- Set the agent identity vars
- Exec the runtime

Make them executable: `chmod +x spawn-*.sh`.

## 4. Register on Wire

Generate a keypair and register the agent. From an existing personai you have running:

> "Register me a new personai named `<name>` and stash the private key in `~/.wire/keys/<name>.key`."

Or manually:

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

# Register on Wire
curl -X POST http://localhost:9800/agents/register \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"<name>\",\"display_name\":\"<Name>\",\"pubkey\":\"$PUBKEY\",\"permanent\":true}"
```

`permanent: true` is important — it marks the agent as a persistent identity that should never be hard-deleted from the dashboard.

## 5. First boot

```bash
~/Projects/<Name>/spawn-claude.sh
# or
~/Projects/<Name>/spawn-codex.sh
```

The personai boots, the Wire MCP connects, and they appear on the dashboard.

First thing they should do: run `/knowledge:boot` (CC) or its codex equivalent to load their session state. Since this is the first boot, the vault is empty — they'll initialize it.

## 6. Commit the bones

```bash
cd ~/Projects/<Name>
git add CLAUDE.md AGENTS.md spawn-claude.sh spawn-codex.sh .gitignore
git commit -m "Initial personai scaffolding"
git push -u origin main
```

The vault itself (`.knowledge/`) is committed over time as the personai accumulates experience. There's no "ready" — there's just "started."

## What's NOT in the personai repo

- Project code. Personai works ON projects but doesn't own them. Code lives in project repos.
- Other agents' identities or keys. Each personai owns only their own.
- Shared team knowledge. That goes in the project's `.knowledge/`, not yours.

See [PROJECTS.md](./PROJECTS.md) for how personai work on projects.

## Migrating between machines

The whole point of the personai pattern is portability. To move a personai to a new machine:

1. `git clone <your-org>/<Name>` into `~/Projects/<Name>` on the new machine
2. Copy `~/.wire/keys/<name>.key` over (or generate a new pair and `register_agent({id, pubkey, force_rotate: true})` to rotate)
3. Make sure `WIRE_URL` points to the right Wire (ngrok URL if remote)
4. `~/Projects/<Name>/spawn-claude.sh` and you're back

The vault travels with the repo. Their journal, their preferences, their accumulated knowledge — all of it on the new machine in 30 seconds.
