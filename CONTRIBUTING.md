# Contributing to the Agiterra Toolkit

Welcome. This doc covers what you need to know to contribute changes to the Agiterra Toolkit that we'll want to merge.

## The packaging standard

This is the single most important architectural rule. Every Agiterra plugin family follows this three-package layout:

```
<X>-tools/             ← vendor-neutral primitives + the MCP server
<X>-claude-code/       ← THIN CC adapter (server.ts is a one-liner: import startServer; call it)
<X>-codex/             ← THIN codex adapter (same pattern)
```

**All real logic lives in `<X>-tools`.** The adapter packages exist only to:
- Declare the right runtime (`bin: ./server.ts`)
- Pin their `-tools` dependency to a specific tag
- Set runtime-specific `postInstall` (e.g., `bun install --frozen-lockfile` for CC)

Concrete reference: [`wire-claude-code/server.ts`](https://github.com/agiterra/wire-claude-code/blob/main/server.ts) is six lines.

Why this matters:

- **One change, two runtimes.** A new tool in `wire-tools/src/mcp-server.ts` ships to both CC and codex with a dep bump in each adapter.
- **No vendor lock-in.** Future runtimes (Cursor, Aider, your-runtime-here) get a new adapter — not a rewrite.
- **Adapters become commodities.** All the value is in `-tools`.

When you propose a change: if you find yourself adding logic to a `-claude-code` or `-codex` package, stop. The logic belongs in `-tools`.

## Separation of plugin domains

Three big plugin families. They stay distinct:

| Plugin family | Owns |
|---|---|
| **Wire** | Agent ↔ Wire connection, identity (register_agent), signed messaging (send_message), broker dashboard, heartbeats |
| **Crew** | Screen sessions, panes, agent launch/attach/detach/close, machine registry, terminal theming |
| **Knowledge** | Vault storage, journal, search, compaction recovery, indexer sidecar |

Cross-cutting features compose via **conventions, not imports**. Examples of doing it right:

- `operator-relay` uses Wire IPC to forward prompts — it doesn't reach into Wire's internals or crew's screen state. It just publishes a Wire message.
- The `$STY` env var convention bridges Wire's "deliver to codex" use case with the screen sessions Crew manages. Wire doesn't import crew; it reads a Unix-level env var that screen sets.

If you're building something that needs Crew AND Wire AND Knowledge to all know about each other, you're probably solving the wrong problem. Step back.

## Pull requests

### Where to file
- Bug or feature for a specific plugin → that plugin's repo
- Cross-cutting issue → [agiterra/handbook](https://github.com/agiterra/handbook) for now (we'll route)
- Question about how to use → [agiterra/handbook discussions](https://github.com/agiterra/handbook/discussions) (if enabled) or open a handbook issue

### PR checklist

- [ ] If you touched a `-tools` package, the change works on BOTH `-claude-code` and `-codex` adapters (or you've documented why one doesn't apply)
- [ ] If you bumped a `-tools` version, you've also bumped the dependent adapter package.json AND the corresponding `.claude-plugin/plugin.json` version
- [ ] You tagged the new version (`git tag vX.Y.Z && git push --tags`) AFTER merging
- [ ] You added tests if the change touches Wire's protocol, Crew's lifecycle, or Knowledge's vault format
- [ ] You DIDN'T add comments explaining what the code does. Only why, when the why is non-obvious. See coding standards below.
- [ ] Commit messages explain motivation, not just mechanics

### Adapter release tagging

When you bump a `-tools` package, the adapters MUST bump atomically:

1. Update `<X>-tools/package.json` version + tag
2. Update `<X>-claude-code/package.json` dep pin + plugin.json version
3. Update `<X>-codex/package.json` dep pin + plugin.json version
4. Tag each adapter at its new version

A `-tools` bump without adapter bumps means CC and codex agents are reading stale code. There's no "library auto-tracks main" — adapters pin to tags.

## Coding standards

### Don't comment what; comment why

Default: write no comments. Only add one when the WHY is non-obvious — a hidden constraint, a subtle invariant, a workaround for a specific bug.

If a comment explains what the code does, delete the comment and rename the identifier.

### Don't write defensive code for impossible cases

Validate at boundaries (user input, external APIs). Trust internal code.

Don't catch exceptions just to log them and re-throw. Don't add fallbacks for cases that can't happen.

### Versioning

Semver. `-tools` packages use major bumps for breaking interface changes (e.g., removing an exported function). Minor for new tools/primitives. Patch for fixes.

Adapter packages typically follow the `-tools` minor — but tag separately so we can land adapter-specific fixes without re-tagging `-tools`.

### Test before you ship

Dogfood it. If you can't easily test your change end-to-end (e.g., needs operator interaction), say so in the PR.

### Commit hygiene

- New commits, not amends. If a pre-commit hook fails, fix and create a new commit. Never `--no-verify`.
- Don't run `git add -A`. Add specific files. Avoid accidentally committing `.env`, `node_modules`, etc.

## Filing issues

Helpful issues include:

- What you were trying to do
- What you observed
- What you expected
- Reproduction steps (or "I'm not sure how to reproduce" if you're not)
- Environment: which plugin versions, which runtime, OS

Less helpful: "X is broken" with no detail.

## Discussions and proposals

For non-trivial design proposals (new plugin, new convention, breaking change), open an issue with the `proposal` label before writing code. Architectural questions are easier to resolve before there's a PR in flight.

## Conduct

Be precise. Be direct. Don't waste people's time. Read the docs before asking. When asking, ask the specific question.

## License

By contributing, you agree your contributions are licensed under MIT (or whatever license the specific plugin uses — see its `LICENSE` file).
