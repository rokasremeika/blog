> Originally published at https://www.dreamchasers.lt/blog/inbound-callers-vanish-when-the-sip-trunk-allowlist-silently-overflows
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 1 November 2026.

When legitimate inbound callers are silently dropped before reaching your telephony application, your per-trunk caller allowlist has likely exceeded its size limit. In a multi-tenant environment, individual tenant additions aggregate until they silently breach the trunk's capacity. The carrier rejects the over-limit numbers before they generate a SIP INVITE, leaving no trace in your logs. To fix this, build the allowlist as an explicit union across assigned tenants and apply hysteresis to the capacity check: warn at 80 percent, flag overflow at 100 percent, and clear the warning only when usage drops below 80 percent.

## Symptoms

- Inbound calls from known, permitted numbers fail to connect, often dropping before the caller even hears a ringing tone.
- Your telephony application records zero trace of the call: no session, no routing decision, and no log line.
- From your side, the call simply never happened.
- The issue appears inconsistently. Older numbers continue to connect without issue, while newly assigned numbers for a tenant fail entirely.
- Because your application cannot report on calls it never receives, the complete absence of errors leads you to assume traffic was dropped inside your own infrastructure.

## What is actually happening

A per-trunk caller allowlist has a hard size limit enforced by the carrier. When the union of permitted numbers across every tenant assigned to that trunk exceeds this limit, the entries that do not fit are simply absent from the carrier's routing table.

Capacity is consumed by an aggregate that nobody explicitly owns. Each tenant addition is individually reasonable, and the total crosses a line no single change is responsible for.

Because the limit is breached at the provider edge, the rejection happens before your code is ever involved. The carrier drops the unauthorized traffic without forwarding a SIP INVITE to your network. Your application cannot log a session it never receives, leaving you completely blind to the failure. The tenant assignment that caused the overflow is often days in the past, entirely divorcing the failure from your most recent deployment.

## How to confirm it

1. Count the distinct permitted numbers across all tenants assigned to the affected trunk. Expect this total to exceed the provider's documented allowlist limit, confirming an overflow state.
2. Inspect the provisioned allowlist payload directly against the carrier trunk limit. Expect newly assigned tenant numbers beyond the capacity threshold to be absent from the active carrier allowlist.
3. Temporarily remove one known-working number from the tenant configuration to free up a slot, add the failing number, and retry the test call. Expect the call to succeed, which isolates the issue strictly to capacity exhaustion rather than a malformed number format.

## How it works

I build the allowlist as an explicit union across all assigned tenants. Rather than relying on independent updates from isolated tenant configurations, a centralized builder aggregates the permitted numbers, deduplicates them, and calculates the total payload before interacting with the trunk API. If you have tenants sharing identical resources, you also need strict rules for how traffic is directed once it gets through. I wrote about that handling in [Deterministic Inbound SIP Routing When Tenants Share A Phone Number](/blog/deterministic-inbound-sip-routing-when-tenants-share-a-phone-number/).

I give the capacity check hysteresis rather than relying on a single threshold. A naive check at the boundary flaps constantly. If one number is added and removed repeatedly right at the limit, it toggles the alert state and trains everyone on the engineering team to ignore it.

To prevent the capacity check from flapping, I tuned the thresholds deliberately. I warn at 80 percent of allowlist capacity. I flag an overflow and increment a persistent counter at exactly 100 percent. I clear the state only when usage drops back below 80 percent. This creates a buffer zone that forces capacity planning rather than reactive firefighting.

I run a pre-flight inspection before any enforcement rollout that inspects each trunk's policy mode. This process refuses to proceed if any trunk is already enforcing rather than notifying. A capacity mistake must produce a notification you can act on, instead of a caller who cannot get through. Failing the pre-flight inspection when any trunk enforces rather than notifies ensures capacity mistakes yield actionable notifications instead of dropped calls.

For instrumentation, I log the allowlist size against the per-trunk limit on every rebuild, per trunk. I keep a persistent overflow counter in the metrics, and I treat any non-zero value as a correctness event rather than a statistic. I also keep rejections at the trunk and rejections by my own routing logic as entirely separate numbers, ensuring that edge drops never pollute the application-layer admission metrics. If you are struggling with absent logs during provisioning, you might also be dealing with API idempotency issues, which I cover in [Inbound SIP Calls Produce No Logs, Or Trunk Provisioning Creates Duplicates](/blog/inbound-sip-calls-produce-no-logs-or-trunk-provisioning-creates-duplicates/).

I built and operate this mechanism on a raw carrier SIP trunk where overflowing numbers are silently omitted, but boundary behavior across carriers varies. Because the rejection happens before your code is involved, you cannot rely on an inbound SIP rejection code reaching your network. You must verify your carrier's trunk-level behavior directly, because your application cannot report on calls it never receives.

## What I would check first

1. The total count of provisioned numbers against the carrier's hard capacity limit. Expect a breach to confirm the allowlist overflow, ruling out any application-layer routing bugs.
2. The trunk policy mode in your infrastructure configuration. Expect an "enforce" state to explain the silent drops, whereas a "notify" state rules out the allowlist as the cause of the missing inbound calls.
3. Trunk-level rejections against application-level routing rejections. Expect non-zero rejections at the trunk to isolate capacity drops at the carrier edge from downstream routing logic failures.

## The Danger of Shared Capped Resources

Any shared, capped resource fed by independent contributors needs an owner for the aggregate and hysteresis on the threshold. In a multi-tenant telephony system, no individual tenant cares about the total trunk capacity. They only care that their specific numbers are permitted.

Without an aggregate owner, the system drifts toward failure automatically. A tenant adding a single phone number is an operation that should always succeed. When it fails because another tenant added numbers days earlier, the failure boundary is drawn in the wrong place. The engineering team must own the aggregate limit, monitor its growth, and provision new trunks or request limit increases long before the threshold is breached.

Warning at 80 percent of capacity provides actionable visibility before callers are dropped.

## Securing Visibility Before the Application

Any control that rejects traffic before your application runs needs its own visibility. Your application cannot report on calls it never receives, which means the absence of errors is not evidence of the absence of rejections.

When a carrier drops a call, they are doing exactly what you told them to do: blocking unauthorized traffic. They view the overflowed numbers as unauthorized because they never received them in the allowlist payload. To them, the system is working perfectly.

By treating the generation of the allowlist as a critical control plane operation, you move the failure domain from the carrier's black box into your own observable infrastructure. Logging the size of the list during the rebuild step is the only way to prove what the carrier actually knows. If the rebuild process logs an overflow, you know exactly why the calls are failing, even if the SIP traces remain completely empty.

## Common questions

**Why do legitimate inbound calls drop without leaving any logs?**

A trunk-level caller allowlist likely exceeded its size limit. The provider rejects overflowing numbers before they reach your network, so no SIP INVITE or error log is ever generated in your application.

**How do I stop capacity alerts from flapping on every update?**

Implement hysteresis by using multiple thresholds for the capacity check. Warn when capacity reaches 80 percent, flag an overflow at 100 percent, and clear the warning only when usage drops back below 80 percent.

**What should I check before rolling out trunk policy changes?**

Run a pre-flight inspection to verify the trunk policy mode. Refuse to proceed if the trunk is set to enforce rather than notify, ensuring capacity mistakes produce actionable notifications instead of dropped calls.

**How do I monitor an allowlist that rejects calls outside my system?**

Log the allowlist size against the trunk limit on every configuration rebuild. Maintain a persistent overflow counter and treat any non-zero value as a serious correctness event rather than a standard operational statistic.

If the boundary this note describes is one you would rather own, [a self-hosted deployment you control](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/inbound-callers-vanish-when-the-sip-trunk-allowlist-silently-overflows
