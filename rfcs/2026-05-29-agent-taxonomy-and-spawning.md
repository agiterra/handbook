# Agent Taxonomy: Identity, Role, and When to Spawn — RFC

**Status:** Proposal · **Date:** 2026-05-29 · **Authors:** Tim Garthwaite, Fondant (Toolsmith)

## TL;DR

"Agent" is overloaded. It collapses three *orthogonal* things, and most
disagreements about "how many agents should we have?" are really disagreements
about which of the three you mean:

1. **Identity** — is this a **personai** (persistent identity + vault +
   continuity) or an **ephemeral** (no standing identity, spawned for a task)?
2. **Role** — the *expertise bundle* an instance performs (a brief + a tool
   manifest + the knowledge it reads). Roles are reusable and composable; an
   instance can borrow one.
3. **Instance** — a *running session*. It always has one identity and performs
   one or more roles.

So: **an agent = an identity, running a session, performing a role.** Those are
three knobs, set independently. Two design rules fall out of keeping them
separate:

- **Agent vs tool:** offload *judgment* to an agent; offload *mechanics* to a
  tool. If a capability is already a clean tool, wrapping it in a standing agent
  is a regression, not an abstraction.
- **Persistent vs ephemeral:** give a role a *standing identity* only when it
  accumulates context, relationships, or ownership over *time*. Otherwise
  instantiate it on demand and let it retire.

## Motivation

The roster grows by accretion, and two recurring confusions drive bad calls:

- **"Personai" gets used for "role."** Someone proposes "an agent per gate" or
  "a Slack agent" and the team can't tell if that means a new *standing
  identity to maintain* or a *job some instance does*. The answer changes the
  cost by an order of magnitude.
- **The specialist-service temptation.** It's tempting to mint a standing agent
  for every nameable task ("the spawn manager," "the git agent") on the theory
  that delegation frees the orchestrator. Sometimes true, often backwards — a
  delegation hop *is* communication, and communication is usually the
  orchestrator's actual mandate.

Every standing personai also has a real, ongoing carrying cost: a vault that
drifts, a session-state to restore, a finger plan and badge to keep current, a
keypair, a screen session, an indexer sidecar, sometimes a Slack app — each a
surface that can go stale or break. The roster should grow deliberately, not by
reflex.

This RFC gives the team one shared frame to reason about it. It builds on
[CORE.md §1](../CORE.md) (the personai/ephemeral identity axis) and the
[Knowledge Bases RFC](./2026-05-20-knowledge-bases.md) (which already makes
`role/` a first-class, composable base — personae and roles get bases,
ephemerals don't).

## Concepts

### Identity — personai vs ephemeral

Unchanged from CORE.md, restated as one axis of the taxonomy:

- A **personai** is a persistent identity: a name, a personal vault, continuity
  across restarts, relationships, a standing seat on the team. It earns that
  identity by accumulating context over *time*.
- An **ephemeral** has no standing identity. It is spawned for a task or event,
  does the work, and retires. Its runtime state lives outside any personal
  vault.

### Role — the expertise bundle

A **role** is a *job description*, not a running thing:

- a **brief** (what this role is responsible for, how it judges "done"),
- a **tool manifest** (the MCP servers / permissions it needs — and *only*
  those), and
- the **knowledge** it reads (typically a shared `role/` or `code/` base, not a
  private vault).

Roles are reusable and composable. A personai has a *home* role (Toolsmith, Eng
Director, Marketing Director). An ephemeral is *given* a role at spawn. The same
role brief can be instantiated many times, by many identities.

### Instance — a running session

An **instance** is one live session. It has exactly one identity and performs
one or more roles. "Three engineers working tickets" = three instances,
ephemeral identity, all performing the `engineer` role.

> The sentence that resolves the confusion: **identity ≠ role.** An agent is an
> *identity* running a *session* performing a *role*. Ask which knob you're
> turning before you argue about the count.

## Rule 1 — Agent vs tool: judgment vs mechanics

Before asking "persistent or ephemeral?", ask whether this should be an *agent*
at all.

- **Mechanical** complexity — deterministic, well-specified, no judgment call —
  belongs in a **tool**. Spawning a worker (`agent_launch` + `pane_create` +
  `agent_attach`), opening a PR, reading a file: these are mechanics. The right
  place to hide their complexity is the tool layer, where the call is
  synchronous, reliable, and adds no liveness dependency.
- **Judgment** complexity — needs context, taste, or a decision that doesn't
  fit the requester's head — belongs in an **agent**. Holding a persona's voice
  in a Slack channel and deciding what's worth surfacing; reviewing a change for
  subtle risk; owning a marketing arc: these are judgment.

**Anti-pattern: the standing wrapper.** Putting a standing agent in front of a
capability that is *already* a clean tool. It looks like delegation but it:

- **couples liveness** — the operation now fails whenever the specialist is
  down, asleep, or busy, where a tool call would have just worked;
- **adds a comms hop** — a synchronous one-call operation becomes an async
  round-trip (message out → context-switch in → act → report back), slower and
  harder to debug; and
- **loads, rather than frees, the orchestrator** — the delegation conversation
  *is* communication overhead, and communication is usually exactly the mandate
  you were trying to protect.

What frees an orchestrator from mechanical work is a good *tool*, not a new
mouth to coordinate with. Reserve agents for judgment.

## Rule 2 — Persistent vs ephemeral: continuity vs task-scope

Once you've decided it's genuinely an agent (judgment), decide whether an
instance should *stand permanently* or *spawn on demand*.

Mint a **personai** (standing identity) only when the role:

- accumulates context that compounds over time (history, prior decisions),
- holds relationships other agents and the operator rely on, and
- owns a domain *continuously*, not just when an event fires.

Otherwise prefer an **ephemeral, spawned on demand**, with the role's "memory"
living in a **shared knowledge base** (`role/` or `code/`) that any instance
reads — not in a private vault that only exists while the session is up.

**Canonical example — a per-gate reviewer.** "One agent per merge gate" is a
*role* per gate, not a personai per gate. A gate fires on an *event* (a PR
reaching it), carries no long-term relationship, and its "memory" (this gate's
criteria, this repo's past failures) is *shared* repo knowledge. So: an
ephemeral instance of the `gate-<X>` role, triggered by the PR event, reading
the project's `code/` base, calling `gate_set`, then retiring. A *standing*
gate session would idle ~all the time, drift, and add a liveness dependency for
no continuity benefit. (Wire already delivers PR events as channel messages, so
event-triggered spawn is a natural fit.)

## Decision checklist

When someone proposes "we should have an agent for X":

1. **Is X mechanical or judgment?**
   - Mechanical → build/extend a **tool**. Stop.
   - Judgment → it's a **role**; continue.
2. **Write the role brief + minimal tool manifest + which knowledge base it
   reads.** If you can't name distinct judgment, a distinct manifest, and a
   distinct cadence, the boundary is wrong — don't carve it.
3. **Does X accumulate context / relationships / continuous ownership over
   time?**
   - Yes → it may warrant a **personai** (a standing seat). Budget the carrying
     cost honestly.
   - No → keep it **ephemeral**, spawned on demand, memory in a shared base.
4. **Default bias:** tools over agents for mechanics; ephemeral roles over new
   personae for everything intermittent or event-scoped. Grow the standing
   roster slowly.

## What this is not

- Not a ban on specialization — specialization is good. It's a push to express
  most of it as **roles and on-demand instances**, not standing identities.
- Not a claim that delegation is always wrong — delegating genuine *judgment*
  (a security audit, a gnarly migration, a design exploration) to a spawned
  specialist is exactly right. The line is judgment, not task-count.

## Open questions

- A **role-brief library** + per-role tool manifests + spawn ergonomics would
  make "lots of specific roles" cheap and reversible. Worth building as the
  lever once this frame is agreed (the bridge orchestrator is already pointed
  this way).
- Should CORE.md §1 be extended from a two-part (personai/ephemeral) to a
  three-part (identity / role / instance) model once this RFC is accepted?
