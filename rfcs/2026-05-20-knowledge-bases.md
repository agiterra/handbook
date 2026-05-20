# Knowledge Bases — RFC

**Status:** Proposal · **Date:** 2026-05-20 · **Authors:** Tim Garthwaite, Fondant (Toolsmith)

## TL;DR

Restructure the `.knowledge/` vault from a flat folder-of-files into a small set of **bases** organized by category (`code/`, `role/`, `persona/`). Each base is a self-contained knowledge container. Personae and roles get their own bases; ephemerals do not. This solves three concrete problems at once:

1. **Toolsmith ↔ Fondant split.** The public Toolsmith knowledge (Agiterra-tools expertise) becomes its own base that anyone can pull in. Operator-specific personae layer on top via private bases.
2. **Multi-ephemeral session-state collisions.** In a single project dir (e.g. `fabrica-v3/`), multiple ephemerals trampling one `meta/session-state.md` goes away — ephemerals have no base of their own and their runtime state lives outside the vault.
3. **`seance:summon` for arbitrary roles.** A borrowed role (Toolsmith, engineer, anyone) cleanly fits into the same category model — the borrower's private workspace is separate from the canonical role content.

## Motivation

Three real incidents drove this:

- **Brian's seance use case** (2026-05-19/20): a "fondant" instance running on Brian's machine helped diagnose bugs and produce PRs (`agiterra/crew-tools` #73, `agiterra/bridge-claude-code` #1). For this to scale beyond a hand-cloned Fondant, the public toolkit-expertise content must be cleanly separable from any one operator's persona.
- **Multi-ephemeral collisions in fabrica-v3** (2026-05-19): now that ephemerals work in a single shared `fabrica-v3/` dir (not per-task worktrees), they trample each other's `session-state.md`, `journal.db`, and precompact recovery — all live at the vault root.
- **Cross-pollination friction** (ongoing): operator preferences and engineering learnings get trapped in one persona's vault with no clean promotion path to "every future agent on this project benefits."

The current `.knowledge/` schema mixes personal identity (`identity.md`), team relationships (`feedback-board-hygiene.md` references Brioche), engineering universals (`feedback-rca-before-theorizing.md`), and operational state (`meta/session-state.md`) in one flat namespace. The mixed schema is the root cause of all three incidents.

## Concepts

### Base

A **base** is a self-contained knowledge container — a directory containing `feedback-*.md`, `convention-*.md`, `design-*.md`, optionally a `meta/`, `vectors.db`, and a `semantic-index.json`. Each base is owned by something with a persistent identity: a code repo, a role, or a persona.

### Vault

A **vault** is a collection of bases, rooted at `.knowledge/`. A vault lives inside a code repo, a personai dir, or any workspace where agent-driven work happens.

### Categories

Three reserved top-level dirs under `.knowledge/`:

| Category | What | Cardinality |
|---|---|---|
| `code/` | Shared knowledge about the repo this vault lives in | Singleton |
| `role/` | Reusable composable expertise (toolsmith, engineer, solidity-engineer, bug-engineer, ...) | Many |
| `persona/` | Operator-specific identities (fondant, brioche, herald, ...) | Many |

### Ephemerals

Ephemerals have a name (for screen sessions, wire identity, audit) but **no base of their own**. They:

- **Read** from their assigned roles' bases + the vault's `code/` base + any persona base if one is attached
- **Write** only through `/knowledge:promote`, which targets a specific shared base (typically `code/` or a role base) with PII review
- Have **runtime state** (session-state, journal, precompact recovery) outside the vault, in a crew-managed location (`~/.crew/runtime/<agent-id>/`)
- Runtime state is discarded on agent close

Ephemerals borrow from bases but contribute back only through explicit, audited promotion. Personal journal-style writes are not a concept for ephemerals — if they learn something, they promote it to a shared base or it dies with them.

## Layout

```
.knowledge/
  code/                          ← singleton: shared knowledge about this repo
    feedback-*.md
    convention-*.md
    design-*.md
    ...
  role/
    toolsmith/                   ← role base (often a nested gitignored repo cloning agiterra's canonical toolsmith)
    engineer/
    solidity-engineer/
    bug-engineer/
  persona/
    fondant/                     ← persona base (nested gitignored repo for the operator's persona)
```

Real example for a fabrica-v3 working dir:

```
~/Projects/Fabrica/fabrica-v3/
  .git/                          ← fabrica-v3 monorepo
  .gitignore                     ← includes:
                                    .knowledge/role/*
                                    .knowledge/persona/*
                                    (only code/ is tracked by fabrica-v3)
  .knowledge/
    code/                        ← tracked, shared project knowledge
      feedback-*.md, ...
    role/
      toolsmith/                 ← nested gitignored: clone of agiterra/agiterra (sparse for toolsmith role)
      engineer/                  ← nested gitignored: clone of role-engineer source
    persona/
      brioche/                   ← nested gitignored: clone of mividtim/brioche-persona
  fabrica-v3-api/, soil-app/, ... (existing submodules, no .knowledge/ of their own)
```

Real example for an "agiterra root" meta-repo (proposed alongside this RFC):

```
~/Projects/Agiterra/agiterra/    ← NEW meta-repo containing all AMAT tools as submodules
  .git/
  .gitignore: .knowledge/role/*, .knowledge/persona/*, .knowledge/role/toolsmith/  ← except toolsmith, which IS tracked
  .knowledge/
    code/                        ← tracked: knowledge about working in the AMAT ecosystem
                                   (e.g., feedback-crew-knowledge-wire-no-cross-refs.md)
    role/
      toolsmith/                 ← tracked by THIS repo: the canonical toolsmith role
        agent.md
        feedback-*.md, convention-*.md, ...
    persona/
      fondant/                   ← nested gitignored: clone of mividtim/fondant-persona
  wire/, wire-tools/, crew/, crew-tools/, knowledge-tools/, bridge-tools/, ...  (submodules)
```

The canonical Toolsmith role lives at `.knowledge/role/toolsmith/` in the agiterra meta-repo. Anyone wanting to adopt the Toolsmith role elsewhere (e.g., inside fabrica-v3) clones the agiterra meta-repo via sparse-checkout for that path.

## Per-vault KX (consolidated)

One KX sidecar per vault, NOT per workspace/base. KX agent ID derived from vault path hash (`kx-<sha8(vault_path)>`, same scheme as today). The KX:

- Discovers all bases in its vault on startup
- Maintains a single queue: `.knowledge/meta/index-queue.jsonl` with records `{base: "<category>/<name>", path: "...", op: "index"|"remove"}`
- Indexes each file into the base-local `vectors.db` and `semantic-index.json`
- One Haiku context serves all bases (prompt-cache reuse across them)
- Launch-once guarded by `.knowledge/meta/kx.pid` + file lock

Per-base vector storage means borrowed roles' embeddings don't bleed across operators (e.g., `fondant/` workspace's vectors stay separate from a Toolsmith-borrower's overlay).

## Schema migration

Migration tool detects v1 vaults via heuristic: presence of `.knowledge/feedback-*.md` or `.knowledge/meta/session-state.md` at the root (versus inside a category subdir).

Auto-detected base name comes from:

1. Explicit `--base <category>/<name>` override (e.g. `--base persona/fondant`)
2. Parent dir matching `~/.personai/<name>/` → `persona/<name>`
3. Repo dir name (e.g. fabrica-v3) → `code/` (singleton)
4. Prompt operator if ambiguous

Migration writes a `.knowledge/.vault-format` file with `version: 2` to prevent re-migration on subsequent runs.

Migration runs once per vault, idempotent, reversible (the migration tool can dump a tar of pre-migration state alongside, kept for ~30 days).

## API changes: knowledge-tools v1.0.0

### Schema constants

```ts
export const CATEGORIES = ["code", "role", "persona"] as const;
export type Category = (typeof CATEGORIES)[number];

export interface Base {
  category: Category;
  name: string;            // "" for the singleton code base
  path: string;            // .knowledge/<category>/[<name>/]
}
```

### Read

```ts
export function readMerged(
  vault: string,
  basesInScope: Base[],     // determined by caller's role assignments + own persona + code
  query: string | RegExp,
): MergedResult;
```

Merge order: last in `basesInScope` wins on file-path conflicts.

### Write

```ts
export function writeToBase(
  vault: string,
  base: Base,
  relativePath: string,
  content: string,
): void;
```

There is no default-base write target in the library — callers must declare which base they're writing to. The `knowledge-claude-code` skills layer the policy (own persona/role for personal saves; explicit promotion for shared bases).

### Promote

```ts
export async function promote(
  vault: string,
  fromBase: Base,
  toBase: Base,
  relativePath: string,
  options: PromoteOptions,
): Promise<PromoteResult>;
```

Runs PII review (configurable patterns + denylist), copies the file, and removes from source if `options.move === true`.

## Skill changes: knowledge-claude-code v1.0.0

- **`/knowledge:boot`** — discover vault, identify caller's persona + roles (from CLAUDE.md or environment), warm merged-read across owned + assigned bases.
- **`/knowledge:save`** — writes go to caller's persona base (for persistent agents) or no-ops (for ephemerals).
- **`/knowledge:fast-save`** — same target as save, skips journal + vector update.
- **`/knowledge:promote <path>`** — new skill. Moves a file from caller's own base to a target shared base with PII review.
- **`/knowledge:recycle`** — for persistent agents, fast-save + clear + boot. For ephemerals: refuses (recycle is not a concept; use `handoff` if state needs to be passed).
- **`/knowledge:migrate`** — runs the schema migration tool on the current vault.

## PII guard

For PRs into public bases (Toolsmith, anything in `code/` of a public repo), a pre-commit hook scans the diff for:

- Absolute paths matching `/Users/<name>/`, `/home/<name>/`
- Email addresses
- Private IPs (RFC 1918 ranges, excluding loopback)
- ngrok URLs
- Operator/persona/agent names from a `.toolsmith-denylist` config file the adopter maintains locally
- Anything outside the base being committed (catches accidental cross-base staging)

The hook blocks the commit and prints a precise error. Adopters extend the denylist per their setup. Toolsmith ships with the common AMAT-team names pre-populated as a starter.

## seance-tools update

`seance:summon <role-or-persona>` clones the relevant source repo into the borrower's working dir as a nested gitignored entry under `.knowledge/role/<name>/` or `.knowledge/persona/<name>/`. Borrower's session state lives in `~/.crew/runtime/<borrower-id>/` (per ephemeral runtime convention).

`seance:promote-back` runs `/knowledge:promote` then opens a PR to the summoned role/persona's upstream repo (Toolsmith, fondant-persona, etc.). PII guard runs against the diff before the PR.

## Implementation order

1. **knowledge-tools v1.0.0** — schema constants, base discovery, read/write/promote API, migration tool, KX-per-vault with multi-base queue, per-base vector storage. Major version bump justified by breaking schema change.
2. **knowledge-claude-code v1.0.0** — new skills (`/knowledge:promote`, `/knowledge:migrate`), reworked save/fast-save/recycle to be category-aware.
3. **knowledge-indexer-tools** — KX boot scans all bases in its vault; queue records carry the base name.
4. **seance-tools update** — workspace-aware summon, PII-guard hook installer.
5. **agiterra meta-repo bootstrap** — new repo `agiterra/agiterra` containing all AMAT tool submodules + the canonical toolsmith role at `.knowledge/role/toolsmith/`.
6. **Fondant migration** — Tim's existing Fondant dir becomes a clone of the agiterra meta-repo + nested gitignored `mividtim/fondant-persona` at `.knowledge/persona/fondant/`.
7. **Brioche migration** — `mividtim/brioche-persona` created from current Brioche/.knowledge content + nested gitignored under `fabrica-v3/.knowledge/persona/brioche/`. fabrica-v3's existing root-level `.knowledge/` content migrates into `.knowledge/code/`.

Steps 1–4 are sequential (schema gates everything). Steps 5–7 can land in parallel after 4 ships.

## Open questions

1. **Codex parity.** Knowledge-codex needs an equivalent v1.0.0. Should land at the same time as knowledge-claude-code v1.0.0, or one cycle later? Lean: same cycle, since Brian's seance use case spans both runtimes.
2. **Role-base distribution.** Are individual roles their own repos (`agiterra/role-engineer`, `agiterra/role-solidity-engineer`) or all bundled in one `agiterra/roles` repo with sparse-checkout? Lean: individual repos, so roles evolve independently.
3. **Operator's choice of agent file** vs the role's bundled `agent.md`. If the operator's persona overrides the role's prompt, how does the merge work? Open.
4. **Submodule-specific knowledge.** Current proposal treats AMAT knowledge as cross-cutting and skips per-submodule subdivision. If specific submodules later accumulate distinct knowledge worth fencing off, revisit.
5. **Plugin KBs.** Plugins like github-claude-code aren't core AMAT but ship their own conventions. For now: tabled, treat all AMAT repos as first-class. Revisit if plugin knowledge becomes substantial.

## Out of scope

- Vendor-specific (Claude Code vs Codex) divergence in the base format. Bases are vendor-neutral; vendor-specific surfaces (CLAUDE.md, AGENTS.md) reference but don't mirror them.
- A2A / x402 integration. Separate concern, separate RFC.
- Cross-machine vault federation (multi-machine sync of bases). Bases sync via their git remotes today; federation is a Wire-level concern.
