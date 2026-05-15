# Projects — the parent-repo pattern

How to organize a real codebase so your personai (and any ephemeral engineers they spawn) can navigate it cleanly.

## The shape

```
~/Projects/<Project>/                  ← parent-repo
├── .knowledge/                        ← PROJECT vault (shared)
│   ├── MEMORY.md
│   ├── journal.db
│   └── ...
├── <service-a>/                       ← submodule (its own git repo)
├── <service-b>/                       ← submodule
├── <library-c>/                       ← submodule
├── worktrees/                         ← branch-per-engineer workspaces
│   ├── <eclair>/<ticket-branch>/      ← <eclair>'s worktree for this ticket
│   └── <palmier>/<ticket-branch>/     ← <palmier>'s worktree, different branch
└── README.md                          ← orchestrator's entry point
```

One parent-repo per **company/job/client/product**. Real-world examples: Tim's `Fabrica/` parent-repo holds all the Fabrica company's services as submodules. His `Agiterra/` parent-repo holds the tooling ecosystem.

## Why a parent-repo

Three reasons:

1. **One vault, many services.** Project knowledge (architectural decisions, conventions, why-this-was-done) lives in `.knowledge/` at the parent-repo level — accessible to any agent working on any service.
2. **Worktrees for concurrent agents.** Multiple agents can work on the same submodule on different branches via git worktrees, without stepping on each other.
3. **Coordinated submodule pins.** When service A depends on a specific version of library C, the parent-repo records the pin atomically.

## Knowledge placement decision tree

Every fact has a home. The orchestrator's job is to put facts in the right home.

```
Is this knowledge about a PERSON (the operator, a teammate, an external collaborator)?
  → Personai vault (your own .knowledge/)

Is this knowledge about how you yourself work — your preferences, your style, your habits?
  → Personai vault

Is this knowledge about the PROJECT — its architecture, conventions, infrastructure, history, who-decided-what?
  → Project vault (parent-repo's .knowledge/)

Is this knowledge tightly coupled to a single service or library?
  → That submodule's own .knowledge/ if it has one; otherwise project vault with a clear tag.

Is this knowledge tactical / session-only (current task state, in-progress reasoning)?
  → Not a vault at all. Use TaskCreate / TaskUpdate. Vaults are for things that should outlive the session.
```

Concrete examples:

| Fact | Home |
|---|---|
| "Tim prefers terse, technical responses" | Your personai vault |
| "Tim is the founder of Fabrica" | Personai vault (about a person) |
| "Fabrica uses React Server Components + Next.js 16 on Vercel" | Project vault |
| "The `fabrica-v3` parent-repo's submodules pin via `tim/<ticket>` branch convention" | Project vault |
| "I learned that `bun install` fails silently when the lockfile is stale" | Personai vault (your learning) |
| "ENG-3149 is in flight, branch is X" | NOT a vault. TaskUpdate or session-state.md. |

## Worktrees per submodule

A worktree is a git checkout living at a separate path from the main checkout, but sharing the same `.git/` directory. Multiple engineers can hold different branches of the same repo simultaneously.

Standard pattern:

```bash
cd ~/Projects/<Project>/<submodule>
git worktree add ../worktrees/<engineer>/<ticket-branch> -b tim/<ticket>
```

Each engineer (personai or ephemeral) gets their own worktree per ticket. Their CWD when working is `~/Projects/<Project>/worktrees/<engineer>/<ticket-branch>/<submodule>`. The CLAUDE.md / project vault is still discoverable via parent-repo path lookup.

When the ticket merges, drop the worktree:

```bash
git worktree remove ../worktrees/<engineer>/<ticket-branch>
git branch -d tim/<ticket>
```

## Bootstrapping a new project parent-repo

```bash
mkdir -p ~/Projects/<Project>
cd ~/Projects/<Project>
git init
gh repo create <your-org>/<Project> --private --source=. --remote=origin

# Initialize the project vault
mkdir -p .knowledge/meta
cat > .knowledge/MEMORY.md <<EOF
# Project memory index

<one-line summary of the project>

## Conventions

(filled in over time)

## Architecture

(filled in over time)

## Stakeholders

(filled in over time)
EOF

# Add a top-level CLAUDE.md so any agent working in this dir gets project context
cat > CLAUDE.md <<EOF
# <Project>

<one-paragraph: what is this project, who owns it, what's the current focus>

## Submodules

<List of submodules + one-liner per>

## Knowledge

Project vault is at \`.knowledge/\`. Run \`/knowledge:boot\` to load context. Project conventions live in there.

## Working pattern

- Worktrees per engineer per ticket: \`worktrees/<engineer>/<ticket-branch>/\`
- Submodule branches: \`tim/<ticket-id>\` (or org convention)
- PRs opened by bot or by engineer; reviews per project policy
EOF

git add .
git commit -m "Initial project scaffold"
git push -u origin main
```

Add submodules as the project grows:

```bash
git submodule add git@github.com:<org>/<service-a>.git
git submodule add git@github.com:<org>/<service-b>.git
```

## Orchestrator's job

The orchestrator (a senior personai — often the "engineering director") is responsible for:

1. **Spawning engineers** to work on submodule branches, per ticket
2. **Aggregating results** — checking on engineers, reading their wrap-ups, coordinating cross-engineer dependencies
3. **Routing knowledge** — when an engineer learns something, the orchestrator decides whether it goes to that engineer's personai vault, the project vault, or both
4. **PR coordination** — once an engineer's branch is ready, the orchestrator triggers review pipelines and merges

The orchestrator is NOT writing code themselves. They're the architect + dispatcher.

## Cross-references

- Per-engineer setup: [PERSONAI.md](./PERSONAI.md)
- Spawning ephemeral engineers: the [crew plugin README](https://github.com/agiterra/crew-claude-code)
- Identity model: [CORE.md](./CORE.md)
