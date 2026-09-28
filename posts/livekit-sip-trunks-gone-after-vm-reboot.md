> Originally published at https://www.dreamchasers.lt/blog/livekit-sip-trunks-gone-after-vm-reboot
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 12 September 2026.

When a voice stack restarts, volatile stores wipe SIP trunks. Browser calls work because the SFU is healthy, but telephony routing fails silently. The fix is a systemd oneshot unit that polls the stack's health endpoint for up to two minutes, then re-provisions trunks automatically. The script must use a short-lived platform token rather than a stored credential, enforce both connect and total timeouts to prevent a stalled socket from hanging the boot sequence, and use idempotent create-or-update requests so successive reboots do not duplicate routing rules.

## Symptoms

- The host reboots or the media containers are recreated.
- The stack comes back online, health checks pass, and browser WebRTC sessions connect normally.
- Outbound phone calls fail silently. The error logged states the SIP trunk does not exist.
- Inbound routing may still work while outbound dialling fails.
- A manual run of your provisioning step fixes the routing.

## What is actually happening

"Self-hosted LiveKit" is not a single binary. Production requires four co-located containerised services: the SFU, the SIP service, Redis, and a TLS terminator on port 443. TURN is not a fifth process. It is a capability enabled inside the media server, exposed on its own ports and reached through the layer-4 TLS route, which is exactly why people forget to think about it. These services fail independently, and only the SFU usually has a health check that anyone bothers to probe.

SIP trunks and dispatch rules do not live in the media server's configuration files on disk. They are derived state written into Redis. A stock Redis container ships with no persistence configured. When the host reboots or the container is recreated, the store empties completely.

Your database of record holds the canonical telephony configuration, but the realtime node has forgotten it. The node is running, the SFU is accepting browser connections, and the SIP service has no routing rules.

If you have a naive startup script ordered after the stack unit, it is likely firing before the SIP service is genuinely serving traffic. The script executes against an endpoint that is not yet ready, returns an error that looks like a configuration problem, and exits. The boot finishes, but telephony is unprovisioned.

If you successfully re-provision the trunks later, the service mints brand-new identifiers. Any identifiers you cached in your database are now stale. You only discover this silent reset when something tries to place a call with the stale trunk id.

## How to confirm it

1. Pull the active trunk list from the running service API immediately after a restart. Expect the response to omit your trunks, which confirms the volatile store was wiped.
2. Attempt an outbound dial using the trunk identifier cached in your database. Expect the specific "trunk does not exist" error, confirming the application is holding a stale identifier rather than failing at the SIP transport layer.
3. Inspect the host filesystem outside the container for the Redis data mount. Expect an empty directory or missing volume; this confirms that append-only persistence is either disabled or writing to an ephemeral container path.

## How it works

I run a systemd oneshot bound to the main stack unit, so it executes every time the stack restarts. Three specific details in this unit prevent it from failing under edge conditions.

The provisioning call must wait for readiness. A unit ordered "after" the stack unit only knows the system process started, not that it is handling requests. The oneshot polls the stack's health endpoint with a ceiling of up to two minutes before making its first configuration attempt.

It authenticates to the control plane using identity from the platform. It requests a short-lived token scoped to the control plane and to the machine's own identity. There is no long-lived secret stored in a file on disk, which means there is nothing to rotate and nothing to leak in a snapshot.

The network call carries both a connect timeout and a total timeout. This is the detail most scripts skip. A TCP connection that establishes and then goes quiet will sit there past any connect timeout. In a systemd boot path, the machine's startup sequence is now waiting indefinitely on a dead socket. Bounding the total request and capping bounded retries to eight attempts lets the unit fail loudly rather than hang the host.

The configuration payload must be idempotent. The provisioning logic relies on create-or-update by name. A startup script firing blindly against a create-only endpoint leaves duplicate drifting trunks on every restart. Duplicates still route, leaving you with identical active trunks, only one of which you are actually maintaining.

## What I would check first

1. Curl the SIP service health endpoint separately from the SFU health endpoint. Expect to see whether the SIP signalling is answering, because folding SIP health into overall stack health is an open gap.
2. Review the logs of the provisioning systemd unit for the retry attempt count. Expect to see the count incrementing. A climbing count is the sign that the health poll is finishing before the stack is genuinely ready.
3. Compare the trunk identifier returned from a manual API provisioning call against the one in your database. Expect the strings to differ entirely. This confirms your system is persisting a transient cache key rather than resolving the stable name.

## The network namespace constraints

Realtime media requires publishing roughly ten thousand UDP ports, plus SIP and TURN ports. You cannot sensibly publish that range through container port mapping.

The containers run in the host network namespace. This means they are no longer isolated from each other or the host network. Redis is exposed directly. I bind Redis to loopback with protected mode enabled, because on host networking there is no container network to hide behind.

The stack binds to explicit host ports: 7880 for the websocket, 7881 for TCP fallback, 50000 to 60000 for RTC UDP, 5349 TLS and 3478 UDP for TURN, 5060 for SIP signalling, and 10000 to 20000 for RTP.

Because host ports are exclusive, a restart must be a single deterministic action. I configure the supervising systemd unit with an `ExecStartPre` command that explicitly tears the compose project down before bringing it up. Without this down-before-up ordering, a partial failure leaves stale containers holding host ports, preventing the new containers from starting. You end up with a half-old stack fighting itself.

## Treating identifiers as caches

A container recreate mints new trunk identifiers. Treating a service-minted identifier as durable is the underlying mistake. The moment you write one into your database of record, you couple your persistence to a volatile third-party cache with no mechanism to notice when they diverge.

The trunk name is the stable identifier. The identifier string is a cache.

I resolve trunks by name and cache the identifier locally. When outbound dialling encounters the narrow "trunk does not exist" error, I evict the cache, re-resolve by name, and retry exactly once.

The error match for the retry is deliberately narrow. That specific error is raised before any INVITE leaves the box. A broader error match risks catching a failure that occurs after the INVITE is sent, which turns a self-heal into a second real phone call to a real person.

Append-only persistence on a mounted directory handles the standard container recreate path. The resolve-by-name self-heal catches the cases where the volume was missing, restored from an older state, or started fresh. A boot-time check that the trunk ids held in your database still resolve would turn a silent breakage into a noticed one at the moment it happens. Counting self-healed placements would tell you whether persistence is actually doing its job, because a rising count means it is not. Neither is built, but both are cheap.

## Open file descriptor exhaustion

A realtime process hits the stock open-file limit long before it runs out of CPU. Every participant holds signalling sockets, media sockets, and relay sockets.

When a process hits this ceiling, it does not crash. It simply refuses new connections. This looks identical to a networking problem.

I raise the open-file limit well past the stock default directly on the supervising systemd unit. This is reasoned hardening ahead of the ceiling, not a ceiling I have measured under sustained load.

## Common questions

**Why do SIP trunks disappear after restarting LiveKit?**

SIP trunks and dispatch rules live in Redis, not on disk. If Redis lacks append-only persistence on a mounted volume, a restart wipes the memory store and destroys the configuration.

**How do I prevent a curl startup script from hanging the boot?**

You must explicitly set both a connect timeout and a total maximum time limit. A socket that connects but stalls indefinitely will block the systemd sequence unless a total timeout terminates it.

**Why are my outbound phone calls failing with trunk does not exist?**

Your application is caching a trunk identifier that was minted before a restart. When the trunk was re-provisioned, the service assigned it a new identifier, rendering your cached key stale.

**Can I use Docker port mapping for the media UDP range?**

No, you cannot sensibly publish ten thousand UDP ports through standard container port mapping. The media services must run in the host network namespace to operate properly.

Canonical URL: https://www.dreamchasers.lt/blog/livekit-sip-trunks-gone-after-vm-reboot
