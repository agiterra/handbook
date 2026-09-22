# Core concepts

The 10-minute mental model for the Agiterra Multi-Agent Toolkit. Read this once and the rest of the docs (and the plugins) will make sense.

## 1. Agent identities: personai vs ephemeral

The Toolkit distinguishes **two kinds of agents**:

### Persistent agent ("personai")

A persona-AI. Has a name, a personality, a vault of accumulated knowledge, a GitHub repo of its own, and a `spawn-claude.sh` / `spawn-codex.sh` that boots it up on any machine. Identity persists across days, machines, and runtime restarts.

You ask your personai to do things. They remember why you asked, what you decided last time, who you mentioned, and what's in flight.

Pastry names are a convention (Fondant, Brioche, Eclair, Croissant, …) but anything works.

### Ephemeral agent

A short-lived worker spawned to do one specific job. Often spawned by a personai (the orchestrator) to parallelize work. Lives in a screen session, does the work, exits.

Ephemerals can still have identity for the duration of their existence — they register on Wire, they get a name, they can send and receive messages. They just don't survive past the task.

You usually don't talk to ephemerals directly. Your personai dispatches them and aggregates their results.

## 2. The Wire

The Wire is a small local message broker. It runs on your machine (or wherever you want), and every agent connects to it.

The Wire does three things:

1. **Routes messages** between agents (Ed25519-signed, dest-or-broadcast)
2. **Tracks liveness** — who's online, when they last spoke, who got reaped
3. **Hosts the dashboard** — a live web UI showing every agent's state, plan, and recent activity

By default, Wire listens on `http://localhost:9800`. Open it in a browser to see who's around.

### The dashboard

Live view of:
- Every registered agent — name, status (connected / stale / disconnected / reaped), kind (agent vs integration), last heartbeat
- Each agent's current plan (a free-text "what I'm working on" they publish)
- Recent message log — who said what to whom
- Wallets (if `wallet-claude-code` is in play) — addresses, balances, access permissions

### Going remote with ngrok

If you want your Wire (and dashboard) reachable from elsewhere — another machine, a phone, a different city — expose it with ngrok:

```bash
ngrok http 9800
```

Then set `WIRE_URL=https://your-tunnel.ngrok-free.app` on remote agents and they'll connect over the public tunnel.

This is the magic that makes "the Toolkit works across machines and locations" real. Your personai on your desktop can dispatch ephemerals running on a beefy cloud VM, and they all show up on the same dashboard.

## 3. Knowledge vaults

Agents need memory. The Toolkit gives each one a **vault** — a `.knowledge/` directory of markdown files with a TOC (`MEMORY.md`), a journal (`journal.db`), and an editorial index (`session-state.md`).

The vault answers two questions: "what does this agent already know?" and "why are things the way they are?"

### Two flavors

**Personai vault** — lives in the personai's own repo (e.g. `~/Projects/Fondant/.knowledge/`). Their identity, learnings, preferences, who they've worked with. Owned by them.

**Project vault** — lives in the parent-repo for a project (e.g. `~/Projects/Fabrica/.knowledge/`). Conventions, architectural decisions, why-this-was-done-this-way. Owned by the project, shared across any agent who touches it.

A personai working on a project reads BOTH vaults. Their personal vault tells them who they are; the project vault tells them how things work here.

### Where to put what

This is judgment work that good orchestrators internalize:

| Type of knowledge | Where it goes |
|---|---|
| "Tim prefers X" | Personai vault (about a person you work with) |
| "We chose React Server Components in 2025 because…" | Project vault (about how the project works) |
| "When I'm stuck, I should X" | Personai vault (about yourself) |
| "Linear tickets are tracked in workspace Y" | Project vault (about the project's infrastructure) |

See [PROJECTS.md](./PROJECTS.md) for the orchestrator's decision tree.

## 4. The plugin map

You don't have to install everything. Start small:

**Minimum useful setup**
- `wire` — the SSE inbound
- `wire-ipc` — outbound signed messaging
- `knowledge` — vault + journal

That gives you a single agent with persistent memory who can talk to other agents (if you spin them up).

**Recommended for an orchestrator**
- Add `bridge` — the orchestrator's plugin. It collapses the multi-step orchestration dances (register identity → assemble env → launch agent → place pane → attach → kick off the task) into single composite MCP calls (`spawn`, `handoff`, `close`, `compose-brief`, `health`, …). Bridge stays domain-naive: capability-specific behavior (GitHub, Linear, …) ships as separate `bridge-X` integration plugins. It spawns agents at the `project_dir` you hand it and forwards their env — it does **not** create worktrees or manage project layout (that's the consumer's and the agent's concern).
- Add `crew` — spawn and manage agents in persistent screen sessions (bridge builds on this)
- Add `knowledge-indexer` — auto-indexes vault writes for semantic + keyword search

> **`operator-relay` is a worker-side plugin, not an orchestrator one.** It runs on the **ephemeral workers** you spawn — a UserPromptSubmit hook that relays prompts typed directly into a worker's session up to its manager (you), so you see out-of-band operator input. You don't install or run it yourself; your workers load it automatically from the spawn root's plugin set.

**Optional, situational**
- `crew-themes` — pane backgrounds (purely aesthetic, but nice)
- `crew-fleet` — when you have more than one machine
- `wallet` — agentic crypto wallet (if you want your agents transacting)
- `agiterra-github` — agentic GitHub PR reviews / webhook integration

## 5. The dashboard, in practice

Once Wire is running, the dashboard is your control panel. Things you'll do there:

- Watch a new agent come online when you launch them
- See "stale" / "disconnected" markers when an agent's machine sleeps
- Read the live message log when you're debugging "did my message arrive?"
- Read each agent's current plan to understand what your team is doing

The dashboard is read-only by default. Some plugins (wallet, GitHub) expose write actions there too.

## 6. Sane defaults

A few things the Toolkit opts you into that may surprise you:

- **Soft-reap, not hard-delete.** When an agent disconnects, its row goes grey, not gone. Identity (name, pubkey) is permanent. Re-registering the same id refreshes the row.
- **Ed25519 signing on everything.** Inter-agent messages are signed by the sender's private key. Forgery is not a concern.
- **No auto-approve.** Wallet operations and other sensitive actions always require an explicit decision — never policy-driven auto-approval.
- **`grep` inside an agent session skips `.gitignore`d files — use `/usr/bin/grep` on installed
  trees.** Claude Code shadows both `find` and `grep` from the shell snapshot
  (`# Shadow find/grep with embedded bfs/ugrep`). The grep function runs
  `ARGV0=ugrep claude -G --ignore-files …`, and `--ignore-files` honours `.gitignore`. Every
  plugin repo ignores `node_modules/` — which for a plugin is **100% of the implementation** —
  so a recursive `grep` over an installed tree reports a symbol ABSENT while it is present.
  ⚠️ It fails as a **false negative**: missing things look missing, so the result reads as a
  finding rather than as a broken instrument. `--no-ignore` and `-R` do not restore the matches;
  only `command grep` or `/usr/bin/grep` do.
  Found 2026-09-22 by a lane whose `grep -rn isReachableCallbackUrl` returned 0 over a tree that
  contained 14 matches. Confirmed in a second session with a discriminating test: one marker
  string in a tracked file and in a gitignored one — the shim found 1, `/usr/bin/grep` found 2,
  with the tracked file as a positive control proving the test could find anything at all.
  `find` is shadowed too (bfs) but does **not** honour `.gitignore` — the same test finds the
  ignored file both ways. So only `grep` corrupts evidence; don't over-correct on `find`.

## Next

- [SETUP-CLAUDE-CODE.md](./SETUP-CLAUDE-CODE.md) — install on Claude Code
- [SETUP-CODEX.md](./SETUP-CODEX.md) — install on Codex
- [PERSONAI.md](./PERSONAI.md) — set up your first persistent agent
- [PROJECTS.md](./PROJECTS.md) — multi-repo project pattern
