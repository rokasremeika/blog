> Originally published at https://www.dreamchasers.lt/blog/deterministic-inbound-sip-routing-when-tenants-share-a-phone-number
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 2 October 2026.

When multiple tenants share an inbound phone number, the caller's phone number is the only routing key available. If a caller exists in multiple accounts, a database query that takes the first match will route the call non-deterministically based on index order. The fix is to gather every matching record across all tenants assigned to the dialed number, then sort them using an explicit four-level tie-break chain ending in the unique record ID. Writing a routing decision log for every outcome makes these transient decisions auditable days later.

## Symptoms

- A caller dials your shared inbound number and reaches the wrong tenant's voice agent.
- The routing changes over time: the same person calls from the same device an hour later and reaches a completely different record.
- A caller is rejected and disconnected, but you verify later that a valid record for them exists in the datastore.
- When a complaint arrives days later, there is no way to reconstruct which records existed at the time or why one of them was chosen. The raw E.164 numbers are usually correct, but the application is guessing what to do with them.

## What is actually happening

When a call arrives from the carrier over a raw SIP trunk, the INVITE contains two critical pieces of routing information. The `To` header tells you which number was dialed (the shared DID). The `From` header contains the caller's phone number. On a multi-tenant platform where several tenants share a single inbound number, the caller's number is the only routing key you have.

The failure occurs because caller numbers are not globally unique to a single record. The same person might exist as a customer under two different tenants assigned to that shared number. They might even have multiple records within a single tenant (for example, an active profile and an archived one).

If your routing logic executes a database query that stops at the first match, you are delegating the routing decision to the database storage engine. A query without a strict total order will return records based on index traversal order, page boundaries, or insertion history. This behaves non-deterministically. The database might return tenant A today and tenant B tomorrow, simply because a background process rewrote a page on disk. The symptom is invisible in standard application logs because the application correctly routes the call to the record it was handed.

Because the interesting information is the set of matches at that exact instant, the context is gone by the time anyone asks. When a complaint arrives days later, there is no way to reconstruct which records existed at the time or why one of them was chosen.

## How to confirm it

1. Query your datastore for the caller's number across all tenants sharing the dialed DID. Expect to find more than one active record; this confirms a routing collision is mathematically possible.
2. Review the routing logic handling the inbound INVITE. Expect to see a simple query limit or a first-match ORM fetch rather than an exhaustive gather-and-sort operation.
3. Ask the database for the same caller number multiple times from an interactive shell while simulating background writes to those records. Expect the returned record to occasionally swap, proving the query ordering is unstable.

## How it works

The solution requires taking the routing decision out of the datastore's hands and enforcing a strict mathematical total order in application memory.

First, gather every matching record for the caller across every tenant assigned to the dialed number. You must retrieve the entire set of potential matches.

Once all matching records are in memory, rank them with an explicit, documented tie-break chain rather than letting the datastore decide. I settled on a four-level deterministic ranking chain:

1. **Status first.** An active or expected record always outranks a stale one.
2. **The expected-call timestamp.** A record with an expected-call timestamp outranks records without one.
3. **Creation time.** Creation time acts as the third tie-breaker before the final unique record ID.
4. **Record ID.** This is the final total order. By ending with a globally unique identifier, the ranking guarantees a deterministic outcome even when everything else ties.

After sorting, the system routes the call to the top record. However, routing is only half the fix. You must write a routing decision record for every outcome: matched, ambiguous, and rejected.

An ambiguous outcome means that more than one plausible record survived the initial status ranking, forcing the system to rely on timestamps or IDs to break the tie. The caller still connects, but the ambiguity is permanently logged. A rejected outcome means nothing matched, or the caller is not permitted.

Because inbound complaints are always retrospective, you need these records to reconstruct any single call afterwards. Caller numbers must be hashed before they reach any log to protect privacy.

## What I would check first

1. Find the inbound INVITE in your SIP trace and extract the exact `From` header. Expect the number to match records across multiple tenants sharing the dialed DID, confirming that the caller ID is not a unique routing key.
2. Search your application logs for that exact caller number hash at the time of the call. Expect to see whether the application found zero records or multiple records. Finding zero records indicates the caller had no matching record or was not permitted, whereas multiple records confirm a routing collision.
3. Check the database indexing for the table holding the caller records. Expect a composite index on the shared DID and the caller number. If the query fetches a single record without an explicit order clause, the datastore returns records based on storage index traversal and page boundaries.

## Treating rejections as system defects

When a call is rejected because the system finds no matching record, the caller is disconnected. In a multi-tenant environment, a rejection represents a dropped interaction.

I review rejections that turn out to have had a valid record as system defects. If a tenant reports that a known customer called in and was rejected, the routing decision log is the only way to prove what happened. The decision record provides the audit trail needed to prove whether an active record existed when the caller was rejected.

## Generalizing across shared identifiers

Any multi-tenant inbound channel keyed by a shared identifier has this problem. Whether you are operating a shared phone number, a shared email inbox, or a shared SMS short code, the mechanism remains identical.

Two rules carry over directly. First, make the ranking a total order, ending in something globally unique, so the answer never depends on storage internals. Second, log the decision, not just the result. You cannot re-run yesterday's set of matches, so the system must freeze its exact reasoning at the moment the event arrived.

## Splitting numbers based on ambiguity

Shared numbers work perfectly until the user bases overlap too heavily. The failure mode of a shared number is ambiguity.

Because I record the ambiguous outcome explicitly on the decision record, I can aggregate these over time. Repeated ambiguity on one number is the leading indicator that a shared DID needs splitting. When a specific tenant begins generating constant collisions, you have concrete data justifying the migration to a dedicated number.

## Common questions

**Why does an inbound call route to the wrong tenant?**

When tenants share a phone number, the caller's number is the only routing key. If the caller exists in multiple tenants, a basic database query returns the first match it finds, which depends on unpredictable disk indexing.

**How do I make inbound call routing deterministic?**

Fetch all matching records into application memory and apply a strict sorting chain. End the sort with a globally unique record ID so the answer never depends on storage internals.

**Why log routing decisions if I already log errors?**

Inbound complaints arrive days after the call. The datastore state will have changed, so you cannot re-run the query to see what matches existed at the time. A routing decision record freezes that instant.

If the boundary this note describes is one you would rather own, [a self-hosted deployment you control](/migration) moves it inside your own infrastructure.

Canonical URL: https://www.dreamchasers.lt/blog/deterministic-inbound-sip-routing-when-tenants-share-a-phone-number
