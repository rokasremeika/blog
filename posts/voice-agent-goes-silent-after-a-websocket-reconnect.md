> Originally published at https://www.dreamchasers.lt/blog/voice-agent-goes-silent-after-a-websocket-reconnect
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 24 September 2026.

A reconnect restores the transport connection, not the conversational turn. Whatever generation was in flight when the session dropped dies with the old connection. Afterwards, the model waits for input it believes it already consumed, and the caller waits for a reply that no longer exists. Because silence is a stable state, no timeout fires and no error is raised. You fix this by building a watchdog that detects the reconnect edge and issues an explicit reply request to the provider. When building this detector, you must latch on the observed absence of the session to avoid the ABA polling trap.

## Symptoms

- The session reconnects successfully and then nothing happens. The room is alive, audio frames are flowing both ways, the caller says something and the agent does not answer. No error is raised and no timeout fires.
- Around ten minutes into a successful call, the agent spontaneously restarts the conversation. It re-introduces itself, asks its opening question again, and has no memory of anything already discussed. The call never dropped.
- The caller interrupts the agent mid-sentence. The agent stops correctly, and then never speaks again. About 90 s later, something else fires and the call either recovers or ends. During this wedge, frames flow in both directions and nothing has errored.

## What is actually happening

These failures all stem from a mismatch between transport health and turn state. A realtime connection can be perfectly healthy while the conversation sitting on top of it is entirely dead.

When a websocket connection drops and reconnects, the plugin restores the transport. It does not restore the turn. Whatever generation was in flight when the network dropped died with the old connection. Once the transport is back up, neither side is driving the conversation. The model is waiting for input it believes it already consumed. The caller is waiting for a reply that no longer exists. The system is structurally idle, which is why no component throws an error. Silence is a stable state, so it persists until something external breaks it.

The mid-call greeting is caused by provider session management. The provider closes long realtime sessions with a server-initiated session-recycle notice on a roughly ten-minute cadence. The plugin reconnects, but unless session continuation is configured in the setup frame, the reconnect opens a brand-new session with no prior context. The system instructions are still there, so the model does the only thing a cold start allows: it greets. To the caller this reads as the agent forgetting the entire conversation. To the logs, it reads as a completely successful reconnect.

The post-interruption wedge is a failure in generation finalisation. The receive path is gated on the current generation reaching a terminal state. On a caller barge-in, the provider stops generating audio, but the turn-complete event for that generation does not always arrive. The plugin keeps treating the dead generation as in flight. Because the system thinks a response is still active, inbound caller audio is never turned into a new generation request. Nothing errors because the client is simply waiting, and waiting is exactly what it is programmed to do.

## How to confirm it

1. Inspect the setup frame dispatched by the client during the websocket connection phase. Look for the session continuation configuration block. If it is absent, expect the agent to re-greet the caller upon the first session recycle.
2. Log the generation open-to-terminal duration per turn, and count generations that never reach a terminal state. Expect these permanently open generations to correlate directly with the calls that fall silent immediately after a caller barge-in.
3. Record the time from a reconnect edge to the first agent audio on each call. This makes a run with the watchdog off directly comparable to one with it on. Expect to see an infinite gap where the transport is connected but agent audio never follows, confirming the turn is dead.

## The fix

For the reconnect silence, you need a watchdog that detects the reconnect edge. If no agent audio follows within a defined bound, the watchdog issues an explicit reply request to restart the turn.

For the mid-call context loss, configure session continuation in the setup frame. The provider will issue a continuation token the reconnect can present, allowing the reconnect to carry on the same conversation rather than starting a new one. On the plugin version I run, that configuration had to be re-added at the setup-frame patch point because the plugin was not sending it. Then, handle the edge case where the provider rejects the continuation. When it reports that the continuation did not take, clear the per-session state you were carrying and re-drive the conversation deliberately rather than letting the model improvise a second opener.

For the barge-in wedge, implement a finalisation watchdog. This watchdog tracks the open generation and the time since its last activity. If the generation remains open with no activity past a bound, the watchdog finalises it locally and releases the gate. Keep this decision logic as a pure function of timestamps so the boundary cases are unit-testable.

## What I would check first

1. I checked if the silence immediately followed a caller interruption. Because it did, I ruled out a network drop and investigated the finalisation state machine for a missing turn-complete event.
2. I checked the call duration when the context reset. Because the agent re-greeted consistently around the ten-minute mark, I checked the setup frame for a missing session continuation token.
3. I checked the state-change detection logic in the supervisor loop. Because the reconnect detector compared the current session handle to the previous one, I rewrote the detector to fix the ABA problem.

## The polling trap in reconnect detection

When building the reconnect re-drive watchdog, the obvious implementation is a trap. You poll the session handle to detect when it changes. The easiest way to write that detector is to check if the current handle differs from the last one seen. This passes every test, because in a test the reconnect happens once and stays.

Under a real recycle, the network drops and restores rapidly. The handle can go present, absent, and present again between two ticks of the supervisor loop. If the observable value on tick three is identical to the one on tick one, a differs-from-last detector concludes nothing changed. It never issues the re-drive, and the call just sits there in silence.

This is the ABA problem, and here it fails entirely silently. The fix is to latch on the gap. You must record that you observed the absence, and treat the next presence as a new session even when the observable value is identical to the one before it. This requires three lines of state management rather than a single equality check.

Being able to reason about this polling failure came entirely from having the reconnect on a fault injector. I had event logs with monotonic timestamps to replay against, which allowed me to step through the state machine ticks rather than reading the loop and hoping it worked.

## Re-driving is not free

Resumption alone is not enough. A resumed session still needs someone to restart the turn, which is the entire purpose of the re-drive watchdog. If the model managed to resume on its own and you issue a re-drive anyway, the agent will talk over itself. The bound you set before re-driving has to be strictly longer than the provider's worst post-reconnect first-token latency. You have to measure that latency in production before choosing the number.

You also need to instrument the outcome. Emit an event for the reconnect edge, an event for the re-drive decision, and an event for the result. This instrumentation makes a re-drive that talked over a live model distinguishable from a re-drive that successfully rescued a dead call. Record the time from reconnect to first agent audio on each call so a run with the watchdog disabled is directly comparable to one with it enabled.

## Tuning the finalisation bound

The finalisation watchdog requires setting a timeout bound on open generations, and that number is a real tradeoff in both directions. Make it too tight, and you cut off a model that is genuinely thinking or producing slowly. Make it too loose, and you have simply reproduced the 90 s wedge with extra steps.

I observed that when a generation's terminal event never arrives on real calls, something else eventually fires and ends the silence after approximately 90 s. Your local finalisation bound must be significantly shorter than that.

Testing this requires an injector. If you rely on organic barge-ins failing, you will never get enough data to tune the bound safely. You need a test harness that deliberately swallows the turn-complete event on a chosen conversational turn, so the wedge reproduces in seconds rather than waiting for a live incident.

Barge-in is the primary trigger for this failure, which means this bug concentrates on exactly the calls where the caller is most engaged. It is also completely invisible to any health check that looks at the room, the transport, or the participant, all of which remain perfectly fine throughout the wedge. The only observable that tells the truth is the time since the agent last produced audio, and you have to watch it explicitly.

## Common questions

**Why does the voice agent stop responding after a websocket reconnect?**

The reconnect restores the network transport but abandons the conversational turn. The generation in flight died, so the model waits for input and the caller waits for a reply.

**Why does the AI agent greet the caller again ten minutes into a call?**

Realtime providers recycle long sessions around the ten-minute mark. If you do not configure session continuation in the setup frame, the reconnect opens a brand-new session and the model defaults to its greeting.

**Why does the agent go silent for 90 seconds after I interrupt it?**

The provider stops generating audio but fails to send the turn-complete event. The client waits for the dead generation to finish, blocking new input until something else fires roughly 90 seconds later.

**How do I safely detect a reconnect to redrive the agent?**

Poll the session handle and latch on the absence. If you only compare the current value to the previous one, you fall into the ABA trap and miss a reconnect that drops and restores between two ticks.

Canonical URL: https://www.dreamchasers.lt/blog/voice-agent-goes-silent-after-a-websocket-reconnect
