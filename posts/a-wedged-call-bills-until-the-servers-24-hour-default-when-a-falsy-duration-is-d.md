> Originally published at https://www.dreamchasers.lt/blog/a-wedged-call-bills-until-the-servers-24-hour-default-when-a-falsy-duration-is-d
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 5 November 2026.

A transport-level maximum call duration fails when the computed value evaluates to falsy. A bad estimate yielding zero, NaN, or undefined is silently omitted by the SDK from the placement request. Without that field, the SIP server substitutes its own generous default, typically 24 hours. Since your application worker is wedged, application-level timeouts fail to run. The fix is to strictly validate the duration, add a 90-second headroom margin, clamp the result to a hard 3600-second ceiling, and throw loudly if the value is non-positive.

## Symptoms

- A call that stopped making progress hours ago remains active on the carrier side and continues to accrue charges.
- You set a maximum call duration when requesting the outbound dial, but it appears to have had no effect at all.
- There is no error returned by the placement request, no early truncation, and no obvious warning in the application logs.
- The leg outlives the conversation by hours, and the only indication of a problem is a 24-hour billing line item on your SIP trunk invoice.

## What is actually happening

Application-level timeouts live entirely within your application process. When the process dies, the worker gets evicted by the orchestrator, or the agent loop wedges without exiting, none of those timeouts run. When the application goes quiet, the only thing that can still end the call is the transport-level duration cap sent with the initial placement request.

Two traps sit in that transport cap. First, if the computed duration value evaluates to falsy in your language of choice (zero, NaN, or undefined resulting from a bad computation), the SDK omits the field from the placement request entirely. It does not send a zero. It sends nothing. The SIP server receives a request with no time limit and falls back to its own configured default. That default is typically 24 hours, which is exactly the runaway billing scenario the duration field was supposed to prevent.

Second, if the cap is derived dynamically from an estimated call length with no fixed ceiling, a single bad estimate produces an absurd cap. A computation error that yields an absurdly large limit is syntactically valid and gets passed to the server, providing no actual cost protection. The combination of these two traps means a computed estimate is a dangerous thing to rely on for raw cost control.

## How to confirm it

1. Inspect the raw outbound placement request payload on a failing call using a network trace or the raw SDK debug logs. Expect to see the maximum duration field entirely missing, confirming the SDK dropped a falsy value before serialization.
2. Log the raw computed duration variable just before it enters the placement call. Expect to see `NaN`, `undefined`, or `0` on the specific runs that resulted in a runaway call.
3. Check the carrier billing logs for the affected leg. Expect a duration of exactly 24 hours, or whatever the maximum allowed server-side session limit is for your specific SIP trunk.

## How it works

The architecture of an outbound AI call involves three independent lifecycles: the application worker, the WebRTC room, and the SIP leg. Your application code is responsible for managing the call under normal conditions. It decides when the conversation is over, tears down the agent, and issues the hangup command.

When the application fails to do this, the SIP service acts as the backstop. Your code passes a maximum duration on placement, and the SIP process is the component that actually tears the leg down when that timer expires. The duration cap is enforced by the SIP infrastructure rather than scheduled inside your application code. The duration cap is a transport-level constraint enforced by the infrastructure, not a scheduled task running in your application.

To guarantee this backstop works, the duration value must be strictly validated before it leaves your process. I compute the cap from the expected conversational length plus a generous headroom. Specifically, I take the estimated seconds and add 90 s of headroom. That 90 s buffer accounts for ringing and a slow goodbye sequence from the user.

Once I have that computed value, I clamp it to a hard ceiling of 3600 s, regardless of what the estimate says. A one-hour call is the maximum acceptable risk for this specific deployment. Finally, I validate the integer. The code throws loudly on non-finite or non-positive values rather than letting them pass into the SDK. I treat this as a cost control that is independent of every application-level timeout, because its whole purpose is to work when your code is dead.

A placement request that omits the duration field relies entirely on the server-side default rather than an explicit transport contract. The SIP server substitutes a 24-hour default when the duration parameter is omitted. Since I cannot control the upstream default across every provider, I assume an omitted field results in a 24-hour runaway, and I enforce a hard, validated integer locally to prevent the omission entirely.

## What I would check first

1. Check whether the SDK omitted the duration field entirely or passed an unchecked, oversized estimate. Inspect the raw placement request payload sent by the SDK. A missing field points to a falsy value in your local code. A present field on a runaway call points to an unchecked computation that produced an absurdly large cap.
2. Check if the runaway call correlates with a specific conversational branch or logic path. Look at calls with zero or unusually large estimated lengths in your logs. Expect to find a division by zero or a missing configuration key that results in `NaN`, ruling out a completely random SDK bug.
3. Check whether the application process actually crashed or if the agent loop is still running but blocked. A crash rules out application-level timeouts immediately. A blocked loop means you have a deadlock that the transport cap must catch, which is exactly why the cap must not depend on the worker staying alive.

## Validating the ceiling locally

Any per-call cost control passed as an optional field has this exact failure mode. The SDK drops falsy values, the server substitutes a generous default, and you find out on the invoice. The rule is to validate the number before it leaves your process.

Do not let a computed estimate be the only thing standing between you and a runaway leg. The calculation is derived from an estimated conversational duration computed by your application. If a computation encounters missing inputs or an arithmetic error, the resulting duration evaluates to zero or NaN. A simple `Math.max(calculated, minimum_safe_duration)` followed by `Math.min(result, hard_ceiling)` prevents the pipeline from outputting a value that the SDK will discard.

If you are struggling with calls that hang open for entirely different reasons, such as the agent worker being dispatched after the SIP participant connects, you might be facing an ordering problem. Creating the room and dispatching the agent before you dial is necessary to prevent dead air. I cover that failure mode in [/blog/outbound-call-answers-to-dead-air-before-the-agent-joins/](/blog/outbound-call-answers-to-dead-air-before-the-agent-joins/).

## Instrumenting the failure mode

A transport cap acts as an infrastructure safety net rather than a standard lifecycle completion. When a call hits the maximum duration limit, the application has failed to tear it down gracefully.

I count calls terminated by the transport cap as a distinct defect class rather than as a standard successful outcome. If a call hits the 3600 s ceiling, record the termination under the wedged-agent defect class in the per-call event log. This distinction is important because the SIP server does not send a special application error when it enforces the cap. It simply tears down the leg. If you do not explicitly track the duration against the cap, the teardown looks like a normal network disconnect.

I also record the computed cap alongside the actual duration on every single call. This allows the 90 s headroom to be checked against reality. If a successful call consistently ends 85 s into the headroom, the estimate is too tight and risks truncating a legitimate conversation. If the call ends 10 s into the headroom, the buffer is overly generous.

Finally, I count placement attempts rejected by the local validation guard. A rejected placement attempt is a saved runaway call. By tracking these rejections, you can trace the bad estimates back to their source in the duration computation without paying the carrier for a 24-hour debug session.

## The interaction with application teardown

The transport ceiling operates independently from the application teardown sequence. A healthy call should never hit this ceiling.

When the application decides the call is over, it executes a graceful hangup. This involves finishing the final spoken sentence, closing the WebRTC room, and signaling the SIP participant to disconnect. Mistakes in this teardown sequence can also cause the room to outlive the conversation. Replacing a fixed sleep with an await on the current speech playout is the correct fix, which I detail in [/blog/agent-last-sentence-is-cut-off-and-the-room-outlives-the-call/](/blog/agent-last-sentence-is-cut-off-and-the-room-outlives-the-call/).

The transport ceiling does not replace the need for a correct application teardown. It exists solely to bound the financial damage when the application teardown logic fails entirely.

## Common questions

**Why does my SIP call continue billing after the agent crashes?**

The application worker orchestrates the call, and all application timeouts die with it. Without the worker, only the SIP server can end the call. If the placement request lacked a hard duration ceiling, the server falls back to its own limit, which is typically 24 hours.

**Why is the maximum duration field missing from the placement request?**

The SDK silently drops falsy values. If your computed duration evaluates to zero, NaN, or undefined, the SDK omits the field entirely before serialization. You must validate the integer and throw locally on non-positive values.

**How much headroom should I add to an estimated call duration?**

I use 90 seconds of headroom on top of the estimated conversational length. This accounts for ringing and a slow goodbye. If the call takes longer to connect, the headroom prevents early truncation.

**How do I prevent a bad estimate from creating an absurd call limit?**

Clamp the calculated duration to a hard ceiling regardless of the estimate. I use a strict 3600-second maximum for outbound calls. This guarantees that even a logic error in the estimation pipeline cannot result in a runaway leg.

If the boundary this note describes is one you would rather own, [moving this layer off LiveKit Cloud](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/a-wedged-call-bills-until-the-servers-24-hour-default-when-a-falsy-duration-is-d
