> Originally published at https://www.dreamchasers.lt/blog/agent-last-sentence-is-cut-off-and-the-room-outlives-the-call
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 10 October 2026.

A voice agent cutting off its last sentence happens because a fixed sleep cannot predict utterance length, or because the session close call interrupts speech by default. Separately, leaking idle workers occurs when a room delete is cancelled by an outer teardown timeout. Fix both by awaiting speech playout with a six-second cap instead of sleeping. Shield your room deletion from the outer cancellation context and give it its own five-second timeout, treating a not-found response as a success.

## Symptoms

- The closing line is truncated mid-word when the call ends.
- The longer the goodbye, the more of the audio is lost.
- A cancelled room delete leaves the room active after the transport call finishes.
- A worker remains attached to the orphaned room until the empty room timeout expires.
- This presents as idle workers piling up, which looks like a load problem rather than a resource leak.

## What is actually happening

This failure stems from three independent mistakes grouped tightly in the same few lines of teardown code. The transport layer needs to disconnect, the agent needs to finish speaking, and the realtime platform needs to destroy the room. Failing to coordinate them leaves speech truncated or resources orphaned.

First, using a fixed sleep before hanging up is a crude guess at the length of a variable utterance. If the sleep is too short, it truncates the speech. If it is too long, every call pays for dead air, which bills unnecessarily. The audio pipeline pushes frames into a buffer continuously. A fixed sleep command completely ignores this buffer state.

Second, the session close call does not drain by default. Calling it while speech is playing interrupts that speech by design. The audio track closes immediately, cutting the stream off regardless of what is still buffered. If you rely on the close call to cleanly wrap up the turn, you lose the end of the sentence every time. The function is designed for a fast exit, not a graceful drain.

Third, a room delete issued inside a teardown block that the outer timeout is about to cancel is cancelled along with it. Teardown routines usually run inside an enclosing timeout wrapper. If the sequence hits that timeout, the coroutine is cancelled mid-await. The delete operation is cancelled before it can complete. The room is left behind, and the worker sits in it until the platform empty timeout finally sweeps it.

## How to confirm it

1. Record the playout wait duration per call, and how often it hits the cap. Expect calls to truncate whenever speech duration exceeds the fixed sleep duration.
2. Review the source code for your session close method invocation. Expect the default configuration to interrupt speech rather than waiting for the buffer to drain.
3. Pair your room creation and room deletion logs by their room identifier. Expect to find unmatched creations that correlate directly with the idle worker count.
4. Count the workers idling in empty rooms. Expect this number to climb exactly when teardown operations time out and cancel the enclosing coroutine.

## How it works

I replaced the fixed sleep with an explicit await on the current speech playout. The system knows when playback finishes and you do not, so you should never sleep for a duration you can await. I capped this await at 6 s. A capped wait ensures that a wedged synthesizer or an infinitely looping playback routine cannot hold the call open indefinitely.

To handle the session close, I stopped using the default close method that interrupts speech. Instead, I end the turn by letting the playout finish naturally before tearing down the transport. This allows the caller to hear the entire goodbye before transport teardown.

For the resource leak, I shielded the room delete from cancellation. Any cleanup that must happen even when the enclosing operation is abandoned has to be shielded from the outer cancellation context. I gave the room delete its own 5 s timeout so it survives the enclosing teardown being cancelled. Without this shield, your cleanup is the first casualty of the timeout it exists to survive.

Finally, I treat a not-found response on the delete operation as a success. If the room does not exist, someone else having already deleted it is the exact outcome you wanted.

## What I would check first

1. Check if the closing line is statically generated or dynamic. Expect dynamic lines to highlight the truncation more clearly, ruling out a static audio file being genuinely shorter than the fixed sleep.
2. Check whether idle workers correlate with teardown cancellations rather than incoming call volume. Expect unmatched room creations and idle workers to spike during outer teardown timeouts, ruling out real traffic load.
3. Check the count of workers idling in empty rooms. Expect workers to remain attached to empty rooms until the platform empty timeout expires, confirming a room delete leak rather than process starvation.
4. Check whether speech truncation happens on every call or only on utterances longer than the sleep window. Expect immediate truncation whenever session close interrupts active speech, distinguishing default close behavior from an undersized sleep timer.

## Why guessing utterance length fails

Synthesized speech length is completely unpredictable. Utterance length varies significantly across phrasing and language. Even within the same language, prosody changes and model variance mean the exact same sentence will yield different audio lengths on consecutive runs. Relying on a fixed timer fails in both directions.

If you set the timer conservatively to avoid truncation, you create a new problem. A fixed sleep that overestimates utterance length forces every call to pay for unnecessary dead air. The caller stays on the line waiting for something else to happen, and you pay your carrier for every second of that silence. If the caller speaks during this dead air, the agent is already tearing down and cannot respond. This leaves the caller talking to a dead line before the hangup eventually fires.

Awaiting the actual playout event from the audio pipeline is the only correct way to handle this. The underlying media server knows exactly how many frames it has pushed to the transport. By hooking into that completion signal, the application logic stays perfectly synced with the actual audio reality. The 6 s cap acts purely as a safety valve. I record the playout wait duration per call, and how often it hits the cap, to ensure the threshold is high enough to let normal speech finish but low enough to aggressively kill broken sessions.

## Managing asynchronous cleanup

Teardown code is inherently risky because it executes when the system is already trying to abandon an operation. If the caller hangs up, the transport disconnects, and the system wants to reclaim the worker immediately. However, the room delete still requires an API call to the realtime platform.

If that API call sits inside the main teardown block, it shares the same timeout. This is exactly where the leak happens. When the timeout fires, Python raises a cancellation error inside the awaitable. The cancellation interrupts the coroutine mid-await, leaving the room active on the platform. The realtime platform never receives the delete instruction, so the room stays open.

Shielding the cleanup operation decouples it from the parent timeout. By wrapping the room delete in an isolation layer and giving it a dedicated 5 s timeout, you ensure the request completes even if the main teardown fails. It is the same principle of ownership that forces you to place the outbound leg carefully to prevent [dead air before the agent joins](/blog/outbound-call-answers-to-dead-air-before-the-agent-joins/).

## The false load problem

The most confusing part of this failure mode is how it presents on monitoring tools. When a room outlives the call, the worker attached to it has nothing to do. The worker process is still alive and bound to that session.

Your capacity metrics will show high worker utilization. This looks exactly like a sudden spike in traffic or a legitimate load problem.

In reality, it is just a leak. The workers are trapped in empty rooms waiting for a server-side empty timeout to finally kick them out. Once you log room creation and room deletion as paired events, the unmatched creations expose the truth. The leak causes a slow buildup of useless capacity that drains your infrastructure budget until the fixed empty timeout provides delayed garbage collection. Counting workers idle in empty rooms gives you the downstream symptom of a failed delete, confirming the leak is active.

## Handling the not-found success state

Treating a not-found response on delete as a success is a critical detail for stable teardown. In a distributed telephony system, multiple actors often try to end the call simultaneously. The caller hangs up their physical phone, sending a BYE over the SIP trunk. The voice agent decides the conversation is over and triggers its own exit. A webhook from the realtime provider fires to indicate the participant left.

If all three events trigger a room delete, the first one succeeds. The next two receive a not-found error because the room is already gone. If your application code treats this as a failure, it will throw an exception, potentially skipping other crucial local cleanup tasks like releasing a database lock or closing a network socket.

By swallowing the not-found error and treating it as a successful deletion, you simplify the state machine. Someone else having already deleted the room is exactly the outcome you wanted. The room is gone, you are no longer paying for it, and the worker can terminate cleanly. This allows you to build a system that can absorb duplicate teardown triggers without throwing spurious errors or breaking the local cleanup path. A similar logic applies when an [end-call tool truncates the final sentence](/blog/voice-agent-end-call-tool-truncates-the-final-sentence-or-loses-the-callers-last/), where arming a pending closure rather than performing one immediately prevents conflicting operations from breaking the lifecycle.

## Common questions

**Why does the voice agent cut off the last word of the goodbye?**

The teardown logic uses a fixed sleep duration that is shorter than the synthesized audio. When the sleep finishes, the transport disconnects and truncates the audio before the phrase completes.

**Why are there idle workers sitting in empty LiveKit rooms?**

A room delete operation was cancelled mid-flight by an enclosing timeout during call teardown. The room stays open until a server-side empty timeout finally sweeps it, keeping the worker attached and idle.

**How can I safely wait for speech to finish playing?**

Await the current speech playout event directly from the media buffer instead of using a timer. Cap this await at six seconds so a frozen synthesizer cannot keep the session open forever.

**Why does the session close function interrupt my agent speech?**

The default behavior of session close methods is often to drop the active buffer immediately for a fast exit. You must let the playout finish completely before invoking the closure.

If the boundary this note describes is one you would rather own, [the migration scope this sits inside](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/agent-last-sentence-is-cut-off-and-the-room-outlives-the-call
