> Originally published at https://www.dreamchasers.lt/blog/opaque-rpc-errors-hide-the-sip-response-codes-you-need-for-outbound-call-retries
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 28 September 2026.

When an outbound SIP call fails, the client frequently throws an opaque RPC error string that hides the underlying SIP response code. Without that code, you cannot distinguish a busy signal from a disconnected number, and your retry policy collapses. To fix this, build a cycle-safe recursive function that walks the error's metadata and trailers to recover the three-digit protocol code. Map that code to a closed set of typed outcomes (such as no-answer, busy, or invalid number). Pass only the typed outcome to your retry logic so it can safely retry busy numbers without hammering dead ones.

## Symptoms

- Failed outbound placements all surface as the same opaque RPC error string in your application logs.
- You cannot separate a normal "they were busy" condition from "the number is dead" or "our trunk credentials broke".
- Your task queue retry policy collapses into an all-or-nothing toggle.
- If you configure the queue to retry everything, you end up hammering disconnected numbers and burning your carrier relationship.
- If you configure the queue to retry nothing, you drop calls that would have easily connected on the second attempt.

## What is actually happening

When a carrier rejects or fails to complete an outbound call, they respond with a standard three-digit SIP status code. However, the RPC client that your application uses to interface with the calling platform frequently drops this protocol code from the top-level exception it throws.

The SIP response code still exists, but it lives buried in the RPC error's metadata or trailers. It appears under one of several possible key spellings, depending on the specific trunk and the vendor routing the call. Sometimes the code is nested a level or two down inside a dictionary. Sometimes it is only embedded within a human-readable text message string.

Without recovering this underlying protocol code, your application is forced to classify failures by running substring matches against the error message text. This approach degrades rapidly because the wording of an error message is prose. A vendor is completely free to change their error text without telling you, which silently breaks substring matching and causes recoverable calls to be dropped.

## How to confirm it

1. Inspect the raw error object from a failed placement attempt. Log the exception's top-level attributes and inspect its metadata and trailers dictionaries directly. Expect to see a generic top-level exception message, but a dictionary of trailers or metadata containing the real SIP code buried a level or two down.
2. Trigger a deliberate auth failure by changing a working trunk credential to an invalid string, then attempt a call. Expect your system to log the exact same top-level RPC error type as it does for a busy signal. This confirms your current exception handling cannot tell an auth failure from a busy callee.
3. Search your logged exception strings for three-digit numbers. Expect to find protocol codes like "486" or "404" embedded in sentences like "Call failed with SIP response 486". This proves the protocol data is present but arriving completely unstructured.

## How it works

To build a working retry policy, you have to recover the underlying protocol code first, map it to a small closed set of kinds you own, and let only that typed kind reach your retry logic. Codes are a spec. Strings are prose.

I handle this by writing a recursive function that walks the error's metadata and trailers. This walk must be depth-limited and cycle-safe, utilizing a visited set. The exception objects thrown by these RPC clients frequently hold self-references (such as an exception pointing to its own cause). A naive recursive walk will immediately hang the process.

Inside the walk, I probe the object for plausible key spellings rather than relying on one canonical name. The useful detail is not in the same place on every trunk.

If the key lookups fail to find a code, I fall back to running a regular expression over the human-readable message. This regex requires a "sip" proximity anchor. I look for the word "sip" near a three-digit number, rather than just matching any three digits in the text. Without the anchor, a three-digit area code that happens to be present in the error string will be mistakenly harvested as a SIP response code, leading to bizarre retry behavior.

Once I successfully extract the three-digit code, I map it onto a small closed set of typed kinds, and I branch my logic on the kind. The mapping handles the distinct failure domains:

- **408, 480, and 487** map to `no-answer`. I return this to the queue with instructions to retry later.
- **486 and 603** map to `busy`. I instruct the queue to retry this later, but with a significantly shorter backoff time than a no-answer.
- **404 and 410** map to `invalid-number`. This is a terminal failure. The system stops calling the number immediately.
- **5xx series codes** map to `carrier-reject`. I retry these, but I escalate if the condition is sustained across multiple calls.
- **401 and 403** map to `auth-failure`. This is terminal for the entire queue run. It represents a human problem where the trunk is broken, not the destination number. Continuing to dial under a 401 just wastes compute.

Isolating retry decisions to this small closed set ensures carrier wording changes cannot alter placement scheduling. Pre-placement markers prevent retrying tasks from double-dialing in-flight calls while classification evaluates earlier attempts. If you are struggling with retries that hit the same customer twice, you can read more about how I solve that in my post on /blog/task-queue-retry-called-the-customer-twice-and-an-exists-check-will-not-stop-it/.

## What I would check first

1. Count your unmapped errors by string across a batch of failed placements. A high volume of identical strings arriving at your fallback logic rules out random network drops. It points directly to a missing key spelling in your scraper that you need to add to the probe list.
2. Cross-reference a specific failed placement in your application logs with the logs provided by your carrier. If the carrier logged a 486 Busy but your application logged an opaque RPC error string, it rules out a carrier routing issue and confirms the RPC layer is masking the response.
3. Check whether unmapped errors contain status codes in the message text rather than the metadata. If the message contains a phrase like 'SIP 486' but metadata extraction returned nothing, your key probe list missed the trailer spelling and you need the proximity-anchored regex.

## Instrumentation and observability

I record the typed kind on every single failed placement. This allows failures to be counted by kind rather than by string. Grouping errors by `busy` or `invalid-number` gives a much clearer picture of list quality than reading through hundreds of slightly different exception sentences.

I log every failure where no SIP code could be recovered by any path. Each one is a sign the scraper needs another key spelling. A new carrier or an updated RPC library version will eventually introduce a new metadata shape, and logging the unmapped errors is how you catch the change before it impacts your retry metrics.

Finally, I record terminal classifications directly against the target phone number in the database. If a systematic misclassification occurs in the pipeline, it shows up as a dense cluster of terminal states against otherwise healthy numbers, allowing me to flag the run and investigate the carrier response.

## Common questions

**Why is the SIP code missing from the error message?**

RPC clients often encapsulate carrier responses inside generic exceptions. The specific protocol code is usually pushed down into the exception's metadata or trailer dictionaries, leaving only an opaque string at the top level.

**How do I tell if a call failed because the user was busy?**

Extract the SIP response code from the error metadata. A 486 or 603 code indicates a busy signal, which you should classify as a temporary failure and retry with a short backoff.

**Why shouldn't I just search the error string for a 404?**

Blindly searching for a three-digit number will cause false positives if the target phone number or request ID happens to contain 404. You must use a proximity anchor like the word 'sip' to ensure you are matching an actual status code.

**What should happen if I get a 401 or 403 on an outbound call?**

Treat it as an authentication failure. This is terminal for the current run because your trunk credentials are broken. Retrying the number will not fix a trunk configuration problem.

If the boundary this note describes is one you would rather own, [moving this layer off LiveKit Cloud](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/opaque-rpc-errors-hide-the-sip-response-codes-you-need-for-outbound-call-retries
