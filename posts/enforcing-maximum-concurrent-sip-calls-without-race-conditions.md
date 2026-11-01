> Originally published at https://www.dreamchasers.lt/blog/enforcing-maximum-concurrent-sip-calls-without-race-conditions
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 18 October 2026.

To prevent concurrent workers from overcommitting a SIP trunk, capacity checks and claims must occur inside the same atomic database transaction. Reading a live-session count and writing a claim in separate steps allows workers to interleave and breach the limit. For scheduled calls, point-in-time checks fail because calls occupy time, not just an instant. Project capacity over a grid of five-minute slots spanning the call's estimated duration, and admit the call against the worst overlapping interval rather than the current count.

## Symptoms

- Your SIP trunk carrier rejects outbound legs outright because you exceeded your licensed channel capacity.
- Logs show two separate dispatch workers inspecting the capacity limit at the same second, passing the check, and dialing simultaneously.
- The capacity count looks perfectly safe at the moment of dispatch, but the concurrency ceiling is breached ten minutes later.
- A single destination phone number exhausts its capacity limit and fails while the overall SIP trunk remains quiet and well below its ceiling.

## What is actually happening

This is a scheduling failure driven by two distinct bugs wearing the same costume.

The first is a classic read-then-write race condition. A dispatcher reads the live-session count from the database, decides there is room, and writes its claim in a second step. When two workers run at the same time, they interleave between the read and the write. Both read a safe number, both admit the call, and your system hands the carrier one more leg than it is licensed to carry.

The second bug is treating a scheduled call as a point in time. A voice agent call is not an instantaneous event. It occupies a SIP channel for its estimated duration. A capacity check that counts what is live right now systematically under-counts what will be live in fifteen minutes. When several long calls admitted at different moments are all still up, they overlap. The ceiling is crossed by calls that were each individually admissible at the exact moment they were evaluated.

## How to confirm it

1. Query your dispatch logs for the timestamps of the capacity read and the capacity write. Expect them to be identical in a healthy atomic system. If they are milliseconds apart, you have a read-modify-write gap where another worker can slip in.
2. Plot the estimated duration of all admitted calls on a timeline grid. Expect the peak concurrency to match your point-in-time metrics. If the timeline shows a peak that exceeds your ceiling, your admission logic is blind to call duration.
3. Look at the rejection timestamps from your carrier. Expect them to cluster during periods where multiple long-running calls overlap, rather than right at the start of a busy minute.

## How it works

I designed and operate an admission control system to make this breach unreachable. It handles capacity at the application layer, running on a single VM, and bounds the concurrency before a call ever starts. No ceiling breach has been observed here because nothing has been run at a volume that could produce one, but the design prevents it structurally.

The core rule is that the whole admission decision happens inside one atomic transaction. The transaction re-reads the capacity configuration and the live-session set after it opens. Trusting anything read before the transaction began is unsafe, because an operator can change the ceiling between the probe and the commit.

To fix the point-in-time counting problem, the system projects capacity over a grid. I tuned this to a five-minute slot granularity. Instead of looking at a single point count, the system reduces peak concurrency over these five-minute slots spanning the call's estimated duration. A new call is admitted against the worst slot it overlaps. If a call is estimated to last fifteen minutes, it must clear the available headroom across all three of the upcoming five-minute slots.

This transaction enforces two limits simultaneously: a global ceiling for the entire trunk, and a per-number ceiling. Both are evaluated together. A single phone number can easily exhaust its own specific concurrency limit while the overall trunk remains quiet, so checking only the global limit is insufficient.

To stop transactions from deadlocking or constantly retrying due to contention, I serialize admissions per capacity pool. This sits behind an exclusive lock in the database. When a dispatcher wants to place a call, it acquires the lock for that specific pool. This ensures pools do not interleave read-modify-write operations on shared counters. Calls in unrelated pools proceed in parallel without waiting on each other.

## What I would check first

1. Correlate carrier capacity rejections with dispatch timestamps to see if failures match simultaneous dispatch bursts. An exact match rules out a read-then-write race condition. A gap of a few milliseconds means you are evaluating capacity outside the transaction.
2. Compare trunk occupancy metrics at dispatch time against occupancy ten minutes later when long calls are still active. A query that only uses the current instant rules out duration awareness, confirming that overlapping long calls will breach your limits.
3. Check whether outbound placement failures are isolated to a single destination phone number or affect the entire trunk. The absence of a per-number ceiling means a single outbound campaign can exhaust the pool, which rules out targeted rate limiting and points to a global trunk exhaustion issue.

## Why application-level counters fail

When engineers first spot the capacity breach, the instinct is often to introduce an in-memory counter in the application layer. If you run a single VM, as I do here, an in-memory counter seems like a straightforward way to track live sessions. You increment the counter when the call starts, and decrement it when the call completes.

This fails because it decouples the capacity state from the scheduling state.

Even backed by a document store, a simple counter only tracks the calls that have already dialed. It cannot see the calls that are scheduled to start in ten seconds.

This is why the database must be the source of truth for capacity. The scheduling data and the capacity claim must reside together, protected by the same exclusive lock.

## Instrumenting the admission control path

When you move capacity checks inside an atomic transaction with exclusive locks, you introduce a new bottleneck. The transaction itself becomes the expensive resource. To maintain visibility, you need to instrument exactly what happens around that lock.

The first metric to track is the time spent waiting on the per-pool admission lock. If your dispatch pipeline slows down, this wait time is the first thing to move. It tells you immediately whether the delay is in the database contention or somewhere else in the application layer.

I also emit the admission outcome per attempt. This must be typed and explicitly record which ceiling bound the decision. You need to know if it hit the global SIP trunk ceiling or the per-number ceiling.

Finally, the system logs the peak occupancy per five-minute slot alongside the forecast the admission decision used. Emitting these together allows the forecast to be compared against reality later, though no such comparison has been run here yet.

## Applying limits to finite external resources

This pattern generalizes beyond SIP channels and voice agents. Any system that admits work against a finite external resource requires the same structural protection. Whether you are dealing with carrier ports, licensed software seats, or GPU slots, the constraint is identical.

The rule remains the same whether you use Postgres or a document store. The check and the claim must be the same atomic act. If the work has a duration, you have to admit against the worst overlapping interval rather than the current count.

## Common questions

**Why are outbound calls exceeding the SIP trunk channel limit?**

Dispatch workers are likely interleaving a read-then-write race condition. Two workers read the capacity count, see available headroom, and dial simultaneously before either writes their claim.

**How do I enforce maximum concurrent calls across concurrent dispatch workers?**

You must move the admission decision into a single atomic database transaction. Use an exclusive lock per capacity pool to serialize the dispatchers before they check the available limits.

**Why does peak concurrency spike above the limit later in the call?**

Your capacity check only evaluates the current point in time. Because calls have duration, multiple long calls admitted safely at different times will overlap and breach the ceiling.

**How should scheduled calls be counted against capacity?**

Reduce peak concurrency over a time grid covering the call's estimated duration. Admit the new call against the highest concurrency interval it overlaps, not the current moment.

Canonical URL: https://www.dreamchasers.lt/blog/enforcing-maximum-concurrent-sip-calls-without-race-conditions
