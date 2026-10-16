> Originally published at https://www.dreamchasers.lt/blog/agent-audio-cuts-off-when-calling-a-function-tool-in-the-same-turn
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 30 September 2026.

On a speech-to-speech model, a single turn produces one atomic generation. Calling a function tool terminates that generation early, which truncates the concurrent audio or stops it from being emitted at all. No framework guards against this because the SDK does not know your specific tool's semantics. The fix is to add an explicit named rule to the tool's description instructing the model to call the tool in a turn containing no speech. This places the constraint exactly where the model makes the decision, avoiding the context decay of the system prompt.

## Symptoms

The agent said a line and the caller heard nothing. Sometimes, the caller heard just the first syllable of a word, followed instantly by silence. The issue appeared intermittent and unrelated to the tool, because the rest of the conversation turns functioned normally.

However, the failure isolated to very specific interactions. It happened only on the turns where the model also called a function tool.

The most misleading observation was the text transcript. When I pulled the session logs, the output text transcript was complete. It showed that the agent generated the correct phrase alongside the tool call. The logs indicated the turn completed and the tool fired, masking the dropped audio from automated log filters. The model had output the text transcription internally, but no corresponding audio reached the caller.

## What is actually happening

On a speech-to-speech model, reasoning, text generation, and audio synthesis operate together behind a single provider connection. A turn produces exactly one generation.

When the model decides to invoke a tool, it emits a function call token. That function call terminates the generation phase for that turn. If the model was synthesizing audio at the same time, the audio generation is abruptly cut off. The audio currently in flight is either truncated mid-word or flushed completely before it ever reaches your infrastructure.

In a speech-to-speech architecture, audio generation cannot be decoupled from the model's turn. When the model invokes a function tool, the single generation for that turn terminates immediately. Audio synthesis and model reasoning share a single execution lifecycle on the provider.

In a speech-to-speech model, there is no text-to-speech plugin running on your server. Hearing, reasoning, and speaking sit behind one model and one websocket. The provider handles the audio synthesis internally. When the model makes a choice to emit a tool call, that action signals the end of its response phase. The provider stops synthesizing audio immediately. The audio track is cut off at the source. If the model generated full text internally but only synthesized one syllable of audio before calling the tool, that single syllable is all that arrives over the network.

No framework can guard against this for you. The LiveKit SDK receives the media stream exactly as the provider sends it. If the provider terminates the generation, the SDK stops receiving audio frames. The SDK does not raise an error because it has no way of knowing this was unintended. Whether this truncation matters depends entirely on your tool's semantics. Nothing in the SDK knows that this particular tool is fatal to concurrent speech.

## How to confirm it

1. Log every turn that contains both a tool call and generated audio. Expect to see this exact combination exclusively on the turns where the audio dropped. That combination is the defect.
2. Score whether the intended speech for a tool-calling turn appears in the agent's own output transcription for that turn. Expect the text transcript to contain the intended phrase, confirming the model attempted to speak before the generation terminated.
3. Put a human on the line to trigger the specific tool, and listen to the output. Expect the human to hear silence or a clipped syllable. Confirming the audio itself reached the caller needs a human on the line, since there is no recording in the realtime media path and no second recognizer to check the final audio output for you automatically.

## The fix

To solve this, I changed how I prompted the tools. The model has complete control over when a tool is invoked. Your code cannot force the model to speak first and call the tool later. The only enforcement surface you have is the text the model reads during its generation phase.

I put the turn-level contract directly into the tool description. I added an explicit, named rule stating that the tool must be called in a turn containing no speech.

I did not put this rule in the system prompt. In a system prompt, behavioral constraints compete with persona instructions, safety guardrails, and the growing conversation history. As the context window fills, a system prompt rule decays and the model forgets it. Function-tool descriptions avoid this because they are the most reliable per-tool instruction channel a model has. They are read in the exact context where the model evaluates its available tools and makes the decision. Anything the model must do around calling a tool belongs in the description.

I gave the rule a specific name, like `RULE_SILENT_TURN`, so I could search transcripts and prompts for it later. Without a name, you cannot easily query your logs to see if the model even acknowledged the constraint.

Once the description was updated, I verified it behaviorally on real runs. A tool description is a request, not a guaranteed constraint. The model can still fail to isolate the turn. I set up my evaluation runs to score whether the intended line appeared in the agent's own output transcription for that turn, rather than just checking whether the tool fired. Scoring the tool execution tells you nothing about the audio failure. You must score the turn isolation.

That is as far as an automated run can take you. Automated scoring confirms that the model emitted the transcript text, but cannot detect whether synthesised audio was dropped before playout. Live listening on test calls remains the only verification that the provider emitted the audio track to the caller.

## What I would check first

1. Check the event sequence in your provider logs for the failing turn. Expect the function call event to arrive in the exact same generation block as the terminated speech. This confirms the generation was killed by the tool call, ruling out a network disconnection.
2. Check if the tool in question tears down the session. If the tool is designed to hang up, you are dealing with a lifecycle race rather than just a generation limit. A function tool that tears the session down ends the call while the closing audio is still being generated or played out. If this is the case, defer the teardown to the next caller turn boundary. I have documented this specific failure in /blog/voice-agent-end-call-tool-truncates-the-final-sentence-or-loses-the-callers-last/. This observation discriminates between a standard silent turn and a teardown race.
3. Check whether the silent turn coincided with a websocket reconnect edge rather than a function tool execution. Expect a reconnect edge to leave both sides waiting silently because in-flight generation died with the dropped connection, which points to a missing reconnect redrive watchdog rather than a tool-turn defect. If the failure occurs on clean transport without reconnect events, you have ruled out transport drops and isolated the failure to concurrent tool calling.

## Common questions

**Why does my voice agent transcript show text but play no audio?**

The model generated text but terminated the turn before synthesizing the audio. This happens on speech-to-speech models when a tool call interrupts the generation phase. The audio being synthesised is truncated or never emitted at all, leaving the text intact while the caller hears silence.

**Can I fix silent tool turns in the system prompt?**

No, because system prompt instructions decay as the conversation history grows. The rule competes with persona constraints and loses attention weight. You must place the constraint directly in the tool description, which is evaluated at the moment the model chooses to call the tool.

**How do I know if the model ignored my tool calling rule?**

You must verify the turn behaviorally on real runs by scoring the text transcripts. Score whether the intended line appears in the output transcript, and verify the tool-calling turn contains zero speech tokens. Scoring whether the tool fired is not enough.

**Does the LiveKit SDK catch aborted audio generations?**

No, because the SDK has no way to know if concurrent speech with a specific tool is an error. The SDK simply stops receiving audio frames when the provider terminates the stream. You must handle this constraint at the model prompting layer.

Canonical URL: https://www.dreamchasers.lt/blog/agent-audio-cuts-off-when-calling-a-function-tool-in-the-same-turn
