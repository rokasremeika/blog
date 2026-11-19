> Originally published at https://www.dreamchasers.lt/blog/completed-calls-are-recorded-as-hangups-because-the-disconnect-wins-the-write-ra
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 28 October 2026.

Your analytics over-report caller hangups because two asynchronous events race to write the session end reason. When a call completes naturally, the agent lifecycle sets a success reason. However, the caller usually hangs up immediately after the final prompt. This triggers a SIP disconnect event, which arrives milliseconds later and overwrites the success state with a hangup reason. To fix this, replace last-write-wins with a precedence rule. Once a terminal reason is set by the agent lifecycle, make the database ignore subsequent disconnect events. The first writer with a valid reason must win.

## Symptoms

- Calls whose transcript shows the agent delivering its closing line and finishing properly are stored with a hangup reason.
- The call outcome field systematically over-reports the person hanging up, skewing overall analytics.
- Every downstream conclusion built on the end reason is wrong, because the telemetry is corrupted at the source.
- The wrongness is systematically biased toward the calls that went perfectly well, masking actual success rates and causing you to chase false drop-offs.
- Every downstream conclusion built on the end reason is systematically biased toward misclassifying successful calls as user drop-offs.

## What is actually happening

Two writers can set the reason a call ended, and they are racing to update the same database row.

The agent's own lifecycle state machine sets a specific reason when it finishes properly. When the agent delivers its final sentence and the conversational logic decides no further turns are expected, it pushes a terminal success status to the call record.

Separately, a hangup watcher fires when the far end disconnects. At the SIP layer, a network `BYE` message arrives, tearing down the media session and triggering a participant-disconnected event.

On a real call, the human hangs up the instant the agent stops talking. This physical response happens extremely fast. The caller hits the red button often within the exact same second the audio playout ends, sometimes before the closing bookkeeping from the agent lifecycle has fully landed in your persistent storage.

The disconnect writer, reacting to the SIP `BYE`, arrives last. If your database uses a basic last-write-wins update strategy, the disconnect event overwrites a reason that was already correct.

Last-write-wins turns the phrase "the call completed" into "they hung up". The calls most likely to be misattributed are the ones that reached a natural ending, because those are exactly the ones where the person hangs up immediately after the final word. A call where the user gets frustrated and hangs up mid-sentence is correctly labelled as a hangup. A call where the user waits politely and hangs up at the end is incorrectly labelled as a hangup.

## How to confirm it

1. Record the time between the lifecycle reason being set and the disconnect event arriving on a per-call basis. Expect to see sub-second gaps on the calls marked as hangups that have complete transcripts. A gap in the tens or hundreds of milliseconds confirms the two events are reacting to the exact same conversational milestone.
2. Compare the final transcript state to the end reason in the database. Expect a mismatch where the transcript has a natural closing line (like a goodbye) but the database reason is recorded as a caller disconnect.
3. Count how many times an update to the end-reason field replaces an existing value for the same call ID. Expect this counter to increment precisely on the calls that ended cleanly, which is the direct measure of how often the race fires in production.

## How it works

I designed the telemetry system to make the close reason single-writer by precedence rather than by timing.

The fix is defining which reasons are terminal, and allowing only a transition from a no-reason state to a terminal reason. If the agent lifecycle has already ended with a valid reason, that reason wins. The database update logic explicitly checks the existing state of the row. If a terminal reason is already present, the new write is discarded.

The disconnect path may still record that the far end went away in a separate connection-state field. You still need to know when the socket actually closed for infrastructure reasons. But it may not replace an existing terminal reason for the business outcome of the call.

First writer with a real reason wins, and the last writer never wins by virtue of being last.

This pattern generalises across realtime systems. Any per-entity outcome field written by more than one asynchronous source needs a precedence rule, not a mutex. A mutex only ensures the database does not corrupt the row during simultaneous writes. It does nothing to prevent a logically older event from overwriting a logically newer one. This is the difference between telemetry you can make decisions from and telemetry that quietly encodes a race condition.

To ensure this behaviour does not regress when new close reasons are added, I assert it in a unit test. The test delivers the two events (lifecycle completion and SIP disconnect) to the state machine in both possible orders. The assertion requires that the first valid terminal reason survives, and any subsequent reason is silently dropped.

## What I would check first

1. Look at your database update queries for call completion. If the transcript terminates before the agent reaches its closing line, the caller genuinely hung up mid-conversation rather than suffering a late disconnect write race.
2. Inspect the timestamp delta between the last agent audio frame and the incoming participant disconnect webhook. If they are nearly identical, it confirms the caller hung up in response to the closing prompt. (If you notice the agent's last sentence is actually truncated on the caller's side before they hang up, you might have a playout sleep issue instead. See /blog/agent-last-sentence-is-cut-off-and-the-room-outlives-the-call/ for that diagnosis).
3. Check whether an unhandled provider error terminated the session before the agent finished talking. If a mid-session provider error occurred, classify whether the session error terminated the call prematurely rather than letting the lifecycle complete cleanly.
4. Review your session finalisation logs. If the disconnect event arrived before lifecycle bookkeeping completed, the hangup watcher recorded its outcome before the terminal reason could be committed.

## The cost of treating connection state as business state

A webhook fires when the peer disconnects, so it feels natural to wire that directly into your analytics as a user-disconnected outcome.

An AI agent has a lifecycle that exists independently of the socket carrying the audio. The agent can complete its goal, summarise the interaction, and decide to end the call. That decision is a rich, high-fidelity business event.

A SIP `BYE` is a low-fidelity network event. It tells you nothing about the user's intent. Overwriting a high-fidelity conversational outcome with a low-fidelity network outcome destroys data.

## Implementing the precedence rule in code

The implementation requires defining an explicit state machine for the call record.

You cannot rely on the framework to handle this, because the framework does not know what your integration considers a successful outcome. The framework only knows that the session ended.

I use a structured failure taxonomy for the termination reasons. Each reason carries a rank or a boolean flag indicating whether it is terminal. When an event attempts to update the call record, the update handler reads the current state.

If the current state is empty or in-progress, the update proceeds. If the current state is already a terminal reason, the handler compares the incoming reason against the existing one. If the incoming reason is a network disconnect and the existing reason is a conversational success, the update is silently dropped. This requires per-call event-sourced traceability.

## Common questions

**Why does my voice agent transcript show a natural end but the reason is hangup?**

The caller hung up immediately after the final prompt, triggering a network disconnect event. This late network event overwrites the earlier conversational success status in your database. You are seeing the result of a last-write-wins race condition.

**How do I prevent network teardowns from overwriting success states?**

Implement a state machine precedence rule for the call outcome field. Define success as a terminal state. Once the agent writes a terminal state, configure the database update logic to reject any subsequent network disconnect reasons.

**Should I delay the SIP BYE event so the success state lands first?**

No. Adding a fixed sleep to race conditions only hides the problem until the system is under load. You must fix the architecture by making the outcome field single-writer by precedence rather than by timing.

If the boundary this note describes is one you would rather own, [owning the TURN and SIP path directly](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/completed-calls-are-recorded-as-hangups-because-the-disconnect-wins-the-write-ra
