# Restart guard: un-gate `agent_resume` — RFC

**Status:** Proposal · **Date:** 2026-09-21 · **Authors:** Fondant (Toolsmith), Brioche (Engineering Director)
**Amends:** the restart-guard spec signed by Tim + Brioche, 2026-08-19
**Decision owner:** Tim — this RFC does not change a signed policy, it asks for one to be changed.

## TL;DR

The restart guard rate-bounds a grantee to N lifecycle actions per window. **`agent_stop` and
`agent_close` consume a slot; `agent_resume` is gated by the budget but never consumes or
replenishes it.** The consequence is that a stop which reaches the bound makes its own matching
resume impossible until a slot ages out — the guard permits the first half of park-and-resume and
then forbids the second half, with no signal to the caller.

**Proposal: stop gating `agent_resume` on the stop budget.** Keep every other restart-guard check on
resume — never-targets, spawner-only scope, manifest checks. Only the rate bound is removed.

## The mechanism

`crew-service/src/methods.ts`, in `requireRestartGuard`:

```js
if (!decision.allowed) throw new Error(`${method} refused: ${decision.reason}`);
if (method === "agent_stop" || method === "agent_close") {
  hist.push(now);
  guardActions.set(actor, hist);
}
```

The bound is checked for every guarded method — `agent_stop` (832), `agent_close` (977),
`agent_resume` (1365) — but only stop and close are recorded. `agent_launch` is not guarded at all.

So resume is subject to a budget it can neither spend nor refill.

## What it cost, 2026-09-21

Twice in one morning, on a live fleet:

- **11:42–11:47Z.** Three RPCs refused at the bound. The ED could not see the refusals (see
  *Aggravating factor*), read them as queued, and held two lane spawns to avoid double-spawning a PR.
- **12:10:56Z.** A stop consumed the slot that had freed at 12:09:56Z, putting the grantee at 5/5.
  Both subsequent resumes of that same lane were refused **by construction**. The ED filed
  `crew.agent_resume` as a broken tool, having built a 360-second watch with a positive control
  that faithfully waited for a launch the service never attempted.

Neither was misuse. In both cases the operator did exactly what the guard's own model of
park-and-resume invites, and the guard made the second half unreachable.

## Why un-gating resume is the right shape

1. **A resume cannot be the runaway the guard exists to stop.** The stated rationale is *"a runaway
   remediation loop is worse than a stalled lane."* A loop needs a repeatable destructive step;
   stops remain bounded, so the cycle rate of any stop/resume loop is still capped by the stop budget.
2. **Blocking resume inverts the intent.** Resume is the action that *undoes* a stop. The guard
   currently protects the fleet from recovering.
3. **A reserved recovery slot is not sufficient** (Brioche): one reserved slot still leaves a
   two-stop burst unable to recover its second lane. The asymmetry has to go, not be padded.
4. **Duplicate spawning is already prevented independently.** `Orchestrator.resumeAgent` refuses a
   live lane outright — *"agent 'X' is already running (screen 'Y')"* — so an un-gated resume
   cannot fan out duplicates. **Verified in source, not assumed**; this is the safety premise the
   proposal rests on and it holds without the rate bound.

## Residual risk, stated plainly

Un-gating resume leaves one loop the stop budget does not cap: **a lane that dies on its own,
repeatedly, being resumed each time.** No stop is consumed, so nothing throttles it.

This is a real gap and the proposal does not close it. Two options if it matters to Tim:

- **Separate, generous resume budget** (e.g. 20/window) — preserves a backstop without making
  ordinary park-and-resume unreachable. Costs a second counter.
- **Accept it**, on the grounds that a crash-looping lane is already visible through other
  instrumentation and a resume of a dead lane is cheap.

Fondant's recommendation is the separate generous budget if a backstop is wanted at all; Brioche's
is plain un-gating. Both of us prefer either to the status quo.

## Aggravating factor, tracked separately

Neither incident would have cost an hour if the refusal had been *visible*. `rpc.reply` frames from
crew-service do not reach a Claude persona session at all — measured on two independent sessions,
zero deliveries since boot. The service answered every call correctly and no answer arrived. That
defect is being fixed on its own track; this RFC does not depend on it, but the two together are
what turned a correct policy decision into a tool-broken report, twice.

## Not proposed here

- No change to never-targets, spawner-only scope, alarm gating, or the stop/close bound itself.
- No change to `agent_launch`, which is unguarded today. Whether that is intended is a separate
  question worth asking, and this RFC deliberately does not bundle it.

## Observability, already shipped

`crew-guard-budget.sh` (+ `sudoers.d/96`) lets the grantee read the budget directly: actions used,
the window, which actions counted and when each frees, and the counting rule inline. Root-held and
argument-free because the audit log is `0700 tim`; verified runnable *as the grantee*, not only as
root. That removes the need to ask the Toolsmith what the guard thinks — but it does not fix the
asymmetry, which is what this RFC is for.
