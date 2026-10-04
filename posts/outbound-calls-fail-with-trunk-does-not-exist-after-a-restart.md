> Originally published at https://www.dreamchasers.lt/blog/outbound-calls-fail-with-trunk-does-not-exist-after-a-restart
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 18 September 2026.

When outbound dialing fails with a 'trunk does not exist' error after a restart, the SIP trunk state store has likely lost its data. The platform mints new trunk IDs on boot, rendering any cached IDs in your database or application memory invalid. The immediate fix is to catch this specific validation error, evict the stale ID, resolve the new ID by its stable name, and retry the placement exactly once. For a durable fix, implement an idempotent boot-time provisioning script that rebuilds the trunk state from your database of truth.

## Symptoms

- Every outbound call failed with a "trunk does not exist" error.
- The error referenced a trunk id that was configured and that used to work.
- Inbound was still fine.
- Nothing was deployed. The only event was a restart of the SIP or SFU stack, or a host reboot.
- The blast radius was total. Outbound was completely dead, and the only evidence was an error string that read like a configuration typo.

## What is actually happening

SIP trunk state is held in a store that is not necessarily durable across restarts. A common shape is a Redis instance deployed without persistence configured. When the platform comes back online, it has forgotten all previously provisioned trunks and mints new trunk IDs for everything it creates.

Every ID you cached in process memory, wrote into a static configuration file, or persisted next to a phone number now points at a dangling pointer. The failure remains completely invisible until the application attempts to dial out.

Because the error is a validation error rather than a call outcome, it is easy to classify as a bad number instead of broken infrastructure. A team that has not seen it before will spend the first hour checking credentials. Any resource identifier minted by a stateful service that you cache or persist elsewhere is a restart away from being a dangling pointer.

## How to confirm it

1. Check your event logs for self-heal events carrying both the old and the new trunk id per restart.
2. Look for placement failures classified specifically as trunk-not-found, separating them from carrier-side rejections.
3. Measure the time from stack restart to the first successful outbound call.

## How it works

Fixing this durably requires handling the failure at two layers. You need a narrow in-flight self-heal to catch the first dead dial, and an idempotent boot-time provisioner to rebuild state without duplication.

The in-flight self-heal targets this specific validation error. When an outbound placement fails with the "trunk does not exist" string, the dial path intercepts the failure. It evicts the cached ID, re-resolves the trunk by its stable name at the edge, and retries the placement exactly once. The freshly resolved ID is then written back to the database of truth so the next process does not repeat the discovery. Without this write-back step, the heal never converges, and every worker process re-learns the same thing.

Retrying a telephony placement is normally unsafe because you cannot tell whether the first attempt reached the carrier. It is safe here only because this specific error is raised during request validation, before any INVITE leaves the server. A retry provably cannot ring a person twice. Widen this error match, and you have built a double-dial.

The in-flight self-heal covers the gap, but the durable fix is boot-time provisioning. Trunk creation APIs generally create resources. They do not upsert. Any provisioning script that runs more than once produces duplicate trunks, leaving you guessing which one the phone number is actually attached to.

I make provisioning create-or-update by name for both directions. The boot-time unit lists existing trunks, matches on the stable name, updates the configuration in place, and creates the trunk only when absent. It runs as a one-shot unit that waits for the stack, authenticates, and re-provisions trunks from the database of record with bounded retries. Recovery after an infrastructure restart is then automated, rather than a runbook executed by hand under pressure.

## What I would check first

1. Examine the exact error string and metadata on the failed outbound placement. An explicit "trunk does not exist" points to state drift. Separate placement failures classified as trunk-not-found from carrier-side rejections.
2. Inspect the persistence configuration on the Redis instance backing your SIP stack. A Redis instance without persistence configured is the common shape, so the platform mints new trunk ids when it comes back. Check that trunk ids cached in process memory have not become dangling pointers.
3. Check your application database for duplicate trunk names. Multiple active trunks with the exact same name mean your provisioning script is not idempotent. Check trunk count by name on a schedule, because anything above one per direction is drift.

## Why an existence check will not prevent a double-dial

When you implement retry logic for failed telephony placements, you run into the limits of your task queue. An at-least-once queue retries the dial task if the worker takes too long to acknowledge it. If this happens, the same person is called twice within a minute. The queue shows one logical task, but the carrier shows two outbound INVITEs.

The guard most people reach for is checking whether the call record already exists in the database. If it exists, they abort the dial. This either never fires, or it fires on the first attempt and blocks a call that was never placed.

Two separate problems stack here. First, the call record is not authored by one writer. The platform native room-started webhook can create it before the dial RPC even returns. The record existing is not evidence that a dial happened. An existence check therefore reads a marker somebody else wrote. Second, the dangerous window is between the moment we start placing the call and the moment we record the placement. Ringing takes tens of seconds. A queue that retries on an acknowledgement timeout re-enters exactly there, when nothing has been written yet.

I use two markers with different jobs. A post-placement marker is written only by the dial path after the placement RPC returns. This is the proof that a dial actually happened. Check that field, never bare document existence.

## Sizing the in-flight lock window

A pre-placement attempt marker, written before the RPC, closes the in-flight window. Any attempt that starts inside a bounded validity window after an existing attempt marker is refused. Too short, and a retry fires while the phone is still ringing. Too long, and a failed attempt locks a number out of being called for minutes.

I size this window to a 45 s ringing timeout plus the task queue acknowledgement budget. This ensures that a genuinely in-flight attempt blocks a retry, while a dead attempt eventually releases the lock instead of wedging the number forever. Enforce this same window at webhook-processing time so the guard does not depend on which path runs first. Take both numbers from configuration rather than from memory.

## Inbound allowlists and total silence

If your provisioning script runs twice and misconfigures the inbound trunk, the symptom is total silence. You will see no logs, no webhook, and no participant, while the carrier insists it is sending the calls.

On a raw carrier trunk, the inbound authenticity check is an IP allowlist of the carrier signaling addresses. If those CIDRs are wrong or incomplete, INVITEs are dropped before your application even exists.

I pin the carrier published signaling CIDR blocks on the inbound trunk directly from their documentation, rather than relying on observed source addresses. When inbound produces no telemetry whatsoever, suspect the allowlist before suspecting your application logic. Your application is not involved yet.

Raw trunking is more work than a CPaaS SDK, and it is also where the interesting failures live. You own the allowlist, the trunk lifecycle, and the URI format, none of which a CPaaS lets you see. The trade is control and cost against a large amount of undifferentiated setup. The engineering claim here is idempotent, self-healing provisioning against a raw carrier trunk, not a carrier abstraction layer.

## Traceability and instrumentation

To make these failures visible, you need per-call event-sourced traceability with a structured failure taxonomy. When the self-heal fires, emit an event carrying the old and the new trunk ID, and count them per restart.

Separate placement failures classified as trunk-not-found from carrier-side rejections. Recovering SIP codes buried in RPC metadata allows you to build a typed SIP failure taxonomy driving retry-versus-terminal branching. A trunk missing error is a local cache invalidation signal.

Measure the time from stack restart to the first successful outbound call.

## Common questions

**Why do outbound calls fail with trunk not found after rebooting?**

The SIP stack stores trunk IDs in a memory store like Redis. If persistence is disabled, a reboot wipes the state, forcing the stack to mint new IDs.

**Is it safe to automatically retry a failed telephony placement?**

Retrying is only safe if the failure happens before the signaling leaves the server. A local validation error like a missing trunk ID meets this criteria.

**How do I prevent duplicate SIP trunks after rerunning setup scripts?**

Use a create-or-update approach keyed by the stable trunk name. List existing trunks, update them in place if they exist, and create them only if they are missing.

**Why does my at-least-once task queue call the same person twice?**

If a queue redelivers a dial task while the first attempt is still ringing, two INVITEs are sent. You need a pre-placement attempt marker to lock the number during the ringing phase.

If the boundary this note describes is one you would rather own, [taking ownership of the infrastructure underneath](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/outbound-calls-fail-with-trunk-does-not-exist-after-a-restart
