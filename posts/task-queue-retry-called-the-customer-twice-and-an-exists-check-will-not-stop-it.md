> Originally published at https://www.dreamchasers.lt/blog/task-queue-retry-called-the-customer-twice-and-an-exists-check-will-not-stop-it
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 22 September 2026.

An at-least-once task queue retrying a telephony placement can ring a person twice, even if you check whether a call record already exists. This happens because the platform's native webhooks can create the call record before your dial RPC returns, and because a queue's retry timeout often fires while the first attempt is still ringing. To fix it, you need two markers. First, a post-placement marker written only by your dial code to prove the call happened. Second, a pre-placement attempt marker that locks the number for a window sized to your ringing timeout plus the queue's acknowledgment budget.

## Symptoms

- An at-least-once task queue retries the outbound dial task, and the same person is called twice within a minute.
- The queue metrics show only one logical task being processed, but the carrier logs show two distinct INVITEs leaving your infrastructure.
- The guard clause most people reach for (checking if the call record already exists before dialing) either never fires, or it fires on the first attempt and blocks a call that was never actually placed.
- The trap worth stating plainly: "does a record for this call exist" feels like idempotency and is not, as soon as more than one component can create that record.

## What is actually happening

Two separate problems stack together to defeat a basic existence check. First, the call record is rarely authored by a single writer. The platform's native room-started webhook is fast and can easily create the record before the dial RPC returns. Because webhooks and the dial process run concurrently, the record existing is not evidence that a dial happened. An existence check in this state is simply reading a marker somebody else wrote.

Second, the dangerous window occurs exactly between the moment we start placing the call and the moment we record the successful placement. Outbound ringing takes tens of seconds. A task queue that retries based on an acknowledgment timeout or visibility budget will re-enter the dial pipeline exactly during this window. Because the first attempt has not returned yet, nothing has been written to confirm placement. By definition, the existence check allows the second attempt through, resulting in a duplicate call.

## How to confirm it

1. Log every queue redelivery of a dial task as an explicit event. This makes retries visible as events rather than being inferred from duplicate INVITEs.
2. Correlate the webhook arrival time with the dial RPC return time for a single session. Expect to see the platform's room-started webhook arriving and writing to your database before the dial RPC completes its execution, proving that record existence is untrustworthy as an idempotency key.
3. Measure the time from the attempt marker being written to the placement marker being written, per call. Count attempts refused by the in-flight window separately from attempts refused as already placed.

## How it works

To prevent double-dials under queue retries, I use two markers with entirely different jobs. Relying on document existence is fundamentally flawed when multiple writers are involved. You have to pick fields that only the dial path can write, and check those fields exclusively.

First, a post-placement marker serves as the final proof that a dial actually happened. This is written only by the dial path, and only after the placement RPC returns successfully. You check this field to know if the side effect is complete, never the bare document existence.

Second, a pre-placement attempt marker closes the in-flight window. This is written immediately before firing the RPC. It acts as a bounded lock: any attempt that starts inside a specific validity window after an existing attempt marker is refused. I size that window to the 45 s ringing timeout plus the queue's acknowledgment budget. This guarantees that a genuinely in-flight attempt blocks a retry, while a dead attempt eventually releases the lock instead of wedging the phone number forever.

Enforce the same window at webhook-processing time, so the guard does not depend on which path runs first.

This is a general rule I apply to any at-least-once queue placed in front of a side effect that costs money and reaches a human. The marker that proves the side effect happened must be written by the code that performed it. The window between starting the action and recording it needs its own bounded lock sized from the transport's real duration plus the queue's redelivery budget.

## What I would check first

1. Check the timestamp of the platform's room-started webhook against the return time of the dial RPC. A webhook that writes the call record before the RPC returns rules out document existence as a reliable proof of placement.
2. Compare the task queue's visibility timeout against your hard ringing timeout. A visibility timeout shorter than 45 seconds rules out the queue's native retry being safe for outbound dialling, because the queue will inevitably redeliver tasks while the phone is still ringing.
3. Review the exact order of your outbound placement operations. Room creation that happens after the SIP participant is created rules out being able to cleanly correlate native platform webhooks, because those webhooks will arrive before your database has the routing metadata they need.

## Ordering the placement operations

Outbound placement consists of three operations, and their order is load-bearing. Getting the order wrong guarantees downstream failures that look like idempotency bugs or leaked resources.

Create the room first, and carry the routing metadata the webhooks will need directly in the room creation request. Creating the room without metadata means the platform's native room and participant webhooks arrive carrying only platform identifiers. They know the room, but they know nothing about your domain model. Anything that tries to resolve the tenant or number from a webhook has nothing to resolve from, forcing expensive database round trips. When you supply the metadata upfront, the webhook handler becomes a pure function of the event.

Create an explicit agent dispatch next. Doing this ensures a worker is in the room before the phone actually rings. Dispatching the agent after the SIP participant means the worker can still be starting when the callee picks up. If the person answers to two seconds of nothing before the agent starts existing, the caller experience is broken from the first word. You can read more about resolving the timing of the first word in [Outbound call greets before the callee's audio is subscribed](/blog/outbound-call-greets-before-the-callees-audio-is-subscribed/).

Create the SIP participant last, with wait-until-answered, dial tone playback, and a 45 s ringing timeout. Give it a deterministic participant identity derived from the session id so every later event can be correlated without a lookup.

From the moment the room exists, you own it. Wrap everything after room creation so that every failure branch deletes the room before returning the error. Any failure after that point (a placement error, a classification error, an unexpected throw) leaks a room that a worker will occupy until the empty timeout expires. I log room creation and room deletion as paired events, so an unmatched creation is a highly visible leaked failure branch.

## Safe retries versus dangerous double-dials

Retrying a placement is only safe when the failure provably happened before any INVITE left the box. This interacts directly with stale-trunk self-healing.

SIP trunk state is often held in a store that is not necessarily durable across restarts. A Redis instance without persistence configured is a common shape. The platform mints new trunk ids when it comes back online. Every id you cached in process memory or persisted next to a phone number now points at nothing. The failure is invisible until someone tries to dial, and because the error is a validation error rather than a call outcome, it is easy to classify as a bad number instead of broken infrastructure.

I built a narrow self-heal for this exact scenario. On that one specific error, I evicted the cached id, re-resolved the trunk by its stable name, and retried the placement exactly once. The narrow match is a strict safety property. That specific error is raised during request validation before any signalling occurs, so a retry provably cannot ring a person twice. If you widen the match to catch generic connection timeouts, you have built a double-dial machine.

You can read the full teardown of that specific error state in [Outbound Calls Fail With Trunk Does Not Exist After A Restart](/blog/outbound-calls-fail-with-trunk-does-not-exist-after-a-restart/). The question that decides whether an automatic retry is safe is always the same: can this failure have had a side effect before it was raised?

## Common questions

**Why did the queue retry while the phone was still ringing?**

The queue's acknowledgment budget or visibility timeout is shorter than the carrier's ringing timeout. If a call rings for 45 seconds but the queue expects an acknowledgment sooner, it will assume the worker failed and redeliver the task.

**Can I just check if the call record exists before dialing?**

No. Platform webhooks often create the call record in your database before the dial RPC completes. Checking for record existence might read a marker written by a webhook, leading you to block a call that was never actually placed.

**How do I prevent a failed attempt from locking out the number forever?**

Size the pre-placement lock to a bounded window. Combine the 45 s ringing timeout and the queue's acknowledgment budget. If the lock is older than that window, the previous attempt is genuinely dead and you can safely retry.

**Is it ever safe to retry a failed SIP placement?**

Retrying is only safe if the failure provably occurred before any SIP signalling left your infrastructure. Validation errors like a missing trunk identifier are safe to retry, while carrier timeouts are not.

If the boundary this note describes is one you would rather own, [taking ownership of the infrastructure underneath](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/task-queue-retry-called-the-customer-twice-and-an-exists-check-will-not-stop-it
