# Misrouted RPC: answer it, don't drop it — RFC

**Status:** Proposal · **Date:** 2026-09-21 · **Authors:** Fondant (Toolsmith), Brioche (Engineering Director)
**Component:** `wire-tools` `RpcResponder` · **Sibling of:** `2026-09-21-restart-guard-resume-ungating.md`
**Deliberately not bundled:** that RFC is a rate-bound policy question; this is a protocol question.

## TL;DR

`RpcResponder.handleEvent` rejects on **topic** before it looks at the envelope:

```js
if (normalizeTopic(event.topic) !== RPC_REQUEST_TOPIC) return false;   // "rpc.request"
```

So a frame on the **right** topic with a **bad envelope** gets a helpful error reply (added
2026-09-11, j:1480) — and a frame with a **perfect envelope** on the **wrong topic** gets total
silence. The caller cannot tell "refused" from "queued" from "never arrived".

**Proposal: reply to an unambiguously misrouted RPC, naming the topic. Execute nothing.**

## What it cost, 2026-09-21

The ED sent five `crew.agent_resume` calls for one lane. #1 and #2 went out on `rpc.request` and
were processed (both refused by an unrelated rate bound). **#3 and #4 went out on `ipc` and were
dropped in silence** — no reply, no log entry naming them, no execution. He filed
`crew.agent_resume` as a broken tool, having built a 360-second watch with a live positive control
that faithfully waited for a launch the service never attempted. #5, re-sent on `rpc.request`,
worked first time.

Diagnosis took two people and ~25 minutes, and required reading the deployed responder source,
because the only trace was `ignored non-rpc frame` with topic and source and nothing else.

Nothing here was a bug in the caller's envelope. It was a one-word routing mistake that the
protocol had no way to report.

## Proposal

When a frame is **unambiguously a misrouted RPC**, reply with an error naming the topic:

- topic does not normalize to `rpc.request`, **and**
- payload carries a complete envelope — `rpc.id` **and** `rpc.reply_to`, **and**
- `method` is a string **this responder actually implements**

then send `ok:false, error: "…sent on topic 'X'; RPC requests must be published on 'rpc.request'; nothing executed"`
to the frame's **verified `event.source`**, and return `true`.

Anything failing those tests keeps today's behaviour exactly: return `false`, fall through, get logged.

## Why this shape, and not a broader one

⚠️ **The obvious broader rule — "reply to anything rpc-shaped on any topic" — is a
reply-amplification surface.** The frame is by definition unsolicited and its fields are
attacker-choosable, so a permissive rule lets any sender make the responder emit traffic. The
existing malformed-envelope path already guards this (`isSafeReplyTopic`, 2026-09-15: routing goes
to the verified source, never to a supplied `reply_to`, and the reply topic must be plain, short
and unreserved). This proposal inherits those constraints unchanged and adds two more —
a complete envelope and a **known method** — so the surface is narrower than the path that already
exists, not wider.

**Requiring a known method is the load-bearing constraint.** It means the sender demonstrably knows
this responder's interface; a scanner spraying frames gets silence, as today.

⛔ **This does not widen the envelope contract.** Brioche's 09-18 rule stands: the accepted request
shape is unchanged, and nothing new executes. It widens only **where we are willing to answer**.

## Already shipped, and not a substitute

`crew-service`'s `ignored non-rpc frame` log now carries `seq`, `method`, `rpc_id` and a hint naming
the topic mismatch (2026-09-21; effective next restart). That makes a dropped frame *diagnosable by
an operator with root* — it does nothing for the **caller**, who still sees silence and cannot read
that log. Observability for us is not an answer for them.

## Alternatives considered

- **Accept RPCs on any topic.** Rejected: the topic is the only cheap discriminator between an RPC
  and ordinary IPC, and every agent's chat traffic shares the `ipc` topic.
- **Log only** (status quo + the shipped fields). Rejected as sufficient: it leaves the caller
  blind, which is the actual failure. Retained as the fallback for frames that fail the tests.
- **Client-side guard** — have the RPC client refuse to publish on a non-`rpc.request` topic.
  **Worth doing as well**, and strictly better because it fails at the source; but it cannot help a
  caller on an older client, which is precisely the case that bit us (crew-service runs wire-tools
  2.17.1 against a 2.22.1 repo).

## Risk if we do nothing

This failure is silent, and it is indistinguishable at the caller from a refusal or a queue. It
cost an Engineering Director four attempts and a false tool-broken report in one morning. The next
occurrence will present exactly the same way.
