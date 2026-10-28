> Originally published at https://www.dreamchasers.lt/blog/inbound-sip-calls-produce-no-logs-or-trunk-provisioning-creates-duplicates
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 14 October 2026.

Trunk creation APIs generally create resources rather than upserting them. Any provisioning path run twice produces duplicate trunks, leaving your outbound dialer pointing to a stale ID. Separately, raw carrier trunks use an IP allowlist for inbound authenticity. If the carrier's signalling CIDR blocks are missing from that list, inbound INVITEs are dropped before your application exists. To fix both, wrap provisioning in a name-keyed create-or-update script, and pin the exact published carrier CIDRs on the inbound trunk.

## Symptoms

You run your infrastructure provisioning script, redeploy your environment, or retry a failed setup step. Shortly after, you open your provider console and see multiple SIP trunks sharing the exact same name but holding entirely different identifiers. Nobody on the team can confidently say which trunk ID the production phone numbers are actually attached to.

When your application attempts to dial out, outbound calls fail abruptly. Outbound calls fail unpredictably because your stored outbound trunk ID points to a duplicate trunk instance that lacks the expected phone number mapping.

Alternatively, your outbound calls work fine, but inbound calls never reach your application code. There are absolutely no logs, no webhooks firing, and no participant joining the session. The carrier insists they are sending the traffic. The carrier insists it is dispatching the calls, while your system records total silence. Because the INVITE is dropped before your application exists in the call path, no log, webhook, or participant is ever generated.

## What is actually happening

There are two distinct mechanisms at play here: one governing the duplicate trunks, and one governing the silent inbound failures.

Trunk creation APIs generally create resources; they do not upsert them. SIP trunking platforms often treat trunks as logical containers and do not enforce uniqueness on the display name. When your provisioning script issues a creation request, the API happily provisions a second logical container with the same name. Any provisioning path that runs more than once produces duplicates. This happens during a redeploy, a rerun after a partial failure, or when a second engineer follows the same runbook.

When the API creates this duplicate, the outbound trunk ID stored next to your phone numbers in your database becomes stale. The stored outbound trunk ID may point to any of the duplicates, leaving your dial path disconnected from the trunk associated with the phone numbers. Outbound dials using that stale ID will fail. I wrote about a similar state loss in [LiveKit SIP trunk does not exist after a container recreate or host reboot](/blog/livekit-sip-trunk-does-not-exist-after-a-container-recreate-or-host-reboot/), where transient state loss breaks the exact same routing path.

Separately, on a raw carrier trunk, the inbound authenticity check relies on an IP allowlist of the carrier's signalling addresses. SIP is a raw network protocol, and anyone can send an INVITE to an open port. On a raw carrier trunk, the inbound authenticity check is an IP allowlist of the carrier's signalling addresses. This allowlist sits at the very edge of the trunk configuration.

If those CIDR blocks are wrong or incomplete, incoming SIP INVITEs are dropped immediately at the boundary. This drop happens before your application even exists in the call path. The failure produces total silence rather than an application-level error because the traffic is rejected at the network transport layer. Your application cannot log a webhook for a network packet it never saw.

## How to confirm it

1. Query your trunk provider API to list all configured trunks by name. Expect to see exactly one inbound and one outbound trunk per environment. Seeing two or more with identical names confirms your provisioning path lacks idempotency and is blindly creating duplicates.
2. Read the outbound trunk ID stored in your database and compare it against the list of active trunk IDs in the provider console. Expect them to match exactly. A mismatch confirms your application is dialing through a ghost trunk while the real phone numbers sit on a different instance.
3. Trigger an inbound test call and inspect your ingress layer or host firewall logs for dropped UDP or TCP traffic on port 5060 or 5061. Expect to see dropped packets originating from carrier IP addresses. A dropped INVITE confirms an allowlist mismatch, not an application routing failure.
4. Cross-reference the source IP of that dropped packet with the carrier's published documentation. Expect the source IP to fall within a CIDR block that is entirely missing from your trunk's configured allowlist.

## How it works

I run provisioning as a name-keyed create-or-update operation for both directions. The script lists existing trunks, matches them on a stable name, updates them in place if they exist, and creates them only when they are absent. This is the only safe shape for infrastructure code that runs multiple times. Provisioning code is run more than once by definition, so name-keyed upserts prevent split-brain routing.

For inbound authenticity, I pin the carrier's signalling CIDR blocks on the inbound trunk directly from the carrier's own published documentation. I never populate this list by observing source addresses during staging tests. Carriers routinely route production traffic from entirely different subnets or failover datacenters. If you only allowlist the single IP you saw during a test call, the first production call from a different switch will hit your port and get dropped immediately. For outbound trunks, I set the carrier's SIP address explicitly.

I give this provisioning system a single idempotent entry point with two operating modes.

The first is an explicit mode. It provisions the exact trunks it is told about via configuration files. This handles the initial setup, deliberate configuration changes, and testing new routing rules.

The second is a self-heal mode. It sources the active phone numbers from the database of record, provisions both the inbound and outbound trunks to match, and then writes the fresh outbound trunk ID back to the exact location where the dial path reads it. When an infrastructure restart or a container recreate occurs, recovery is one command rather than a multi-step manual runbook. The system boots, detects its state, provisions exactly what is missing, and binds the correct IDs automatically.

To keep this system honest, I check the trunk count by name on a schedule. Anything above one trunk per direction constitutes drift. I also explicitly separate inbound INVITEs rejected at the trunk from those that reach the application. This ensures an allowlist error is instantly distinguishable from a routing error in my telemetry. I record every self-heal provisioning run and its outcome, giving me a clear audit trail of when the system had to rebuild its own trunk bindings.

## What I would check first

1. Look for raw network traffic on the SIP port. If the carrier claims a call is active but your application is silent, run a packet capture or check your host firewall logs. Seeing packets arrive and get dropped rules out a carrier routing issue and points directly to your IP allowlist. Seeing no packets at all means the carrier is sending them to the wrong destination IP entirely.
2. Check the trunk configuration for the inbound IP allowlist. Compare it against the carrier's official documentation. An exact match rules out the allowlist as the culprit and points to a misconfigured local firewall or a crashed application listener. A missing CIDR block confirms the trunk itself is rejecting the calls.
3. Read the trunk ID out of your database and compare it to the API response. If the ID in your database does not exist in the active trunk list, this confirms your provisioning script created a new trunk without updating the application state. It rules out a temporary network glitch as the reason for outbound call failures.
4. Inspect trunk telemetry to separate rejected inbound INVITEs from traffic reaching the application. Seeing INVITEs dropped at the trunk confirms an allowlist mismatch, whereas packets reaching the application rule out the allowlist and isolate the issue to downstream routing. This ensures an allowlist rejection is immediately distinguishable from an application dispatch failure.

## Common questions

**Why do inbound SIP calls fail without any logs?**

Incoming SIP traffic must pass an IP allowlist on the raw carrier trunk. If the carrier's signalling IP is missing from that list, the INVITE is dropped before reaching your application. You see no logs because the traffic is rejected at the network boundary.

**How do I prevent duplicate SIP trunks during deployment?**

Stop using create-only APIs in your deployment scripts. Implement a script that lists existing trunks, matches them by a stable name, and updates them in place. Create new trunks only when the name search returns zero results.

**Where should I get the IP addresses for my SIP allowlist?**

Always use the official signalling CIDR blocks published in your carrier's technical documentation. Never rely on the source IP addresses you observe during staging tests, as production calls will route from different subnets.

**How do I recover lost trunk IDs after an infrastructure restart?**

Build a self-heal mode into your provisioning tool that sources numbers from the database of record, provisions both trunks, and writes the fresh outbound trunk ID back where the dial path reads it.

If the boundary this note describes is one you would rather own, [a self-hosted deployment you control](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/inbound-sip-calls-produce-no-logs-or-trunk-provisioning-creates-duplicates
