> Originally published at https://www.dreamchasers.lt/blog/echo-cancellation-warm-up-eats-a-short-first-answer-on-phone-calls
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 10 September 2026.

The echo-cancellation warm-up substitutes silence for the far-end audio during the agent's first turn to prevent it from hearing its own greeting. The stock 3.0 s window is sized for browsers. On a phone leg, a person answers inside that window, so their first utterance is replaced by silence. Setting this to zero stops the erasure but lets late-subscribing carrier echo trigger a false interruption. The fix is a transport-specific 2.0 s warm-up for SIP calls.

## Symptoms

A one-word first answer like "yes", "speaking", or "hello" never reaches the agent on telephony legs. Longer first answers survive, but their opening word is clipped from the transcript.

Browser calls into the exact same agent work perfectly, which typically sends teams looking at the model instead of the call setup.

If you set the echo-cancellation warm-up to zero to fix the missing word, the agent immediately starts interrupting its own greeting. The call begins with the person asking whether anyone is there, because the agent stopped talking mid-sentence.

On outbound PSTN calls, the first thing the person says might be missing entirely. The agent talks over them, asks something they already answered, or the whole conversation runs one turn out of step.

## What is actually happening

The echo-cancellation warm-up substitutes silence for the far-end audio during the agent's first turn. This erasure prevents the agent from hearing its own voice arriving back through the echo path.

The stock window is 3.0 s, which is sized for browser audio. On a phone leg, the person answers inside that 3.0 s window. Their first utterance is replaced by silence before anything downstream can see it.

Zero is the tempting alternative value. It fails in production rather than in testing because carrier echo depends entirely on the path and only shows up on some routes. Setting the window to zero removes the erasure entirely. Late-subscribing carrier echo reaches the input, the agent treats its own greeting as a barge-in, and the turn wedges.

There is a second race condition hiding behind the warm-up. The SIP participant attribute that reports the leg as active flips a beat before the callee's audio publication is actually subscribed by the agent worker. Greet on that active signal, and the opener plays into an input pipe nothing is reading yet. The first answer arrives before anything is listening.

## How to confirm it

1. Log every first user turn that arrives empty, tagged with the transport. Expect SIP and browser to show entirely different failure rates; a SIP-exclusive failure points straight to the transport boundary.
2. Measure the time from the end of the greeting to the first inbound audio frame that survives the warm-up. Expect this gap to match your current warm-up window if the cancellation logic is eating the audio.
3. Count barge-in events attributed to the agent's own greeting. Expect zero on a healthy configuration; a high count means the warm-up is either missing or too short to cover the carrier echo path.
4. Record the milliseconds between the leg reporting active and the first audio track subscription, per call. Expect a measurable gap. If your greeting fires before this gap closes, the agent is speaking before it can hear.

## How it works

The fix is making the warm-up transport-dependent instead of global. I settled on a 2.0 s echo-cancellation warm-up on SIP legs. This is long enough to cover the carrier echo path and short enough that a normal phone answer lands outside it.

I kept the stock 3.0 s warm-up on WebRTC, where the browser's own echo cancellation changed the problem. I implemented this as a single override point that returned no override at all for non-SIP sessions. Because the override returned nothing, the browser path stayed byte-equivalent to stock. Any regression is unambiguously attributable to the SIP branch.

For the audio subscription race, the fix is gating the greeting on an audio publication actually being subscribed, not on the leg reporting active. The active signal only arms a bounded grace window of 2.5 s. Track subscription preempts that window the moment it fires.

The 2.5 s grace window is a fallback, not the primary mechanism. Its only job is to stop a call hanging forever when the subscription event never arrives at all. I clamped this grace to whatever remained of the overall 45 s ringing and answer budget, so a late active signal could never turn a would-be success into a timeout. I removed both event handlers in a finally block so a retry could not stack listeners. If the handler registrations themselves failed, I failed open and greeted rather than hanging the call forever.

## What I would check first

1. Verify whether the missing first turn happens on browser calls or only on SIP. A failure restricted to SIP rules out the model, isolating the issue to transport-specific echo or timing.
2. Check what event triggers your agent's greeting. Finding a greeting triggered by the call reporting active, rather than the track being subscribed, guarantees a missing first answer regardless of echo cancellation.
3. Look at your agent's interruption metrics. Finding an agent that interrupts itself during the greeting indicates either a warm-up value of zero or an unhandled carrier echo path.

## Why the subscription gate is not enough

This is the second half of the outbound first-answer bug. The subscription gate decides when the greeting starts; the warm-up decides whether the reply to it is heard at all.

Teams that fix only the subscription gate observe that the caller's first word is still missing. They invariably conclude the gate did not work. Fixing one without the other leaves the symptom entirely in place.

The two obvious alternatives for the subscription gate are both wrong. A fixed sleep after the active signal is a guess at a distribution you have not measured. If the sleep is too short, the symptom stays. If it is too long, you have added dead air to every call and made the agent feel slow. Greeting on participant-connected is worse, because that fires earlier still.

Instrument the per-call gap before changing anything. The time between the two signals is the single number that tells you whether this is your bug, and it costs one timestamp subtraction per call to collect.

## Shutting down cleanly

Ending a session exposes another set of timing failures that present as truncated audio or resource leaks. When a call ends, the closing line is often truncated mid-word. Separately, you might find idle workers attached to empty rooms long after the call finishes. This presents as idle workers and reads as a load problem rather than a leak.

A fixed sleep before hangup is a guess at the length of a variable utterance. Too short and it truncates the goodbye; too long and every call pays for dead air. The standard session close call does not drain by default. Calling it while speech is playing interrupts that speech by design.

I replace the sleep with an await on the current speech's playout, capped at 6 s. The cap ensures a wedged synthesiser cannot hold the call open indefinitely. End the turn by letting playout finish naturally.

Waiting for playout lengthens the window in which the far end can hang up. This makes a precedence rule on the call's end-reason necessary rather than optional.

For the leaked rooms, the mechanism is a cancelled teardown. A room delete issued inside a teardown that the outer timeout is about to cancel is cancelled along with it. The timeout fires, the coroutine is cancelled mid-await, and the room is left behind.

I shield the room delete from cancellation and give it its own 5 s timeout so it survives the enclosing teardown being cancelled. Finally, I treat a not-found response on delete as a success. Someone else having already deleted the room is the exact outcome you wanted.

## Common questions

**Why does the voice agent not hear the caller say yes?**

The echo-cancellation warm-up window deletes the first few seconds of inbound audio. A short first answer falls entirely inside this window and never reaches transcription.

**Why does the agent interrupt its own greeting on a phone call?**

Carrier echo returns the agent's voice back to its own input. Without an adequate echo-cancellation warm-up, the agent hears itself and treats its own greeting as a user barge-in.

**How do I stop the agent greeting before the caller is ready?**

Gate the greeting on the audio track being actually subscribed, not just the call reporting active. The active signal flips a beat before the audio pipe is ready to receive input.

**Why is the agent's last sentence cut off when the call ends?**

Closing the session without awaiting speech playout interrupts the synthesiser. You must await the playout with a hard timeout cap before issuing the hangup command.

If the boundary this note describes is one you would rather own, [a self-hosted deployment you control](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/echo-cancellation-warm-up-eats-a-short-first-answer-on-phone-calls
