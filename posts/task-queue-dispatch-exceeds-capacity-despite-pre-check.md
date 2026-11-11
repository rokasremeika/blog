> Originally published at https://www.dreamchasers.lt/blog/task-queue-dispatch-exceeds-capacity-despite-pre-check
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 26 October 2026.

A single-stage admission path forces a structural failure under load. Opening a transaction for every scheduled task means rejected calls consume database resources and contend with valid ones. Trusting a fast read outside a transaction leads to race conditions where a call is dialed but capacity is already gone. The fix is a two-stage admission path. Probe capacity cheaply outside the transaction to drop hopeless candidates early, then perform an authoritative re-check atomically inside the transaction immediately before placement. The atomic result always wins.

## Symptoms

- Dispatch throughput collapses under a burst. Transactions pile up, contend, and retry during a burst while most of them were always going to be rejected for want of capacity.
- Transactions pile up, contend, and retry in your database or document store, even though the system was always going to reject most of them for lacking channel capacity.
- A dispatcher trusts a fast pre-check and dials a call the authoritative check would have refused.
- Legitimate inbound calls are starved and rejected because an outbound dialer surge consumed all available slots.

## What is actually happening

Transactions are the expensive, contended resource in any scheduling system. When a burst of scheduled tasks wakes up, they all want to run at exactly the same time. If you use a single-stage admission path, you are forced into a structural trade-off.

Opening a transaction for every due call means the calls that cannot possibly be admitted still pay the full price. They pay the full transaction cost and contend with the calls that can actually be admitted. The database spends all its time rejecting tasks instead of placing them.

The mirror failure is skipping the transaction entirely because a cheap read said there was room. That cheap read is just a hint from the past. By the time the dial RPC fires and hits the carrier network, the capacity that the read saw may be completely gone. A fast read authorises a call that a strict transactional check would have blocked. Both of these are structural properties of a single-stage admission path.

## How to confirm it

1. Graph your database transaction contention against the scheduled task queue depth during a burst. Expect transaction contention and retries to rise with the volume of due tasks even when channel capacity is exhausted. This confirms that doomed calls are contending for database resources.
2. Count the number of placed outbound dials against your authoritative concurrency check. Expect to see placed dials exceeding the strict limit if your dispatcher is trusting a stale read outside of a transaction boundary.
3. Measure the inbound call rejection rate during a burst of scheduled outbound tasks. Expect to see inbound calls dropping when outbound tasks surge. This confirms that both traffic types are competing blindly for the exact same admission transaction pool.

## How it works

I two-stage the admission path. I run a cheap probe outside any transaction to drop the hopeless candidates early, then I re-decide atomically inside the transaction immediately before placement.

The atomic result wins every time. The probe has only one job: to reduce transaction pressure. It is never allowed to authorise a dial. If the probe says the system is full, the task backs off immediately without touching the database. If the probe says there is room, the task is allowed to attempt the expensive transaction.

Inside that transaction, the system performs an authoritative re-check of the capacity pool and peak overlapping reservations before claiming a slot. If you want the exact database mechanics of that inside-transaction check, I wrote about [enforcing maximum concurrent SIP calls without race conditions](/blog/enforcing-maximum-concurrent-sip-calls-without-race-conditions/). The core rule here is that the final decision happens under a lock, but we only acquire that lock if the cheap probe told us it was worth trying.

## What I would check first

1. Look at the ratio of opened admission transactions to successful channel allocations. Expect opened transactions to track admitted placements rather than total scheduled volume. If transaction volume scales with total due tasks rather than available channel capacity, transactions are contending before admission is decided. This rules out network capacity limits and points entirely to transaction pressure.
2. Check the time elapsed between your capacity read and the actual dial RPC execution. Expect this gap to be under a few milliseconds. If the gap spans network calls or complex business logic, your read is stale. This rules out accurate capacity tracking and confirms a race condition.
3. Review the concurrency limits for inbound versus outbound traffic. Expect to find a shared pool with no priority mechanism. If they share a hard cap without deferral logic, this rules out network capacity issues and points directly to your outbound dispatcher starving your inbound routes.

## Instrumenting the disagreement gap

The most dangerous part of an optimistic filter is that it silently drifts. If the probe logic becomes too slow, or if the cache it reads from falls behind, it stops being a helpful hint and starts being a liability.

To fix this, I instrument the gap between the two stages. I keep a counter of how often the probe and the authoritative transaction disagree. If the probe says there is room but the transaction says there is no capacity, that is a disagreement.

I emit this counter into the event log as a first-class number. It is worth watching as a service level of its own, though nothing here watches it today. A rising disagreement rate means the probe is reading state that is too stale to be useful. It is buying you nothing but risk.

This disagreement counter is the part almost nobody builds, and it is the only thing that tells you whether the optimisation is still safe six months later.

## Deriving dispatch mode from the task name

Task queues often retry jobs. Sometimes a job needs to run aggressively, and sometimes it needs to act as a gentle backstop.

Two dispatch rules sit alongside the admission control logic. The handler derives its own mode strictly from the task name prefix. A primary dispatch and a backstop dispatch run the exact same underlying code, but they run with different aggressiveness profiles.

By pulling the mode from the task name prefix rather than a field inside the JSON payload, a handler can never guess or inherit the wrong role. If a payload gets corrupted, or if an engineer manually re-enqueues a job with a missing field, the task name remains intact.

## Protecting inbound traffic from outbound bursts

Admission control protects shared channel capacity as well as database availability.

When inbound traffic surges, outbound dispatch defers rather than competing with it for the same channels. An inbound call is a real human being waiting on the carrier network right now. If you reject them, the call is gone permanently.

An outbound call is a scheduled intent. A missed outbound attempt is reschedulable. If the system is busy processing a spike of inbound activity, the outbound dispatcher simply logs a deliberate deadline overcommit, backs off, and tries again later.

I record the count of deliberate deadline overcommits. This makes the cost of the policy visible.

## Common questions

**Why does dispatch throughput drop when scheduled calls burst?**

Doomed calls are opening database transactions and contending for locks. You are paying the full database cost to reject a call because you lack a fast pre-check.

**Can I just check capacity before dialing to save database load?**

A cheap read is a hint from the past. By the time the dial RPC fires, the capacity may be gone, leading to over-commitment. You must always re-verify inside a transaction.

**How do I know if my pre-check is reading stale data?**

Keep a counter of how often the pre-check and the authoritative transaction disagree. A rising disagreement rate means the probe is reading state too stale to be useful and is buying you nothing but risk.

**Why do inbound calls drop when the outbound dialer starts?**

They are competing for the same channels without priority. Outbound tasks must defer during inbound surges because outbound is reschedulable and inbound is not.

Canonical URL: https://www.dreamchasers.lt/blog/task-queue-dispatch-exceeds-capacity-despite-pre-check
