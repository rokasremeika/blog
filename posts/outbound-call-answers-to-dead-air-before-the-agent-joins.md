> Originally published at https://www.dreamchasers.lt/blog/outbound-call-answers-to-dead-air-before-the-agent-joins
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 6 October 2026.

Getting the order of outbound placement wrong causes dead air, unresolvable webhooks, or leaked rooms. The fix is ordering three operations strictly. First, create the room with your custom routing metadata so webhooks can resolve the tenant. Second, dispatch the agent so the worker is fully joined before the phone rings. Finally, create the SIP participant to dial the number, using a 45 s ringing timeout. Wrap the entire flow in a block that catches any error after the room exists and explicitly deletes the room before returning.

## Symptoms

- The callee answers the phone but hears two seconds of dead air before the agent starts speaking.
- Your webhooks arrive with a room identifier but no custom metadata, leaving you no way to tell which tenant or phone number the call belongs to.
- A failed SIP dial leaves an empty room behind, and a worker sits idle in that room until the empty timeout expires.

## What is actually happening

Outbound SIP placement in a realtime system requires three distinct operations: creating the room, dispatching the agent, and creating the SIP participant. The sequence you use to execute these operations is load-bearing. Getting the order wrong causes race conditions at the start of the call and leaks infrastructure when calls fail.

If you dispatch the agent after the SIP participant is created, the dial is already in flight. A carrier network can route and connect a SIP call very quickly. By the time the callee picks up the phone, the agent worker can still be starting. The greeting gate logic cannot help you here, because there is nobody in the room to evaluate the gate. The callee says hello into a void, waits two seconds, and then the agent abruptly joins and starts speaking.

Additionally, creating the room without metadata means the platform's native room and participant webhooks carry only platform identifiers. Platform native room and participant webhooks carry only platform identifiers. They know the internal room identifier, not your domain model. Any downstream service that attempts to resolve a tenant or phone number from a webhook event will fail because it has nothing to resolve from.

Finally, resource ownership is strict. From the moment a room exists, you own it. Any failure after that point requires explicit cleanup. If you experience a placement error, a classification failure, or an unexpected exception before the dial succeeds, that room is left hanging. A worker will occupy it and wait for activity until the default empty timeout expires. This leaks a room that a worker will occupy until the empty timeout expires.

## How to confirm it

1. Log the time difference between the agent dispatch creation and the worker joining the room, then compare it against the time between SIP placement and the callee answering. Expect the dispatch-to-join duration to be strictly shorter than the placement duration. If the worker joins after the answer event, your placement order is causing the dead air.
2. Query your webhook receiver logs for events that fail to resolve a tenant or session identifier. Expect to find these unmatched events correlated with outbound calls where the room was created without metadata.
3. Search your application logs for room creation events without a corresponding room deletion event on a failed call branch. Expect to find leaked rooms that remained open until the generic worker empty timeout finally destroyed them.

## How it works

The fix is to enforce a strict ordering for resource creation and to establish explicit ownership over the room lifecycle. I organize the outbound placement pipeline to always execute in this exact sequence.

First, I create the room explicitly. This is the only way to carry the routing metadata the webhooks will need. By passing a JSON payload containing the tenant identifier, session identifier, and destination number into the room creation request, every subsequent webhook emitted by the platform will include this context. Your webhook handlers no longer need to hit a database to figure out who the call belongs to.

Second, I create an explicit agent dispatch. Initiating the dispatch immediately after the room is created ensures a worker is assigned and joins the room before the phone ever rings. The agent is already sitting in the room, waiting for the media tracks to arrive.

Third, I create the SIP participant. I configure this with wait-until-answered enabled, dial tone playback on, and a 45 s ringing timeout. I also assign a deterministic participant identity derived directly from the application session id. This means every later event involving this participant can be correlated directly back to the database row without any secondary lookups.

Crucially, I wrap everything after the room creation in an error handling block. Because the system acquires ownership of the room in the first step, every failure branch must delete the room before returning the error. If the agent dispatch fails, if the SIP participant rejects the dial, or if an internal API throws an exception, the catch block intercepts it, issues a room deletion request, and then bubbles the error up.

## What I would check first

1. Check the timestamp of your agent dispatch request against your SIP participant creation request. This discriminates between an initialization race and a media subscription delay. If the dispatch request fires after the SIP creation request, the agent is late to the room and you must fix the ordering. If the dispatch happens first, the dead air is likely an audio subscription timing issue.
2. Inspect the metadata field on the earliest webhook event you receive for a new call. This isolates whether you are relying on implicit or explicit room creation. If the metadata field is empty, the room was created implicitly by the platform and you must switch to explicit room creation to pass your domain identifiers.
3. Search your application logs for room creation events on failed calls and check whether a deletion event was emitted. This discriminates between an unhandled exception during placement and an orphaned worker caused by a downstream teardown failure. If the logs show a room being created but no subsequent deletion log within the same request lifecycle, your error handling is leaking infrastructure.

## Why the greeting gate cannot save a late agent

You might already have logic that gates the initial greeting to prevent it from playing into a cold connection. I use a 2.5 s answer grace window combined with an audio track subscription trigger to solve the problem where an [outbound call greets before the callee's audio is subscribed](/blog/outbound-call-greets-before-the-callees-audio-is-subscribed/). However, that gate solves a signaling issue, not an agent lifecycle issue.

If the SIP participant is created before the agent dispatch, the agent process is simply not running when the SIP leg connects. The greeting gate logic lives inside the agent code. If the agent is not in the room yet, it cannot evaluate track subscriptions, it cannot enforce a 2.5 s grace period, and it cannot observe the active signaling state. By the time the agent finally joins, the initial SIP events have already passed. Because the worker is still starting when the callee picks up, no agent is present to execute the greeting gate. This results in the callee hearing two seconds of absolute silence before the greeting suddenly fires.

## Deriving identity for deterministic correlation

Generating identifiers at the correct boundary reduces operational overhead when debugging a failed call. If you allow the realtime platform to generate the participant identity for the SIP leg, you are forced to store that identifier and use it as a lookup key for all future webhook events.

Instead, I derive the SIP participant identity directly from the application database session id. During the third step of the outbound placement pipeline, I format the session id into the participant identity field. When a webhook arrives reporting that the SIP participant disconnected, the webhook payload includes that deterministic identity. The webhook receiver correlates the event directly to the session without an external lookup. It never needs to query a mapping table.

## Protecting capacity with strict teardown

An unhandled exception during outbound placement might seem like a minor transient error, but in a realtime voice stack, it directly impacts capacity. A room created without a SIP participant is an active resource rather than an inert record. It actively holds a worker assignment.

When the agent dispatch succeeds but the SIP dial fails, the worker process is commanded to join the room. The worker joins, observes that it is alone, and waits for instructions. Without explicit cleanup, that worker will sit idle until the platform's empty room timeout eventually evicts it. If a misconfigured destination number causes a burst of placement failures, you can exhaust your entire worker pool with idle agents sitting in leaked rooms.

Treating resource creation as an acquisition of ownership solves this. The function that requests the room must guarantee its destruction on every exit path.

## Common questions

**Why does the person hear dead air when answering my outbound call?**

The LiveKit agent was dispatched after the SIP dial was initiated. By the time the person answered, the agent process was still initializing and not yet in the room to speak.

**Why are my LiveKit webhooks missing the tenant metadata?**

The room was created without metadata. You must explicitly create the room first and pass your domain metadata in the creation request.

**Why are empty rooms staying open after failed outbound dials?**

An error occurred during outbound placement after the room was created, but before the dial succeeded. You must catch these placement errors and explicitly delete the room.

If the boundary this note describes is one you would rather own, [a self-hosted deployment you control](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/outbound-call-answers-to-dead-air-before-the-agent-joins
