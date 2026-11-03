> Originally published at https://www.dreamchasers.lt/blog/the-realtime-session-dies-on-connect-with-a-websocket-1007-close
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 20 October 2026.

The websocket closes with code 1007 because the realtime plugin built a session-setup frame using a configuration schema the provider recently deprecated. The provider treats the outdated shape as malformed payload data and drops the connection immediately instead of returning a validation error. Because the plugin constructs this frame internally before returning control to your application, you cannot fix it in your configuration. You must patch the plugin's setup construction code at runtime to emit the current schema.

## Symptoms

- The agent joins the LiveKit room and the participant shows as connected.
- The worker takes the job successfully.
- The session produces no audio at all.
- The provider closes the backend websocket almost immediately.
- The close code is 1007, with no validation message and no useful error text.
- The room state is entirely misleading. From the client's point of view, everything is healthy, but nobody speaks.

## What is actually happening

The failure happens entirely on the backend leg between your agent worker and the realtime model provider. The WebRTC connection to the caller is perfectly healthy, which is why the room looks fine.

When a realtime speech session starts, the provider expects a session-setup frame over the websocket. This frame dictates the voice, the temperature, the endpointing configuration, and the system prompt. In a managed framework, you do not build this JSON frame yourself. The LiveKit Agents realtime plugin builds the setup frame for you, translating your application-level configuration into the exact JSON shape the provider expects.

The 1007 close code means "invalid frame payload data". The provider deprecated a specific configuration schema block in its API, specifically a generation-parameters block whose key naming changed between API versions. The realtime plugin was still building the setup frame using the old schema.

When the provider receives the outdated shape, it does not respond with a field-level validation error. It simply treats the frame as malformed and terminates the websocket immediately with a 1007 close code. Nothing in your application code can prevent this natively, because your application never sees the final frame. The plugin constructs it internally and puts it directly on the wire.

## How to confirm it

1. Record the websocket close code and close reason on every session end, not just on errors. Expect to see exactly 1007 logged milliseconds after the connection opens. A 1007 close code confirms the provider rejected the frame payload rather than terminating cleanly.
2. Patch the provider client's session-connect entry point to log the full outbound setup payload at the boundary, exactly once, before the connect call. Expect to see the raw JSON dictionary the plugin is about to send.
3. Compare the logged outbound payload against the provider's current realtime API documentation. Expect to find a key in the generation-parameters block that the documentation no longer lists, or a new required key that your logged payload lacks.

## The fix

Because I work in Python on a pinned SDK version, the solution was to patch the plugin's setup-config construction directly at runtime to emit the current schema.

You cannot just change your application config dict, because the plugin will map your inputs back into its hardcoded internal representation. You must intercept the step where the plugin formats the outbound frame.

I wrote a monkey patch for the plugin's setup frame builder. The patch wraps the builder method, calls the original implementation to get the base dictionary, and then mutates the dictionary keys to match the provider's new API requirements before returning it.

Apply this as an idempotent runtime patch. Guard the replacement with a sentinel attribute on the patched class so re-import or double application is a no-op. Check for this attribute before applying the wrapper. This ensures that if the module is re-imported, or if a queue worker forks and runs the patch logic a second time, the double application is a safe no-op.

Gate the schema change on the exact model generation you are dialling. The provider only expects the new schema for the newer models. The older generation you were already working with keeps the exact bytes it was working with. If you rewrite the keys unconditionally, you will break older model generations that expect the deprecated schema.

## What I would check first

1. The exact websocket close code. A 1007 means the payload structure was rejected by the parser. A 1007 close code confirms the provider rejected the frame payload data as malformed. If you do not have the close code logged, you are guessing.
2. The exact bytes of the outbound setup payload. This rules out your own application logic. Inspecting the logged frame reveals whether the plugin emitted the deprecated generation-parameters schema before going on the wire.
3. The model version string in the request. This rules out using a new model with an old SDK. If the provider shipped a new model generation last week, their API boundary for that model likely expects the new setup shape.

## Why documentation will lie to you

When a provider deprecates a config shape, you get a close code rather than a validation error, and the only place to fix it is the layer that builds the frame.

If you read the provider's API documentation, you will see the correct, current schema. If you read the plugin's documentation, you will see the configuration shape for the newest release. Neither of these helps you debug the live incident.

You must read the installed SDK source on your worker. The documentation describes the version they shipped today. Your worker is running the version you pinned in your dependencies. Opening the local site-packages directory and reading the frame-builder function is the only way to prove what your system is actually doing.

## Gating the patch by model generation

The fix must be conditional. Realtime providers version their APIs tightly with their model generations. An older speech-to-speech model will reject the new generation-parameters block just as aggressively as the new model rejected the old one.

I have hit a similar schema rejection later in the call loop, exactly at the ten-minute mark, when the provider recycles the session. In that case, the realtime plugin reconnects and replays the conversation so far into the new session's setup frame. If you see a 1007 close code after ten minutes instead of on connect, you are hitting the history replay variant. You can read the full breakdown of that failure in my post on why a [realtime session reconnect fails with 1007 at ten minutes](/blog/realtime-session-reconnect-fails-with-1007-at-ten-minutes/).

If the reconnect opens a new session entirely without context, causing the agent to greet the caller again mid-conversation, you are missing the continuation token in this same setup frame. I covered that fix in my post on why a [voice agent goes silent after a websocket reconnect](/blog/voice-agent-goes-silent-after-a-websocket-reconnect/).

## Common questions

**Why does the realtime websocket close with code 1007?**

Code 1007 indicates invalid frame payload data. The realtime model provider is rejecting the initial session-setup JSON frame because its schema is malformed or deprecated.

**Why is the LiveKit room healthy but there is no audio?**

The WebRTC transport between the caller and your server is completely separate from the backend websocket to the AI provider. The provider rejected the setup frame and closed its leg, leaving the caller in a silent room.

**Can I fix the 1007 error by updating my agent config?**

No. The SDK plugin rebuilds the session-setup frame internally before sending it. You must patch the plugin's internal builder at runtime to emit the current schema.

**How do I see the frame causing the 1007 error?**

You must intercept the provider client's session-connect entry point. Log the full outbound setup payload just before it goes on the wire to see the exact bytes being rejected.

Canonical URL: https://www.dreamchasers.lt/blog/the-realtime-session-dies-on-connect-with-a-websocket-1007-close
