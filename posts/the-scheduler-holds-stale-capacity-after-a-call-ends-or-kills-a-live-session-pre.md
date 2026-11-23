> Originally published at https://www.dreamchasers.lt/blog/the-scheduler-holds-stale-capacity-after-a-call-ends-or-kills-a-live-session-pre
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 3 November 2026.

The scheduler is guessing call states using timeouts instead of observing them. A time-based reconciler is correct for states the scheduler controls entirely, like dispatch wait times, but destructive for states owned by the media server. To fix it, split your reaping logic by state. Stop time-reaping in-flight reservations. Instead, cross-reference them against the media server API. Only force-end a reservation when the media server explicitly confirms the room is gone. If the room is live, extend the expected end timestamp and let the conversation continue.

## Symptoms

* Calls that are genuinely up get force-ended mid-conversation.
* A stale reservation holds capacity long after the media server ended the call.
* The scheduler believes the trunk is full and refuses new outbound dials, despite low actual traffic.
* Database active call counts drift away from the media server room counts in both directions over time.

## What is actually happening

The scheduler uses a time-based reconciler to clean up stuck state. Reconcilers reap on age because age is the only signal available without querying an external system. This works for states the scheduler alone controls, like a reservation sitting in a pre-dispatch queue. It is destructive for states owned by another system. A reservation that is ringing, in flight, or active has its truth held by the media server, not by a timestamp in your database.

Reaping an active call on age is a guess. That guess is wrong exactly when a real call runs longer than your maximum duration estimate. The result is that you break the most successful, longest-running conversations your agent has.

On the flip side, if the media server shuts down a room but the webhook fails to reach the scheduler, the scheduler holds that reservation open indefinitely. If you rely solely on event streams to clear state, a missed event equals a permanent capacity leak. The system accumulates these leaks until it refuses to dial.

## How to confirm it

1. Query your scheduler logs for the exact reason a dropped call was terminated. Expect to find a reconciler or timeout reaper force-killing the session based on a maximum duration threshold, rather than a SIP BYE or room closure event. This confirms your reconciler is blindly reaping active state.
2. Take the room ID of a reservation that the scheduler believes is currently active but has been running for an unusually long time. Run an explicit API call against your media server for that room ID. Expect the room to be absent from the media server's live room list. This confirms the media server tore it down, but the scheduler missed the transition.
3. Count the total concurrent active reservations in your database and compare it to the live room count on the media server. Expect the database count to diverge from the media server room count. A persistent gap confirms you are leaking capacity through missed terminal events.

## The danger of time-based reaping

Age is a local property. You do not need to make a network call to know how old a database row is.

A single call involves a SIP trunk, a media server, and a task queue. The scheduler sits at the top, handing out capacity. When the scheduler hands a slot to the media server, the scheduler no longer owns the lifecycle of that slot. The user and the media server own it.

If you enforce a strict maximum duration on all calls, you guarantee that your best users will have a terrible experience. The reconciler wakes up, sees that the call has exceeded its duration estimate, and terminates the session. The user is mid-sentence, and the line goes dead.

When terminal events are missed, the scheduler keeps the reservation active forever. Eventually, the capacity pool fills up with ghost calls. The scheduler believes the trunk is saturated and refuses to dial new outbound calls, even though the actual system is completely idle.

## How it works

I split the reaping logic entirely based on which system owns the state. States that the scheduler owns and can decide alone are time-reaped against their specific service level. If a reservation never leaves the pre-dispatch state and passes its strict deadline, the reconciler fails it outright. The scheduler knows it never dispatched the work, so it safely cleans it up.

In-flight states are never time-reaped. Instead, they are cross-referenced against the media server's live room list. A force-end on an active reservation happens only when the media server explicitly confirms the room is gone.

When the room is still there and the call has simply outrun its estimated duration, the reconciler extends the expected end timestamp. It also escalates the log level for that reservation from a warning to an error on repeated overruns. That escalation raises the log level and does absolutely nothing else. No alert is raised, and the call is not killed. It makes the wedged case separable from a healthy long call when somebody actually goes looking in the logs.

The reconciliation poll itself runs on a five-minute self-rescheduling cadence. This broad pass catches the silent leaks but is too slow for immediate capacity recovery. To fix the latency, I run a targeted sixty-second fast sweep that only checks reservations that recently changed state. This ensures ordinary drift is repaired in a minute, while keeping the broad API pass cheap.

The instrumentation splits drift events by direction. You track when the scheduler thinks a call is live but the room is gone, versus when a room is live well past the expected end. You track the repeat-overrun count per reservation, which is the exact metric used to escalate a warning to an error. Finally, every repair event includes fast-sweep versus full-poll attribution, so you can tell which loop is doing the actual work.

## What I would check first

1. Are the long calls actually being killed by the reconciler, or is the media server dropping them? Check the termination initiator. If the scheduler killed it on a timeout, the reconciler is at fault. If the media server initiated the disconnect, you have a network or provider issue, not a scheduling bug.
2. Does the fast sweep see the same media server state as the full poll? Compare the results of the sixty-second check against the five-minute pass. If the fast sweep misses state changes but the five-minute poll catches them, your query filter for recently changed reservations is missing state transitions.
3. Is the webhook endpoint actually receiving events from the media server? Check the ingress logs for room closure events. If the events never arrive, your primary signaling path is broken, and the reconciler is acting as the necessary fallback. If the events do arrive but the state stays stuck, your webhook handler is failing to update the database.

## Common questions

**Why did the reconciler kill a live call in progress?**

The scheduler used a time-based timeout to reap long-running calls. When a conversation outran the maximum duration estimate, the reconciler forcefully ended it instead of checking if the media server still had an active room.

**Why does the scheduler think a call is active after the room closed?**

The webhook or event stream that normally signals the end of a call was dropped. Because the scheduler relies purely on incoming events, it never cleared the reservation, resulting in a ghost call holding capacity.

**How do I clear stale reservations holding capacity?**

Run a periodic background task that cross-references active reservations against the media server's live room list. If the media server confirms the room is absent from its live room list, safely mark the reservation as terminal and release the trunk capacity.

**Should I alert on calls that run longer than estimated?**

Do not raise an alert. Escalate the log level from warning to error on repeated overruns, which separates the wedged cases from healthy long calls when you investigate, but avoids waking you up for a successful conversation.

Canonical URL: https://www.dreamchasers.lt/blog/the-scheduler-holds-stale-capacity-after-a-call-ends-or-kills-a-live-session-pre
