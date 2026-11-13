> Originally published at https://www.dreamchasers.lt/blog/task-queue-executions-climb-indefinitely-with-no-visible-errors
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 30 October 2026.

When a task queue shows a monotonically growing execution rate but zero errors, a payload has likely been enqueued against the wrong self-rescheduling handler. If an accounting payload reaches a reconciliation loop that reschedules itself every five minutes, the handler processes the unknown payload silently and schedules its successor. Each dial attempt forks a brand new immortal chain. You must bind the payload type to the handler route at the enqueue boundary, and force the handler to reject unrecognized payloads instead of blindly perpetuating them.

## Symptoms

- Task queue executions climb with dial volume and never come back down.
- Nothing errors, nothing retries, and no reservation is affected.
- Every individual execution looks functionally normal and exits with a success code.
- The system produces a background execution rate that grows monotonically with outbound dial volume.
- A cost line tracking worker compute mirrors this execution rate perfectly.
- Nobody notices the bug during functional testing because no call quality or scheduling logic fails. The shape of the bug has to be found by reading code rather than by noticing an outage.

## What is actually happening

A usage-accounting task was enqueued against the reconciliation handler route rather than its own dedicated route. The reconciliation handler contract is to reschedule itself at the end of every execution. This is correct behaviour for the loop it belongs to, but it is catastrophic for a payload that was never meant to enter that loop.

In a distributed call scheduling system, a reconciliation loop runs on a recurring schedule to audit system state. A reconciliation handler often runs as a continuous cycle. Instead of relying on a centralized scheduler, the worker processes its batch and enqueues its own subsequent execution with a five-minute delay. This design relies on each execution enqueueing its successor, which functions correctly until an unexpected payload enters the loop.

When the usage-accounting payload arrived at the reconciliation route, the worker deserialized it. Because nothing in the self-rescheduling handler asked whether it should exist, it completed its run without error. It completed its run without throwing an exception and executed its final instruction: schedule the next tick.

Each misrouted task became the root of a brand new self-rescheduling five-minute chain. One chain was created per dial attempt. The chains never terminated because nothing in a self-rescheduling handler asks whether it should exist. The implication of the five-minute reconciliation cadence is that, by arithmetic alone, each forked chain adds twelve executions an hour indefinitely. By arithmetic alone, each forked chain adds twelve executions an hour indefinitely, accumulating with every dial attempt. The system stayed functionally correct throughout. This is a cost and blast-radius bug, not a call-quality one, which is exactly why it survived normal code review.

## How to confirm it

1. Group your task queue executions by handler route and chart them over a twenty-four hour window. Expect to see one specific handler route climbing linearly with outbound call volume, while the rest remain flat. If you see the reconciliation route execution rate climbing in lockstep with dial volume while other handlers stay flat, you have found the leak.
2. Inspect the raw payload of a recent execution on the runaway handler. Expect to find fields belonging to a completely different domain, like usage-accounting keys or dial-state identifiers, rather than the expected reconciliation state. This proves the route mismatch occurred at the source.
3. Audit chain identity directly by counting distinct chain roots currently alive rather than bare executions. Expect to find the count of live chain roots climbing above one with dial volume rather than remaining a small constant.

## The fix

The remediation this points to requires a structural design change, rather than just pointing the task at the right route and walking away. You must bind the payload type to the handler route at the enqueue boundary and assert it. Do this in the same place the deterministic task name is asserted, so a mismatch fails loudly at enqueue instead of quietly at execution.

Assert the payload type against the handler route directly at the enqueue boundary, in the same place the deterministic task name is asserted. Binding the payload type to the handler route ensures that an invalid route pairing fails immediately at task creation. The client validates that the route string exactly matches the expected handler for that specific payload class. If an accounting payload is submitted to the reconciliation route, the assertion fails loudly at enqueue instead of quietly at execution.

Next, make self-rescheduling handlers refuse to reschedule on a payload shape they do not recognize. This turns a wrong task in the loop into a single logged rejection rather than a new immortal chain. The handler must explicitly validate that the data it receives is something it has the authority to perpetuate.

Finally, audit chain identity directly. Count distinct chain roots rather than bare executions. The count of live loops is the number that should be exactly one. The count of executions is the number that hides the problem. One live chain is the correct count. Anything else is a bug.

## What I would check first

1. Check the queue error rate. A zero error rate rules out standard poison-pill retries and points to a self-perpetuating success path.
2. Check the count of distinct chain roots currently alive against the expected baseline. A count exceeding one confirms that dial attempts are forking immortal poll chains.
3. Check the cadence of the executions. A strict five-minute interval between runs matches the reconciliation loop schedule, pointing to a misrouted payload caught in that specific loop.

## Why code review misses routing bugs

The most dangerous aspect of this failure mode is its silent correctness. When an engineer reviews an enqueue call, they look at the payload construction and the target queue. If the route string is imported from a shared constants file, a slip of autocomplete can easily select the reconciliation queue instead of the usage-accounting queue.

Because the payload is valid JSON and the queue exists, the enqueue operation succeeds. Because the reconciliation worker is designed to catch generalized state payloads and ignore fields it does not care about, it does not crash. The worker simply processes the task, finds no actionable reconciliation work, and schedules the next tick. The only artifact is a compute bill that slowly creeps upward.

## Counting loops instead of executions

The reason this bug survives in production is a failure of instrumentation. We typically monitor task queues by looking at throughput, execution latency, and error rates. When this bug occurs, throughput goes up, latency remains low, and errors remain at zero. By all standard metrics, the system appears exceptionally healthy.

To catch this, you need instrumentation focused on distinct chain roots currently alive, which should be a small constant. This is the number that would have caught this anomaly immediately had anything been counting loops instead of executions. You also need executions per handler broken out by payload type, so a payload arriving at a handler that should never see it is visible. Finally, enqueue-time assertion failures must be counted, because a rise means something upstream is routing wrongly.

## The limits of this diagnosis

I must state explicitly that while this defect occurred in a production distributed scheduling layer, my confidence is medium regarding how identically this presents across every third-party task framework. I have not mapped the exact routing mechanics of every available queue provider.

However, the structural vulnerability generalizes completely. It applies to any queue where the handler route is chosen at enqueue time and any handler in the system reschedules itself. The dangerous combination is a routing mistake meeting a self-perpetuating contract, and it exists on every durable-task platform. The two defences carry over regardless of the framework you use: assert the payload and route pairing where the task is created, and never let a self-rescheduling handler perpetuate a payload it cannot definitively identify.

## Common questions

**Why are task queue executions climbing with no errors?**

A payload was likely misrouted to a self-rescheduling handler. The handler processes the task silently and schedules a successor, creating an immortal chain.

**How do I stop a self-rescheduling queue loop?**

You must configure the handler to explicitly reject and drop payloads it does not recognize. This prevents unknown tasks from being passed to the next execution tick.

**Why does my reconciliation worker run more often than scheduled?**

Outbound dials or external events are enqueuing tasks against the reconciliation route by mistake. Each errant task spawns a parallel execution chain that runs on the same cadence.

**How do I monitor runaway queue loops?**

Track the count of distinct chain roots rather than bare executions. The number of active roots for a scheduled loop should be exactly one.

Canonical URL: https://www.dreamchasers.lt/blog/task-queue-executions-climb-indefinitely-with-no-visible-errors
