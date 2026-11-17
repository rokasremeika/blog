> Originally published at https://www.dreamchasers.lt/blog/the-scorer-is-a-program-too-and-it-is-wrong-more-often-than-the-agent
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 8 October 2026.

When a voice agent evaluation pipeline scores a flawless transcript as unclassifiable, or flags an agreement as a refusal, the scoring logic itself is introducing noise. Deterministic regex matches are brittle against spontaneous speech. Callers elide expected words, causing false negatives, or use negated constructions like "I would not say no" that falsely trigger decline signatures. The fix is a two-tier scoring system. Tier one uses strict deterministic matching for predictable phrasing, defaulting to an explicit unresolved state instead of failure. Tier two escalates only the unresolved transcripts to an independent model for adjudication.

## Symptoms

- An evaluation run that a human reads as a flawless conversation comes back unclassifiable.
- A paraphrase that means the opposite of a refusal gets filed as a refusal by the pipeline.
- The pass rate numbers move between sweeps because the scoring code changed, not because the agent prompt or model did.
- A perfect run scores as unmatched because a signature expected a word the speaker elided in spontaneous speech.

## What is actually happening

The scorer is a software program too, and it is wrong more often than the voice agent it evaluates. Deterministic matching over transcripts is cheap and reproducible, but it is brittle in exactly the way spontaneous speech is unpredictable.

A typical home-grown scorer relies on standard regular expressions to parse the agent's output transcription. A signature written from the prompt's phrasing expects a specific word the speaker elided in conversation. Spontaneous speech regularly omits connecting words that deterministic substring rules rely on to detect a match. When the elided word is missing from the transcript, deterministic matching fails to trigger and leaves a successful run unmatched. The aggregate metrics move because the measurement instrument failed to match the transcript, not because the agent failed the scenario.

Conversely, consider a decline signature. You want to know if the user refused to answer a qualification question. The signature looks for "no", "not", or "refuse". The user says, "I would not say no to that." The regex engine hits the "not" and the "no", and the run is permanently marked as a refusal. The caller was actually agreeing, but because negated constructions contain the exact stems a decline signature expects, the agreement is punished.

Neither error is visible in the aggregate pass rate. Both errors quietly move the number you are about to make a deployment decision on. This quietly corrupts the evaluation signal before any deployment decision is made.

## How to confirm it

1. Read ten transcripts from your last sweep that scored as unclassifiable. Expect to find at least one where the caller clearly agreed or declined but used a synonym or elided the verb your regex expected.
2. Search your false-negative bucket for negated phrasing. Look for the words "not", "never", or "wouldn't" sitting immediately next to your target decline stems. Expect to see successful agreements falsely flagged as rejections.
3. Run your scoring script against a static batch of agent transcripts, using two different recent commits of your scoring code. Expect the total pass rate to shift. A shift confirms that your scoring instrument is injecting noise independently of agent behavior.

## The fix

I fixed this by splitting the evaluation pipeline into a two-tier scoring system. The architectural pattern was straightforward: cheap-and-exact first, expensive-and-judgmental second, with an explicit unresolved state sitting between them.

Tier one was deterministic substring and stem matching. It resolved runs at zero marginal cost wherever the phrasing was highly predictable. I maintained this deterministic matching in the two languages the agent was validated in end to end (the agent itself is configurable across 60 languages, live-validated in two, and scoring signatures existed for those two).

Crucially, tier one was not allowed to guess. If a transcript did not match the expected signatures perfectly, it did not default to a failure, and it did not default to a pass. Anything it could not confidently resolve was emitted as an explicit `unresolved` state and passed to tier two.

Tier two was an independent adjudication pass executed by a separate large language model. This model judged the raw transcript alone, without seeing the agent's internal state. It was invoked only for the runs that tier one left unresolved.

To keep tier one honest and prevent the regex rules from becoming an unmaintainable mess, I enforced two discipline rules. First, signatures were loosened only against speech that was actually observed in the real world. When a run exposed a gap in the matching logic, that exact run was recorded in the codebase directly next to the new signature. This made the loosening traceable rather than folkloric. Second, every single decline signature carried its own negation. I required that a phrase meaning the opposite could not possibly be matched by the same stems.

## What I would check first

1. Check the default verdict of your scoring function. Inspect whether unclassified runs reflect callers eliding expected prompt phrasing or the agent failing to elicit the turn. Expect an unresolved state. A default failure rules out accurate visibility into missing signatures, meaning your pass rate is artificially low.
2. Measure the ratio of deterministically resolved runs versus escalated runs in a single sweep. Expect the vast majority to resolve in tier one. If the adjudicator is processing most of the sweep, this rules out a healthy deterministic tier and confirms your signatures are too narrow.
3. Check your scorer test suite for negation coverage. Run a test where a mock user says "I am not refusing". Expect the scorer to explicitly bypass the refusal signature. A false positive here rules out the agent being at fault for declining agreement metrics.
4. Check the audit trail of your signature modifications. Look at the last regex you loosened. Expect to see a link to the specific transcript that forced the change. A missing trail rules out an evidence-based evaluation process, meaning developers are likely introducing false positives by guessing.

## Why the scorer test suite is roughly two-thirds the size of the scorer

If the scorer is broken, every other metric is compromised. To guarantee its accuracy, I built a dedicated regression test suite for the scoring code itself.

My scorer test suite ended up roughly two-thirds the size of the entire scorer codebase. Every time a new edge case was discovered in a real conversation, a mock transcript representing that exact phrasing was added to the test matrix.

This test coverage was the only way to ensure that adding a new negation check did not accidentally break an existing agreement signature.

## Instrumenting the adjudication boundary

When you delegate decisions to a second model, you have to watch the boundary between the deterministic code and the LLM. I instrumented the pipeline to count the runs resolved deterministically versus those escalated to adjudication, and I logged this ratio per sweep.

Every escalated run was archived alongside the exact transcript the adjudicator saw. This was critical for debugging. When a sweep produced a surprising number, I needed to know if the agent changed its behavior or if the adjudicator hallucinated a verdict. Archiving the prompt and transcript together meant I could replay the exact adjudication step in isolation.

This architecture generalizes to any transcript-scored evaluation, voice or text. It gives you reproducibility where the language is predictable and coverage where it is not, and it keeps the model-judged fraction small enough to review by hand. Behavioral edits to a voice agent trade failure classes against each other, a dynamic I detailed when writing about how a [prompt change improved one behavior but broke another](/blog/voice-agent-prompt-change-improved-one-behavior-but-broke-another/). If the scorer randomly misclassifies those trade-offs, you will inevitably revert good changes and ship regressions.

## Common questions

**Why do flawless agent runs score as unclassifiable?**

Callers routinely elide the specific verbs or nouns your deterministic signatures expect. When a required word is missing, the regex fails to match, leaving a successful run unclassified.

**Why does the scorer flag user agreements as refusals?**

Users often speak in negated constructions, such as "I would not refuse". The transcript contains the exact stems a decline signature looks for, causing a false positive.

**How should a transcript scorer handle unrecognized phrasing?**

It must default to an explicit unresolved state. Defaulting to a pass or a fail masks the fact that your signatures are missing common phrasing.

**What is a two-tier evaluation system?**

It applies cheap, exact deterministic matching first. Any transcript it cannot resolve is escalated to a second tier, where an independent LLM adjudicates the conversation.

Canonical URL: https://www.dreamchasers.lt/blog/the-scorer-is-a-program-too-and-it-is-wrong-more-often-than-the-agent
