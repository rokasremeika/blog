> Originally published at https://www.dreamchasers.lt/blog/livekit-agent-opener-answers-a-phantom-turn-and-reply-requests-time-out
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 12 October 2026.

The framework forces the model to speak first by pushing a one-character text placeholder into the session, then waits for the provider to acknowledge it. A speech-to-speech model reads this punctuation as a real user turn and shapes its greeting as a reply to it. Because it is a client-initiated kickoff, the provider does not reliably emit the expected generation-created event, causing explicit reply requests to time out while the audio is already playing. The fix is patching the plugin to use an XML-shaped internal control marker on a private message channel, and resolving the reply future immediately upon enqueueing.

## Symptoms

- The agent's first spoken line sounds like a disjointed reply to something the caller said, even though the caller has not spoken yet.
- The transcript carries a spurious one-character user turn right before the agent's greeting.
- Explicit reply requests raise a timeout error waiting for `generation_created` or a similar provider acknowledgment.
- The timeout fires on calls where the model demonstrably does speak and the audio reaches the caller.

## What is actually happening

To make a realtime model start a turn without user audio, the framework needs something for the model to respond to. It pushes a one-character placeholder text message into the session to simulate an initial prompt. The framework then waits for the provider's `generation-created` event before it resolves the reply future and considers the turn successfully started.

This approach was written for text-first LLM providers, and on a speech-to-speech provider, both halves of the strategy are wrong.

First, the placeholder is treated as real conversational content. The model reads a lone punctuation character as the caller's actual turn. The model reads a lone punctuation character as the caller's turn and shapes its opening greeting as a response to it. This is why the greeting sounds subtly off, and why the first transcript line belongs to a phantom user instead of the agent.

Second, the provider does not reliably emit a `generation-created` event for this specific type of client-initiated text kickoff. The provider begins generating audio and streaming it down the websocket, but the explicit server event the framework expects never arrives. Because this event never arrives, the pending generation future remains unresolved while generation is in fact under way. Eventually, the explicit reply call raises a timeout exception in your application code, even while the model generation is in fact under way and the caller is hearing the greeting.

## How to confirm it

1. Log every conversation item the framework creates on a failing call, including the items your code did not explicitly author. Expect to find a one-character text item injected right at the start of the session. If this item is present, the framework is forcing the turn.
2. Time your explicit reply request from the function call to the exact millisecond the timeout exception fires, and compare it against the arrival time of the first agent audio frame. Expect the audio frame timestamp to fall within the timeout window. An audio frame arriving while the future is still pending confirms the provider is generating speech but skipping the event acknowledgment.
3. Search the installed framework source code for the payload it injects to start a turn. Expect to find a hardcoded string containing a single punctuation mark. Check which public API method it uses to send this string; if it uses the public turn API, it will inevitably appear in the conversation transcript.

## The fix

I patched the plugin's kickoff path directly in the installed source. The goal was to remove the false conversational context while still forcing the model to generate a response.

I replaced the punctuation placeholder with an explicit internal control marker. This marker was XML-shaped rather than something that reads as human speech, ensuring the model would treat it as a system directive rather than a user utterance. I pushed this payload onto the plugin's own internal message channel rather than passing it through the public turn API. Routing it this way meant it never became a conversation item, keeping the public transcript clean.

I also updated the framework's future resolution logic on the kickoff path. I modified the kickoff path to resolve the pending generation future as soon as the kickoff payload was enqueued on the transport, instead of waiting on a server event that might never arrive for this specific execution path.

Finally, I wrapped these changes as an idempotent runtime patch. I gated the patch on the model generation state, exactly like the other realtime patches in my infrastructure, so a re-import or a double application would be a safe no-op.

## What I would check first

When a reply timeout fires or an agent starts with a bizarre greeting, I run through this triage order to isolate the boundary responsible.

1. Check if the timeout is accompanied by dead air. If the caller hears absolutely nothing, you might be dealing with a severed media path or a failed audio subscription, not a state machine hang. If the caller hears a greeting while the logs show a timeout, the transport is fine and the framework's event loop is waiting on a missing acknowledgment.
2. Check the source stream for the spurious transcript character. A speech-to-speech session carries two separate transcription streams. If the spurious character comes from the inbound caller audio stream, verify whether the plugin is reading from the wrong side's transcription stream rather than an injected turn. If it appears instantly without any caller audio, the framework is injecting it locally.
3. Check the reconnect logic. If this timeout only happens halfway through a call rather than at the very beginning, inspect your websocket re-drive mechanisms. A reconnect restores the transport, not the turn. As I covered in a previous write-up, a [voice agent goes silent after a websocket reconnect](/blog/voice-agent-goes-silent-after-a-websocket-reconnect/) because the model waits for input it already consumed. Issuing a new explicit reply request there can trigger this exact same timeout if the framework relies on the same flawed kickoff logic.
4. Check the installed framework's turn-start contract. Ask the installed source what server event it waits on before it considers the turn started. The framework was written for a text-first provider, whereas speech-to-speech providers do not reliably emit generation-created events for client-initiated text kickoffs.

## Generalizing the State Machine Mismatch

This failure class generalizes to almost any agent framework that kicks off the first turn on your behalf. Ask two questions of your installed source: what content does it inject to start the turn, and what server event does it wait on before it considers the turn started.

A framework designed for text-first APIs assumes that pushing text into a session will reliably trigger a symmetrical server event. Speech-to-speech providers often treat text inputs, audio inputs, and control messages through distinct pipelines with different event guarantees.

I time each explicit reply request from the initial call down to the first agent audio frame, and I track that metric completely separately from the provider's event acknowledgment. When the audio frame latency remains normal but the provider acknowledgment metric spikes to a hard timeout, the metrics prove the audio pipeline is active while the event machine is stuck.

## Common questions

**Why does my voice agent transcript start with a single dot?**

The framework injected a one-character text placeholder to force the model to start the conversation. Because it goes through the public turn API, it appears as a user message in your transcript.

**Why does generate_reply time out when the agent is actually speaking?**

The framework awaits a generation-created event from the provider to resolve the reply future. For a client-initiated kickoff, speech-to-speech providers often omit this event, leaving the future pending until it times out.

**How do I force a speech-to-speech model to start the call?**

Push an explicit XML-shaped control marker through the plugin's internal message channel instead of a punctuation character. Then, resolve your reply future as soon as the kickoff is enqueued.

Canonical URL: https://www.dreamchasers.lt/blog/livekit-agent-opener-answers-a-phantom-turn-and-reply-requests-time-out
