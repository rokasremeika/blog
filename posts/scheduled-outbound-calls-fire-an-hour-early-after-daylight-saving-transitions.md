> Originally published at https://www.dreamchasers.lt/blog/scheduled-outbound-calls-fire-an-hour-early-after-daylight-saving-transitions
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 22 October 2026.

Scheduling outbound calls using naive local time arithmetic breaks twice a year. A spring-forward gap creates a wall-clock time that does not exist, causing date libraries to throw, while a fall-back overlap creates a time that exists twice, causing unpredictable dialing an hour early. To fix this, ask your date library if a constructed local time is valid to detect gaps, and snap to the next existing instant. For overlaps, consistently select the earlier instant. Finally, validate that the projected call end falls within the calling window using the estimated conversation duration.

## Symptoms

- An outbound call goes out at 07:00 local time instead of the expected 08:00 on the Monday following a clock change.
- A scheduled dial executes legally at 19:50, but the conversation is still running at 20:20, pushing past the strict 20:00 cutoff the customer configured.
- A scheduling job throws a runtime exception on a specific date because the underlying date library asserts the constructed local time does not exist.

## What is actually happening

These present as three separate errors that all look like timezone bugs.

The first error stems from deriving a timezone exclusively from a phone number's country code. This approach works for most countries, but it is simply wrong for countries that span several zones. Guessing the local time for a contact in a country spanning several zones based purely on the country code is a coin flip. Eventually, your scheduler will guess incorrectly and place a call while the recipient is eating breakfast.

The second error comes from checking window membership at the start instant only. A calling window constrains the entire conversation rather than the moment of dialing. When you evaluate a scheduled task by comparing the dial instant to the cutoff time, a half-hour call started ten minutes before the cutoff violates the constraint for twenty minutes. The system believes it acted legally because the dispatch cleared the gate, but the recipient experiences an out-of-hours intrusion.

The third error is raw arithmetic failing across a daylight-saving transition. When calculating future schedules, a spring-forward gap creates a wall-clock time that does not physically exist. A naive local-time construction will immediately throw an exception. Conversely, a fall-back overlap creates a time that exists twice in the same day. Without explicit instructions, date libraries silently pick one of the two instants, usually placing them exactly an hour apart from what you intended. The library makes an arbitrary decision, and your scheduler executes on it.

## How to confirm it

1. Search your application exceptions for invalid date or time errors clustered specifically on the exact day of a spring-forward transition. Expect to see date library validation errors during the spring-forward gap where the target wall-clock time does not exist. This confirms your code is constructing a non-existent wall-clock time and the library is throwing a validation error.
2. Query your call logs for the conversation end time against the allowed window cutoff for any complaints about late calls. Expect the start time to be legally inside the window, but the duration pushing the end time past the limit. This confirms the scheduler only validates the dial instant and ignores the call duration.
3. Check the timezone assignment logic for multi-timezone regions in your database. Look at a random sample of numbers from multi-zone countries. Expect to find a default offset being blindly applied. This confirms derivations are being made where explicit user selection is required, proving the system is guessing rather than knowing.

## How it works

I built my scheduling layer to treat local time complexity as a strict boundary. The first rule I enforce is to refuse to guess. The scheduler derives the country from the parsed phone number and maps that country to an IANA zone only where that mapping is unambiguous. For countries that span multiple zones, I require an explicit selection rather than defaulting. The customer configuring the campaign knows which zone their contact lives in, and I do not. If the explicit selection is missing, the schedule is rejected.

To enforce the calling window, I require the projected call end to fit entirely inside the allowed window rather than checking only the dial start time. The system calculates the projected end by adding the estimated call duration to the target dispatch time. The scheduler already carries this duration estimate for capacity accounting. When deciding admission inside one transaction, as detailed in /blog/enforcing-maximum-concurrent-sip-calls-without-race-conditions/, the system allocates a slot based on how long the call will occupy it. I feed that same duration metric into the window evaluation. This projection relies on the estimated call duration that the scheduler already tracks for capacity accounting. If the total expected duration pushes the end of the call past the cutoff, the dispatch is declined.

For daylight-saving transitions, I handle both directions explicitly rather than trusting library defaults. In a spring-forward gap, I detect the missing time by asking the date library whether the constructed local time is valid at all. I use this validity check instead of hardcoding transition dates. When it returns invalid, the system snaps to the next instant that actually exists. In a fall-back ambiguity where the time exists twice, I configure the library to choose the earlier of the two instants consistently. This makes the behavior a documented rule rather than whichever branch the library happens to prefer on a given language version.

Finally, I instrument every scheduling decision. The scheduler logs the resolved IANA zone and records a boolean flag indicating how it was resolved: explicitly selected or safely derived. I maintain a strict count of gap snaps and ambiguity resolutions, which is the only way you find out your DST handling ran correctly during a transition weekend. I also track a distinct count of dispatches declined specifically because the projected end fell outside the window, separating these rejections from capacity-based delays.

## What I would check first

1. Review how your date library handles ambiguous local times during fall-back overlaps. Expect it to default to one of the two instants silently without warning. Check whether early dispatches happen only during transition hours or persist on standard days to distinguish overlap ambiguity from invalid timezone offsets.
2. Inspect the mathematical validation logic for your calling windows. Expect the code to evaluate only the current timestamp against the cutoff limit. If the check lacks an addition of the projected call duration, this rules out complex timezone math errors and confirms a logic gap in your boundary enforcement.
3. Look at your timezone mapping dictionary or function for parsed phone numbers. Expect single-zone countries to map cleanly. Correlate out-of-hours complaints with recipient country codes to separate multi-zone derivation errors from daylight saving arithmetic bugs.

## Making daylight saving time an explicit decision

This architecture generalises to any scheduler that acts on someone else's local time. Whether you manage outbound calls, SMS campaigns, appointment reminders, or delivery windows, the local time belongs to the recipient.

Three rules carry across all these domains. First, never infer a timezone you can ask for. A mapping function is a convenience for single-zone regions, not a replacement for domain knowledge. Second, constrain the end of the activity rather than its start. The user constraint always applies to the entire duration of the intrusion. Third, make both DST directions explicit decisions with a written rule.

Every mature date library exposes a validity check that doubles as a gap detector. You must use it rather than building a transition calendar. Hardcoding transition dates guarantees your system will fail the moment a local government changes their DST observance policy, which happens frequently. By asking the library if the constructed time is valid, you offload the political complexity to the underlying tz database updates, while your application code safely snaps to the next available instant.

## Common questions

**Why did a scheduled call dial an hour early on Monday?**

Your scheduler performed local-time arithmetic across a fall-back daylight saving transition. Naive local-time arithmetic across the clock change calculated an instant an hour off from the intended local hour.

**How do I handle a scheduled time that does not exist during spring forward?**

Ask your date library whether the constructed local time is valid instead of hardcoding transition dates. When it returns invalid, snap the scheduled time to the next instant that actually exists.

**Should I derive the timezone from the phone number?**

Only for countries that do not span multiple timezones, mapping the country to an unambiguous IANA zone. For countries spanning several zones, require an explicit selection instead of defaulting.

**Why did my call run past the calling window cutoff?**

Your scheduler checked the window membership using only the dialing start instant. A window constrains the conversation, so you must validate the projected call end using the estimated duration.

Canonical URL: https://www.dreamchasers.lt/blog/scheduled-outbound-calls-fire-an-hour-early-after-daylight-saving-transitions
