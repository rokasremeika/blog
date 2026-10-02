> Originally published at https://www.dreamchasers.lt/blog/livekit-sip-trunk-does-not-exist-after-a-container-recreate-or-host-reboot
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 16 September 2026.

SIP trunks and dispatch rules live in Redis, not on disk. Recreating a stock Redis container wipes this state. Your provisioning path recreates the trunk but receives a brand-new identifier. Any trunk ID you cached in your database is now stale, and outbound calls will fail silently with a missing trunk error. Fix this by enabling append-only persistence on a mounted Redis volume. Stop treating trunk IDs as durable keys: resolve trunks by name, cache the ID, and re-resolve on that specific missing-trunk error before retrying once.

## Symptoms

- After a host reboot or a container recreate, the media stack comes back and appears healthy.
- Browser calls connect without issue.
- Inbound phone calls might still route correctly.
- Outbound dialling fails quietly. The error logged in your application names a SIP trunk that does not exist.
- Nothing failed during the restart sequence itself. The stack looks fully operational until the next outbound call attempt.

## What is actually happening

When you deploy a self-hosted realtime stack, the SIP trunks and dispatch rules do not live in the media server's memory, nor do they live on disk. They live in Redis. A stock Redis container ships with no persistence configured. When you recreate the container, the entire store is emptied.

Your infrastructure comes back up. The media server is healthy. At this point, your stack has no telephony configuration. Your automated provisioning path runs against the control plane to recreate the missing trunks. The API obliges, creates the trunk, and hands back a brand-new identifier.

The breakage happens because your application's database of record is now out of sync with the media stack. Every trunk ID you cached in your own database is stale. A stale trunk ID only surfaces when a component tries to place an outbound call with it. The restart was silent, the health checks are green, and the state that routes your phone calls has been quietly reset.

When people say they self-host LiveKit, they usually mean one binary. Production is actually four co-located containerised services: the SFU, the SIP service, Redis, and a TLS terminator on port 443. These four fail independently. Only the SFU has a health check that most setups bother to probe. You are observing a silent failure in the SIP dependency while looking at a green light from the SFU.

## How to confirm it

1. Check your application logs for the failed outbound call and extract the specific trunk ID it attempted to use. The logged error will specifically name a trunk that does not exist.
2. Resolve the specific trunk by name against the media server. Expect to find the trunk you want by name, but with an entirely different ID than the one your application just tried to dial with.
3. Inspect the Redis container from the host system to see if the data directory is actually mounted. Expect to find an empty or non-existent volume mount on the host file system.

## How it works

Fixing this requires two distinct changes. Persistence handles the common case. Resolving by name survives the case where persistence was not enough.

First, you must turn on append-only persistence in Redis with a real mounted data directory. A container recreate will then keep the store intact. Since a realtime stack running on host networking has no container network namespace to hide behind, you must also bind Redis to the loopback interface with protected mode on.

Second, stop treating a SIP trunk ID as a durable key. The underlying mistake here is writing a service-minted identifier into your own database. The moment you do that, you create a coupling between your persistence and someone else's volatile cache, with no mechanism to notice when they diverge. The trunk name is the stable thing. The ID is just a cache.

I resolved trunks by name, cached the resulting ID, and placed the call. If that call failed, I matched on the narrow "trunk does not exist" error. On that specific error, the application evicted the cached ID, re-resolved the trunk by name, and retried exactly once.

You must keep the retry's error match deliberately narrow. That specific missing-trunk error is raised before any SIP `INVITE` leaves the box. This is the only reason retrying is safe. A broader match that catches failures after the `INVITE` has been sent turns a self-heal into a second real phone call to a real person.

## What I would check first

1. Check if the Redis data directory has survived the recreate. A fresh data directory points to a missing volume mount, which rules out an application-layer cache invalidation bug.
2. Check the exact text of the dial failure. The exact 'trunk does not exist' error is raised before any INVITE leaves the box, confirming the state mismatch.
3. Check the open-file descriptors held by the stack against the systemd unit's configured limit. A ceiling hit here manifests as refused network connections, which rules out missing trunks and points to resource starvation.
4. Check if the SIP container is currently restarting in a loop while the systemd unit shows as active. This discriminates between a partially failed compose project and a completely crashed systemd service.

## Boot-time SIP reprovisioning without hanging the host

A host reboot is the most common trigger for this silent outage. The core issue is that the machine starts up with no telephony configuration. Every restart, planned or not, requires a human to log in and run a provisioning step by hand unless you automate it.

I used a systemd oneshot bound to the stack unit, so it re-ran whenever the media stack did. It polled the stack's health endpoint for up to two minutes before doing anything. A unit ordered "after" the stack unit only knows the process started, not that it is actively serving. Provisioning against a media server that has not finished coming up returns errors that look like configuration problems. Polling prevents this.

Once healthy, the script took a short-lived identity token from the platform the host runs on. This token was scoped to the control plane it was calling. I never used a stored credential for this. A long-lived key file on a disk is a durable credential in the least defensible location you have. A platform-issued token binds to the machine's own identity, leaving nothing to rotate and nothing to leak in a disk snapshot.

The script then called the control plane's provisioning endpoints with eight bounded retries. The provisioning logic itself must be an idempotent create-or-update by name, not a plain create. A boot unit that runs on every restart against a create-only endpoint will produce drifting duplicate trunks. Since duplicate trunks route traffic, you will end up with multiple paths, one of which nobody is maintaining.

Crucially, every network request in this script carries both a connect timeout and a total timeout. A TCP connection that establishes and then goes quiet will sit there past any connect timeout. In a boot unit, that means the machine's startup sequence is now waiting on a dead socket. You must bound the whole request and bound the retry count to let the unit fail loudly rather than hang indefinitely.

## Restart ordering and the host network namespace

A production realtime topology sits on host networking. The media services need roughly ten thousand UDP ports for RTC (documented as 50000 to 60000), the SFU websocket on 7880, TCP fallback on 7881, TURN on UDP 3478 and TLS 5349, plus SIP signalling on 5060 with its own RTP range from 10000 to 20000. You cannot sensibly publish this many ports through the container runtime's port mapping.

Running in the host network namespace means services can collide on ports. Restart ordering has to be explicit because two versions of the same container cannot both hold the same signalling port at the same time.

I manage the compose project with a single systemd unit. The unit carries an `ExecStartPre` directive that brings the entire project down before bringing it up. A restart is exactly one deterministic action. Without this down-before-up step, a partially failed start leaves old containers alive that the next start cannot replace. You end up with a half-old, half-new set of containers fighting over exclusive host ports.

Because a realtime process holds several sockets per participant, plus relay sockets, it walks past the default Linux open-file limit long before it runs out of CPU. The systemd unit must raise the open-file limit well past the stock default. The failure mode when you hit this limit is not a crash; it is refused connections, which reads as a networking problem. The raised limit is a reasoned hardening decision taken before the ceiling is ever hit.

## Instrumenting the silent failures

The thing that makes this entire class of failure expensive is the silence. There is no error logged at restart time. The only broken path is the one that needs a trunk ID, which is the path you exercise least often in automated testing and care about most in production.

I recommend two specific additions to surface this state mismatch before a real caller experiences it. First, add a boot-time check that verifies the trunk IDs held in your own database of record still resolve correctly on the media server. This turns a silent breakage into a noticed one at the moment the mismatch happens, rather than delaying the discovery until the next call attempt.

Second, count the self-healed placements. The narrow retry logic is there to save the call, but it also hides the fact that a cache eviction was necessary. Emitting a metric every time the application successfully falls back to resolving the trunk by name tells you whether the Redis persistence is actually doing its job. A rising count of self-healed calls means the persistence layer is failing, and your volume mounts or configuration have drifted.

## Common questions

**Why did my SIP trunk ID change after a restart?**

The trunk configuration lives in Redis. If the Redis container is recreated without persistence, the state is lost. Re-provisioning the trunk mints a new identifier.

**How do I wait for LiveKit to be healthy before reprovisioning?**

Use a systemd oneshot that polls the stack's health endpoint. Set a maximum wait time of two minutes before making the first provisioning call.

**Why does my startup script hang indefinitely?**

You are likely missing a total timeout on your network request. A connect timeout alone will not catch a socket that establishes and then goes quiet.

**Does the LiveKit server health check include SIP?**

No. The media server answers and is marked healthy even if the separate SIP service is down or completely missing its configuration.

Canonical URL: https://www.dreamchasers.lt/blog/livekit-sip-trunk-does-not-exist-after-a-container-recreate-or-host-reboot
