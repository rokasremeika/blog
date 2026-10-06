> Originally published at https://www.dreamchasers.lt/blog/voice-agent-end-call-tool-truncates-the-final-sentence-or-loses-the-callers-last
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 20 September 2026.

A function tool that tears the session down when called ends the call while the closing audio is still being generated or played out, so the caller hears half a sentence. Deferring teardown to the next caller turn boundary fixes truncation but creates a second failure. If a pacing watchdog force-arms the end while the caller is answering, the teardown trigger consumes the answer and it never reaches the transcript. The fix is an armed closure that carries a flag for whether it may fire on the next turn. Tool-initiated ends set it true, and force-armed ends set it false.

## Symptoms

- The agent signs off, but the connection drops before the audio finishes playing. The caller hears half a sentence or nothing at all.
- The caller answers the agent's final question, the call ends politely, but the caller's final answer is entirely missing from the transcript.
- The agent says its closing line, but the caller hears absolute silence on that turn.
- The conversation is supposed to be bounded, but the agent ignores time limits. Calls regularly overrun until a hard transport or provider limit drops them.

## What is actually happening

If a tool ends the session, the teardown begins while the text generation or audio synthesis for that same turn is still in flight. The connection closes while the closing audio is still being generated or played out, resulting in truncated audio.

Moving the teardown to fire on the next caller turn boundary fixes the truncation, allowing the agent's speech to play completely. However, this introduces a second failure. If something else arms the teardown, such as a pacing watchdog forcing a close because the time budget ran out, the very next caller turn is the caller's final answer. If the code fires the teardown right on that boundary, it consumes the answer as the teardown trigger. The answer never reaches the transcript entirely.

If the agent is going silent on the turn where it calls the tool, the issue lies at the generation level. On a speech-to-speech model, a turn produces one generation. A function call terminates that generation. If the model attempts to generate speech and call an end-tool in the exact same turn, the audio being synthesised is truncated or never emitted. No framework guards against this because nothing in the SDK knows this specific tool is fatal to concurrent speech.

The call overruns happen because nothing in a realtime session tells the model how much of its budget it has used. Elapsed time is not in the context window. Prompt-level pacing instructions compete with everything else in the prompt and decay as the conversation grows.

## How to confirm it

1. Log the turns that contain both generated audio and a tool call. Expect to find that every time the agent goes silent on its closing line, the model attempted to invoke the end tool concurrently with speech.
2. Record the specific path that triggered the teardown on every call missing a final transcript. Expect the transcript loss to correlate perfectly with force-armed closures happening on a caller turn boundary, not tool-armed closures.
3. Inspect the agent's own output transcription for the final turn. Expect the intended sign-off text to be present in the output transcription, confirming the model attempted to speak but the function call terminated the concurrent generation.

## How it works

End tools do not perform a teardown; they arm a pending closure. The armed closure carries a flag indicating whether it may fire on the next caller turn.

Tool-initiated ends set this flag to true. The closing line plays, the caller gets their beat to respond, and the following turn boundary provides a clean place to tear down the connection. Force-armed ends set this flag to false. They are driven by a different path, ensuring the caller's answer is completely recorded and transcribed first, with the teardown executing only after the answer completes.

I guarded the arm and fire state with a lock that was held briefly and had no asynchronous await inside it. Arming the closure races with the turn boundary that fires it. Putting an await inside that lock simply reintroduces the exact race condition you just closed.

The constraint that the closing line and the arming call must be separate turns is enforced in the tool description, not in code. The model decides which turn to call the tool in, so the only enforcement surface is the text the model reads.

## What I would check first

1. Check whether the missing transcript correlates with a watchdog timeout or an agent tool call. An agent-initiated end rules out the watchdog, meaning your turn boundary logic is consuming the answer. A watchdog timeout means you need a separate path for force-armed teardowns.
2. Check if the closing line exists in the model's output generation payload. If the text is there but the caller heard nothing, it rules out a logic failure and confirms the tool execution is destroying the concurrent audio generation.
3. Record whether each call closed by agent decision, by pacing force-arm, or by a hard cap to isolate the exact limit being hit.

## Pacing watchdogs and pure functions

To fix calls overrunning the budget, I run a pacing watchdog as a separate async task alongside the session. Each tick computes how much of the budget remains and decides whether the agent should begin closing. When it should, the watchdog injects a closing instruction at a turn boundary rather than mid-generation. If the agent fails to wrap up, the watchdog force-arms the teardown.

Two structural rules govern this approach. First, the decision logic is a pure function of elapsed time and session state. It takes timestamps and state, returning a decision. This means the boundaries can be unit tested exhaustively without a live model or a room. The async task is just a thin loop that gathers state, calls the function, and acts.

Second, the watchdog is default-on but gated by an environment kill switch. Watchdogs act on live calls unprompted, so a misfiring watchdog must be disabled in seconds, not in a full deploy. The operational question is not whether it works, but how fast you can stop it when it is wrong.

## Putting behavioural rules in tool descriptions

Naming the rule inside the tool description is a small structural choice that pays off. The model decides when to call a tool. Any turn-level contract, such as instructing the model that this tool must be called in a turn containing no speech, belongs in the description.

Do not put it in the system prompt where it competes with everything else. Give the rule a name so you can search transcripts and prompts for it later. The description is the most reliable per-tool instruction channel the model has, read right in the context where the decision occurs.

Verify it behaviourally on real runs. A description is a request, not a constraint. That is as far as an automated run can take you. Confirming the audio itself reached the caller requires a human on the line, since there is no recording and no second recognizer to check it for you. Both halves of this failure came from sitting on my own test calls and listening to them end badly. The truncated sign-off is obvious to anyone on the line and completely invisible in logs. The eaten answer is the opposite: it sounds perfect to a listener, but exists only as an absence in the transcript. You only notice it if something downstream depends on that last answer.

## Common questions

**Why does the agent audio cut off when it calls the end tool?**

On a speech-to-speech model, a function call terminates the generation for that turn. If the model attempts to generate speech and call a tool concurrently, the audio synthesis is truncated or destroyed entirely.

**Why is the caller's last answer missing from the transcript?**

Your teardown logic is firing exactly on the final caller turn boundary. By executing the close at that moment, the system consumes the answer as the teardown trigger before it reaches the transcript.

**How do I enforce a strict time limit on a voice agent?**

A realtime model has no clock, so prompt instructions decay over time. You must run a pacing watchdog as a separate asynchronous task that observes the clock, injects closing instructions, and force-arms a teardown if necessary.

**Where should I put instructions about how a tool is used?**

Put behavioral rules in the tool description as an explicit named rule, not in the system prompt. The description is the most reliable per-tool instruction channel, read exactly when the tool choice is made.

Canonical URL: https://www.dreamchasers.lt/blog/voice-agent-end-call-tool-truncates-the-final-sentence-or-loses-the-callers-last
