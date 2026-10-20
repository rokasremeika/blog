> Originally published at https://www.dreamchasers.lt/blog/voice-agent-prompt-change-improved-one-behavior-but-broke-another
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 4 October 2026.

A system prompt change that improves one voice agent behavior will almost always trade away attention from another. You only see this trade if you measure the untouched behavior in the same run, against a threshold written before you see the data. To stop regressions from reaching production, register a falsifier for every hypothesis before you run it: a specific metric that kills the change if it drops, regardless of how good the headline number looks. In my own environment, this pre-registered falsification forced me to revert three of my own prompt fixes before they shipped.

## Symptoms

- A system prompt change measurably improves the specific behavior you were targeting.
- The change ships to production because the headline metrics look positive.
- Something else the agent used to do reliably quietly stops working.
- Nobody notices the regression until a live call goes badly in a way that has nothing to do with the original change.
- The misleading observation: local testing of the targeted scenario shows a clear win, tricking you into believing the prompt change was an isolated fix with no side effects.

## What is actually happening

Behavioral edits to a voice agent trade failure classes against each other. When you add instruction text to tighten one specific behavior, that new text spends the model's attention. That attention is finite, and it was previously holding up another behavior elsewhere in the conversation.

The model has no way to tell you which behavior it dropped to accommodate your new rule. You only see the trade if the behavior you did not touch is measured in the exact same run. That measurement must be evaluated against a threshold written before the run even started, on a sample size large enough that a real move is distinguishable from statistical noise.

If you only score the agent on the problem you are trying to fix, every prompt expansion looks like a success. The regression hides in the unmeasured branches of the conversation state machine.

## How to confirm it

1. Run the new prompt and the old prompt against a baseline regression suite of at least a hundred recorded calls using a synthetic caller. Expect the target metric to improve; expect at least one unrelated metric to shift.
2. Compare the failure rates of mandatory agent behaviors (like a required decline or a strict data-collection step) between the old and new prompt. Expect the failure rate on the unrelated mandatory step to increase.
3. Score the new prompt against an adversarial arm targeting behaviors adjacent to the change, not just the change itself. Expect the adversarial arm to expose edge cases the new prompt instruction inadvertently opened up.
4. Check the timing of your threshold decisions. Ask the team when the passing score for the run was decided. Expect to find that the threshold was agreed upon after the data landed, which confirms the evaluation was contaminated by the desire to ship the fix.

## The fix

To stop these silent regressions, I stopped treating prompt changes like isolated bug fixes. For every hypothesis, I registered a falsifier first. This was a specific measurement that killed the change if it moved the wrong way, regardless of how good the headline number looked.

Then, I ran a matrix large enough to detect the regression. Here, that was 123 scripted conversations driven by the synthetic caller. The synthetic caller joins as a real RTC peer and speaks real audio, with no mocks in the media path. The synthetic caller joins as a real RTC peer to drive scripted dialogue against the agent without mocks in the audio pipeline. I repeated each of the 123 conversations three times against a pristine control, scored by the same evaluator with full denominators.

Alongside this main matrix, I added a 48-run adversarial arm aimed specifically at the behaviors most likely to be collateral damage from the prompt edit.

Following this strict methodology, three separate prompt-level fixes were designed, built, measured, and reverted by their own pre-registered falsifiers. A fourth was built and also falsified.

The most instructive hypothesis improved the behavior it targeted from 20 bad cases in 123 down to 16. The headline metric was an obvious win. However, it completely destroyed the agent's ability to decline an invalid request. A scenario where the agent is strictly required to refuse fell from 6 out of 6 successful refusals to just 2 out of 6.

Because I had registered the falsifier beforehand, the revert was forced by that stage's own logic. It was not argued afterwards in a meeting. The change died in testing, and the production agent was saved from a catastrophic behavioral regression.

## What I would check first

1. Check the sample size of the evaluation run. If the run relies on an unrepeated sample rather than a repeated evaluation matrix, rule out the run as statistical noise. You need a repeated matrix of 123 conversations across three repeats to reliably detect regressions.
2. Check the scope of the scenarios tested. If the evaluation only included the scenario the prompt tweak was meant to fix, rule out the test as a complete measure of agent health. The regression is hiding in the paths you did not test.
3. Check whether the apparent regression stems from transcript scoring brittleness rather than agent behavior. If the team picked the passing threshold after looking at the numbers, rule out the evaluation as objective. Verify the failure against raw transcripts before concluding the prompt logic regressed.
4. Check the denominator of the scoring logic. If the test only measures the count of successful passes without measuring the total count of valid attempts, rule out the metric as a reliable indicator. You need full denominators to know if a behavior actually improved or if the agent just avoided the scenario.

## The illusion of isolated prompt edits

In a traditional codebase, a localized patch to a pure function has isolated effects. You write a unit test for the function, verify the output, and ship the patch.

A system prompt is not a pure function. It is a shared, globally weighted context window. Every word you add shifts the relative importance of every other word.

This is why traditional unit testing methodologies do not map to voice agents. The entire persona, including its constraints and refusal mechanisms, is recomputed on every single audio turn. Testing one branch requires re-verifying the structural integrity of every other branch.

## Structuring the evaluation matrix

Catching these cross-pollinating failures requires a specific testing topology. A handful of transcribed text chats will not expose the regression. You need an evaluation matrix built for scale and objectivity.

In my environment, this meant building a synthetic caller that actually negotiates WebRTC media and speaks real audio into the agent's microphone track. The agent does not know it is talking to a test harness. The matrix executes 123 synthetic-caller conversations. Because LLM behavior is non-deterministic even at low temperatures, a single pass is insufficient. Every call in the matrix is repeated three times to establish a reliable failure rate rather than a single anecdotal pass or fail.

The critical component is the pristine control. Every new prompt is scored simultaneously against the unmodified production prompt. The scorer evaluates both arms using the same deterministic criteria. If the treatment arm improves the targeted behavior but the control arm retains a higher success rate on mandatory refusals, the falsifier triggers.

## The psychology of the pre-registered revert

If you wait until after the evaluation run finishes to decide what an acceptable regression looks like, human nature will compromise the agent.

The pre-registered falsifier removes this negotiation entirely. By writing down the exact metric that kills the change before the run starts, you bind the team to the data. It shifts the discussion from "is this regression acceptable" to "the prompt failed the contract, try another approach". It is the strongest available answer to a client asking how they know your changes will not break their live agent: because three of mine did, and the falsifiers killed them before they shipped.

## Common questions

**How do I know if a prompt change broke unrelated behaviors?**

You run the new prompt against a regression suite of synthetic calls that test behaviors you did not touch. If the failure rate on those unrelated paths increases, the change caused a silent regression.

**What is a pre-registered falsifier in prompt testing?**

It is a specific metric and threshold written down before you evaluate a prompt change. If the run fails to meet this threshold, the change is automatically discarded regardless of other improvements.

**Why did my voice agent stop declining invalid requests?**

Adding new instructions to the prompt diluted the weight of the instruction telling the agent to decline. The model traded away the refusal behavior to spend attention on your new rule.

**Why is testing the targeted scenario not enough?**

Because system prompts are not isolated functions. Changing one rule shifts the weight of all other rules, meaning regressions will hide in the conversation paths you chose not to measure.

Canonical URL: https://www.dreamchasers.lt/blog/voice-agent-prompt-change-improved-one-behavior-but-broke-another
