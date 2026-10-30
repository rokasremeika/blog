> Originally published at https://www.dreamchasers.lt/blog/realtime-session-reconnect-fails-with-1007-at-ten-minutes
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 16 October 2026.

Your realtime provider is recycling long sessions on its own schedule, usually at the ten-minute mark. When the plugin reconnects, it replays the conversation so far into the new session setup frame. This includes model-role turns. Newer model generations reject that specific history shape outright and close the new connection with a 1007 invalid payload code. To fix this, patch the setup frame to carry the initial replay as client content, which the newer generation accepts. Gate this change on the model generation so older models retain their expected behavior.

## Symptoms

Every call ends around the ten-minute mark. The caller is speaking or listening to the agent, and suddenly the agent is gone while the caller is still on the line.

Short calls are perfect. Audio flows cleanly in both directions on calls under ten minutes. The system survives all testing because developer tests and evaluation runs rarely simulate a single continuous ten-minute conversation.

The session log shows a normal provider-initiated close. Immediately following that close, there is a reconnect attempt that fails with a websocket 1007 code. After the 1007, the agent process cleans up the activity and drops out of the room.

## What is actually happening

The disconnect at the ten-minute mark was not a bug. The provider recycled long realtime sessions on its own schedule, issuing a server-initiated session-recycle notice before closing the connection cleanly.

My plugin correctly received the notice and triggered a reconnect. The reconnect is a completely different code path from the initial connect, and it carries state that the initial connect did not have. To ensure the new session had the context of the last ten minutes, the plugin replayed the conversation so far into the new session's setup frame. This history replay included model-role turns, meaning the setup frame told the provider exactly what the model had already said.

A newer model generation rejected that history shape outright. The schema rules had tightened, making model-role turns in a setup frame illegal. Rather than returning a field-level error indicating which key was wrong, the provider simply treated the entire payload as malformed and closed the new connection as invalid, issuing a 1007 code.

The disconnect itself was routine and expected. The failure was that the reconnect path sent a payload the first connect never sent, and that payload was now fatal.

## How to confirm it

1. Record wall-clock call duration as a field on every call end event. Expect the durations of the failed calls to pile up at a round number, such as 600 seconds. A tight clustering at a round number confirms this is a provider-scheduled event rather than a random network coincidence.
2. Record every reconnect as a first-class event, including the close code that preceded it. Expect to see a standard recycle close code followed immediately by a 1007 on the reconnect. This sequence confirms the crash is happening on the resumption attempt.
3. Log the setup frame payload sent on the initial connect and compare it to the payload sent on the reconnect. Expect the reconnect payload to contain model-role messages in its history array. Finding these model-role turns confirms the plugin is sending the shape the newer generation rejects.

## The fix

I configured the history so the initial replay was carried as client content. The newer model generation accepts client-role content in the setup frame without rejecting the connection. I applied this fix at the same setup-frame patch point inside the plugin where schema corrections live.

Because I occasionally route traffic to older models, I gated this behavior on the model generation. The older generation kept the original behavior it was already working with, and only the newer generation received the modified client-content shape.

I verified the fix using a synthetic session-recycle injector. Sitting on ten-minute calls is an unacceptable debug loop. I built an injector that raised the exact same reconnect path a real session recycle raises, but timed it to fire after five seconds. This turned a ten-minute edit-test loop into a five-second one.

## What I would check first

1. Check the call durations on the dropped sessions. Expect exactly ten minutes. This rules out random network instability and rules in a provider-enforced boundary.
2. Check the payload structure on the second connection attempt. Expect model-role messages in the history array. This rules out missing authentication credentials and confirms a schema rejection.
3. Check whether session continuation is configured in your setup frame. Expect it to be missing if the agent greets the user again instead of dropping the call entirely. If the agent greets the caller a second time, you are hitting the failure described in /blog/voice-agent-goes-silent-after-a-websocket-reconnect/ where the session restarts cold.

## Why the reconnect path lacks test coverage

Reconnect is the path least covered by tests. Reaching it honestly takes as long as the provider's session lifetime.

Any long realtime session has this shape, whichever vendor you are on. The client must maintain context across physical connections, which means it will attempt to serialize the state of the room and send it upstream. If the upstream provider changes its schema validation for history payloads, the connection will drop.

## Patching plugin internals in production

When the SDK itself is the blocker, you have to patch the plugin internals in production. The plugin builds the session-setup frame for you, abstracting away the raw JSON payload.

I wrote the patch against the installed SDK source. I intercepted the setup-config construction, inspected the target model generation, and mutated the history array before it went over the wire.

## Common questions

**Why does my voice agent drop the call at exactly ten minutes?**

Your realtime provider recycles long sessions around the ten-minute mark. Your agent attempts to reconnect but sends a payload the provider rejects, killing the call.

**What does websocket close code 1007 mean during a reconnect?**

Code 1007 means invalid frame payload data. During a reconnect, your agent is sending a conversation history shape in the setup frame that the model considers illegal.

**How do I test a ten-minute session timeout without waiting?**

Build a fault injector at your session event boundary. Create a timer that triggers the exact session-recycle notice and reconnect path after five seconds.

**Why does the reconnect path fail when the first connect succeeds?**

The reconnect path carries state the initial connect did not have. It replays conversation history, including model-role turns, which the newer schema rejects on session setup.

Canonical URL: https://www.dreamchasers.lt/blog/realtime-session-reconnect-fails-with-1007-at-ten-minutes
