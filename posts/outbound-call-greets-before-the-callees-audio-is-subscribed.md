> Originally published at https://www.dreamchasers.lt/blog/outbound-call-greets-before-the-callees-audio-is-subscribed
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 14 September 2026.

The SIP leg reporting active is only a hint that the call is up, not a guarantee that audio is flowing. If your agent greets on the active signal, it speaks into a cold pipe before the callee's audio track is actually subscribed. The first thing the user says arrives before the agent worker is listening, so it never reaches transcription. Fix this by explicitly gating the greeting on the inbound audio publication being subscribed, using the active signal only to arm a bounded 2.5 s fallback grace window.

## Symptoms

On outbound PSTN calls, the first thing the person says is missing from the transcript. The agent talks over them, asks something they already answered, or the whole conversation runs one turn out of step. Browser calls into the same agent are fine, which sends everyone looking at the model instead of the call setup. The symptom is intermittent because nothing guarantees the carrier timing is the same on the next call or the next route, which makes it read as a probabilistic model failure rather than a deterministic timing race.

## What is actually happening

Any outbound voice agent on any carrier has two distinct events at the start of a session. The first event is the call coming up, where the SIP participant attribute reports the leg is active. The second event is audio actually flowing into your process, where the callee's inbound audio publication is subscribed by the agent worker.

The active signal flips a beat before the audio publication is actually subscribed. If your system triggers the agent greeting based on the leg reporting active, it plays the opening line into an input pipe that nothing is reading yet. The person on the other end hears the greeting and answers, but their first answer arrives before the agent worker is fully listening. The utterance never reaches the transcriber or the model.

The gap between these two events is short.

## How to confirm it

1. Record the milliseconds between the leg reporting active and the first audio track subscription, per call. The gap between the two signals is the single number that tells you whether this is your bug, and it costs one timestamp subtraction per call to collect.
2. Count calls where the first user turn is empty, grouped by SIP versus browser transports.
3. Record which of the two paths released the greeting: track subscription or grace expiry.

## How it works

The correct approach is to gate the greeting on an audio publication actually being subscribed, not on the leg reporting active. The active signal is only a hint.

The obvious fix of adding a fixed sleep after the active signal is wrong. A fixed sleep is a guess at a distribution you have not measured. If you guess too short, the symptom stays. If you guess too long, you add dead air to every call and make the agent feel slow.

Instead, the active signal should only arm a bounded grace window of 2.5 seconds. The moment the actual track subscription fires, it preempts that window and releases the greeting immediately. If the subscription never arrives, the window expires and releases the greeting anyway, so the call fails open rather than hanging forever in silence. I settled on 2.5 seconds as a bounded fallback chosen to sit comfortably above the gap a normal SIP trunk shows. You should instrument the per-call gap first, so the number can be tuned from your own real data.

Clamping this grace window matters more than it looks. The call needs an overall answer budget, which I set to 45 seconds for ringing and answering combined. If a leg reports active at 44 seconds into that 45 second budget, a 2.5 second grace window pushes the greeting past the deadline. A call that was about to succeed is recorded as a timeout. To prevent this, clamp the grace window to whatever remains of the answer budget. The grace can only ever consume time that already belongs to the call.

Finally, remove both event handlers in a finally block. A retry cannot be allowed to stack listeners, and if the handler registrations themselves fail, the code must drop through to the greeting rather than leaving the worker permanently waiting.

## What I would check first

When an outbound agent misses the first thing the caller says, the subscription gate is not the only candidate. Here is the order I check things to isolate the exact cause.

1. Check if the agent worker is actually present in the room before the phone rings. If your code dispatches the agent after creating the SIP participant, expect the worker to still be starting when the callee picks up. This rules out the audio gate, because there is no agent in the room to gate yet.
2. Check the echo-cancellation warm-up duration on SIP legs. If the greeting waits for the audio track perfectly but a short "hello" still vanishes, expect the stock 3.0 second WebRTC warm-up to be substituting silence for the caller's first word.
3. Compare the time from dispatch created to worker joined against the time from placement to answer. If the worker join time is later than the answer time, the worker was still starting when the callee picked up.

## Creating the room and dispatching before dialing

Agent presence before answer is a precondition for the subscription gate to work, not an alternative to it. Outbound placement consists of three operations whose order is load-bearing.

If you dispatch the agent after the SIP participant is created, the worker can still be starting when the callee picks up. The greeting gate cannot help you, because there is nobody to greet yet. If you create the room implicitly without metadata, the platform's native webhooks carry only platform identifiers. They know the room identifier, but they have no access to your database to resolve the tenant or number.

The correct order fixes both the timing and the ownership. First, create the room explicitly, carrying the routing metadata the webhooks will need. Second, create an explicit agent dispatch so a worker is sitting in the room before the phone ever rings. Third, create the SIP participant last, with wait-until-answered, dial tone playback, a 45 s ringing timeout and a deterministic participant identity derived from the session id so every later event can be correlated without a lookup.

From the moment you create that room, you own it. Any failure after that point (a placement error, a classification failure, or an unexpected throw) leaks a room that a worker will occupy until the empty timeout expires. Wrap everything after room creation in a block that guarantees every failure branch deletes the room before returning the error.

## Echo-cancellation warm-up on telephony legs

Even with the subscription gate perfectly implemented, the echo-cancellation warm-up default can still erase a short first answer on telephony legs. Fixing one without the other leaves the symptom in place, which is the usual reason a team concludes the subscription gate did not work.

The warm-up substitutes silence for the far-end audio during the agent's first turn so the agent does not hear itself arriving back through the echo path. The stock window is 3.0 seconds, which is sized for browser audio. On a phone leg, the person answers inside that window, so their utterance is replaced by silence before anything downstream can process it.

Setting the warm-up to zero breaks the agent in the opposite direction. Late-subscribing carrier echo reaches the input, the agent treats its own greeting as user barge-in, and the turn wedges. I cover this exact failure in my post on why [echo-cancellation warm-up eats a short first answer on phone calls](/blog/echo-cancellation-warm-up-eats-a-short-first-answer-on-phone-calls/).

The fix is to make the warm-up transport-dependent. A value of 2.0 seconds on SIP legs is long enough to cover the carrier echo path and short enough that a normal phone answer lands outside it. Keep the stock 3.0 seconds on WebRTC, where the browser's own echo cancellation changes the problem entirely. Implement it as a single override point that returns no override at all for non-SIP sessions. Because the override returns nothing for browser calls, the WebRTC path runs exactly the stock code it ran before, making any regression unambiguously attributable to the SIP branch.

## Common questions

**Why is the caller's first word missing on outbound SIP calls?**

The agent is greeting based on the SIP leg reporting active, before the audio track is subscribed. The caller replies into a cold pipe, so the transcription never captures their first word.

**Should I add a sleep timer before the agent greeting?**

No. A fixed sleep is a guess at a variable network delay. Too short and you still miss words; too long and you add dead air to every call. Gate the greeting on actual track subscription instead.

**Why does my agent interrupt its own greeting on phone calls?**

If you disable the echo-cancellation warm-up completely, carrier echo feeds the agent's voice back into its own input. The agent treats its own echo as user barge-in and stops talking.

**In what order should I place outbound AI calls?**

Create the room with routing metadata first, then dispatch the agent explicitly. Create the SIP participant last so the worker is waiting in the room before the phone rings.

If the boundary this note describes is one you would rather own, [running the SFU yourself](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/outbound-call-greets-before-the-callees-audio-is-subscribed
