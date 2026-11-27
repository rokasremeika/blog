> Originally published at https://www.dreamchasers.lt/blog/a-scheduled-call-needs-an-explicit-transition-table
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 7 November 2026.

When a dispatcher, a dial RPC, a media server webhook, and a reconciliation loop all write to the same call status field, the last writer always wins. Webhooks are delivered at least once and unordered, so late events can overwrite terminal states and resurrect completed calls. To stop this, model the call lifecycle as a single explicit transition table. Route every update through a guarded write path, reject any transition not explicitly defined in the table, and make terminal states truly immutable so duplicate events become harmless no-ops.

## Symptoms

- A scheduled outbound call sitting in a state nothing owns, showing as ringing hours after the line went dead.
- A completed call flipping back to active when a late webhook lands.
- Two handlers writing conflicting statuses within the same second, resulting in the last writer winning regardless of logic.

## What is actually happening

A scheduled call is touched by at least four independent writers during its lifecycle. The task queue dispatcher claims the reservation, the dial RPC places the outbound leg, the media server sends room webhooks, and the reconciliation loop cleans up lingering database rows.

If each of these independent writers sets a status field directly, the call lifecycle is only implicit. Ordering across these distributed systems is merely a hope, not a guarantee. Every out-of-order event and every duplicate webhook becomes an unguarded write directly to your database.

Webhook delivery is at-least-once and unordered by design. The media server does not guarantee that a tear-down event will arrive after a participant-joined event. Because the stream is unordered, adopting a "later message wins" strategy is wrong by construction. It applies a sequential assumption to a fundamentally asynchronous stream, guaranteeing that delayed events will overwrite the current truth.

## How to confirm it

1. Extract the event logs for a call that resurrected from a completed state to an active state. Look at the timestamps. Expect to see a delayed media server webhook arriving seconds or minutes after the tear-down event, which confirms the system is blindly accepting writes.
2. Query your task queue and webhook handler logs for the same call ID within a one-second window. Expect to find overlapping writes where the final database status matches the slowest network commit rather than the actual sequence of events.
3. Count calls stuck in a ringing or dispatching state for more than an hour. Expect a non-zero count, confirming that state writes are being dropped, overwritten, or stalling without a mechanism to advance or terminate them.

## How it works

I model the call lifecycle as one explicit transition table with a single guarded write path. Instead of applying every update that arrives, the system validates the requested transition against the explicit table. It rejects any transition not found in the table instead of applying it.

The state machine runs on five live states and five terminal states. The live states progress predictably from pre-dispatch to dispatching, ringing, and active, eventually reaching completed.

The five terminal states accept no further transitions at all. They are strictly immutable. When a late duplicate webhook fires against a terminal reservation, the transition check fails. The event becomes a logged no-op rather than a resurrected call, preserving the integrity of the completed state.

## What I would check first

1. Check whether terminal states accept updates. Inspect whether transition rejections cluster on dispatching-to-active pairs rather than terminal states. This discriminates between a missing legal shortcut and out-of-order webhook delivery landing after call teardown.
2. Check how concurrent writes are handled at the database level. Review the SQL or ORM update path for the status field. If calls sit in ringing past the 45 s timeout with zero rejected transitions, an upstream writer stalled rather than losing a write race.
3. Check the event delivery guarantees published by your media server. Measure the duration calls remain in dispatching and ringing against their configured timeout. This discriminates an abandoned or stalled upstream writer from concurrent handler contention.

## The four independent writers

The task dispatcher initiates the process by claiming a call from the database. It updates the state to dispatching, signaling to other workers that the task is owned.

Immediately after, the dial RPC negotiates with the carrier to place the leg. Once the carrier signals progress, the RPC updates the database state to ringing.

Simultaneously, the media server observes the realtime room. When a participant joins or room state changes, the media server emits webhooks to the application. These webhooks attempt to move the state to active or completed.

Finally, a background reconciliation loop sweeps the database to catch failures. It checks for stalled calls and attempts to terminate them.

Every one of these writers operates with different latency profiles.

## Designing the explicit transition table

Fixing the race condition requires removing raw write access. The update path must become a requested transition from a known current state to a desired next state.

The implementation is a strict mapping of allowed paths. A pre-dispatch call can become dispatching. A dispatching call can become ringing. A ringing call can become active. An active call can become completed.

When a component requests a state change, the database query enforces the transition table using a conditional update. The SQL UPDATE statement includes a status condition matching the expected previous state. If the row has already progressed, the conditional update modifies zero rows, and the application knows the transition was rejected.

The five terminal states act as a one-way door. Once a call hits a terminal state, there are no valid outgoing transitions defined in the table. The conditional update will always fail, permanently locking the row against late webhooks.

## Documenting the shortcut transition

You cannot design a rigid state machine for distributed systems without acknowledging dropped signals. Strict adherence to a linear path will break legitimate calls when the underlying platform misses an event.

There is exactly one documented shortcut transition in my table: moving straight from dispatching to active, skipping the ringing state entirely.

This shortcut is mandatory because the media server event stream does not guarantee a complete sequence. The room webhook can report the human participant joining without a matching agent-join event ever appearing on the stream.

If the transition table rigidly demanded a ringing state first, it would reject the active update. The call would stall in dispatching forever, even though the human participant joined successfully.

Writing this shortcut directly into the table, with the reasoning documented next to it, is a structural defense. It stops a future engineer from reviewing the explicit paths, assuming the shortcut is a bug, and fixing it back into a rejected transition.

## Instrumenting the rejected transitions

An explicit state machine transforms invisible race conditions into measurable data points. Every rejected transition is an observable event that tells you how the surrounding systems are behaving.

I instrument the count of rejected transitions, keyed explicitly by the attempted from-state and to-state pair. This specific metric is exactly where out-of-order webhook delivery becomes visible. A spike in active-to-ringing rejections immediately highlights that the dial RPC is lagging behind the media server webhooks.

I also track the time spent in each non-terminal state. A state that no writer is advancing becomes visible without custom alarm configurations. If a call sits in a ringing state for more than 45 s, the transition table is operating correctly, but the upstream signal that should advance it has failed. Reaping these stalled states safely requires splitting the reconciliation logic, as detailed in [the scheduler holds stale capacity after a call ends or kills a live session prematurely](/blog/the-scheduler-holds-stale-capacity-after-a-call-ends-or-kills-a-live-session-pre/).

## The guarded write path

Enforcing the transition table must happen at the data store level. Attempting to validate the transition in application memory introduces a race condition. Two webhook handlers will read the same active state, both will validate their transition locally, and both will overwrite the database sequentially.

The single guarded write path relies on the database transaction enforcing the transition check. If the transaction layer is highly contended, probing before opening the transaction can save database resources. Evaluating the state cheaply before committing to an atomic check follows the same pattern used when [task queue dispatch exceeds capacity despite pre-check](/blog/task-queue-dispatch-exceeds-capacity-despite-pre-check/).

The core principle remains identical: the database must verify the current state atomically before committing the transition.

## Common questions

**Why does my completed call change back to active?**

A delayed webhook from the media server arrived after the call tear-down event. If your status field accepts blind writes, this late event overwrites the completed state.

**How do I handle concurrent call state updates?**

Model the call lifecycle using an explicit transition table. Enforce the update with a conditional database write that checks the current state before applying the new one.

**Should the system reject out-of-order webhooks?**

Yes, out-of-order events should be rejected if they violate the transition table. A late webhook against a terminal state should be logged as a no-op instead of altering the call.

**Why skip the ringing state entirely?**

The media platform may emit a participant joined event without ever sending an agent join event. Allowing a direct transition from dispatching to active prevents these calls from stalling.

Canonical URL: https://www.dreamchasers.lt/blog/a-scheduled-call-needs-an-explicit-transition-table
