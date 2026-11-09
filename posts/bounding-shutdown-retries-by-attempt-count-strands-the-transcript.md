> Originally published at https://www.dreamchasers.lt/blog/bounding-shutdown-retries-by-attempt-count-strands-the-transcript
> by Rokas Remeika, Realtime Voice & Telephony Field Notes.

By Rokas Remeika. Published 24 October 2026.

When a voice agent worker shuts down, the framework gives it a fixed time window before killing the process. If your transcript persistence logic retries based on a maximum attempt count, the combined HTTP timeouts can exceed the shutdown window. The process dies mid-write, losing data that only existed in memory. The fix is to budget retries by wall clock. Give the operation a total time budget, and size it so the budget plus one maximum-length HTTP timeout remains strictly inside the framework's shutdown window.

## Symptoms

- Transcripts and end-of-call records go missing for calls that ended normally.
- The write operation is attempted, but the logs stop mid-attempt with nothing after them.
- The missing records look like flakiness, happening randomly across healthy instances.
- The failure gets significantly worse whenever the downstream storage service is slow, which is exactly when the end-of-call record matters most for debugging.

## What is actually happening

When a worker process finishes a call or receives a termination signal, the framework running it initiates a teardown phase. The framework gives your shutdown callback a fixed window to clean up and save state. After that window expires, the framework or the host operating system kills the process entirely.

Most persistence paths use standard retry libraries or custom loops that bound retries by attempt count. A network write might be configured to retry three times, with a ten-second HTTP timeout per attempt. This works perfectly during normal operation because the process will live indefinitely to see the attempts conclude.

During teardown, the process is living on a fixed timer. A persistence path that bounds its retries by attempt count has a worst case measured in attempts, not seconds. Three attempts at a ten-second HTTP timeout can block for thirty seconds. If your framework gives you a twenty-two second graceful shutdown window, you have an eight-second deficit.

The downstream service experiences a latency spike. Your first write attempt takes ten seconds and times out. Your second attempt takes ten seconds and times out. You start the third attempt. Two seconds later, the framework's shutdown timer expires. The process receives a SIGKILL while the write is in flight. The memory containing the final transcript is wiped, and data that existed only in that process is gone forever.

The bound on your retry logic is real, but it is denominated in the wrong unit. Attempt counts bound the number of failures, not the time. Shutdown windows bound the time, not the failures.

## How to confirm it

1. Filter your logs for the worker process ID that handled the lost call, starting from the shutdown signal. Expect to see the persistence attempt begin, followed by a hard stop with no subsequent log lines indicating success or failure.
2. Measure the timestamp gap between the shutdown signal and the final log line of the process. Expect this gap to match your framework's exact graceful shutdown timeout limit, proving the process was forcefully killed.
3. Check the downstream storage service logs for the exact minute the call ended. Expect elevated latency or HTTP gateway timeouts that pushed the write duration past its normal baseline, triggering your retry loop.
4. Compare the recorded wall-clock duration of the shutdown callback directly against the framework window in your call event logs. Expect the duration to reach the framework shutdown window limit without any subsequent log indicating completion.

## How it works

The fix is to budget the teardown persistence by wall clock instead of attempt count. You give the whole operation a total time budget, give each attempt its own HTTP timeout, and size them so that the budget plus one maximum-duration in-flight attempt still lands safely inside the shutdown window.

Here is the sizing I run in production. The framework provides a 22-second shutdown await window. I give the end-of-call record write an 8-second total wall-clock budget. I give the HTTP client a 10-second per-attempt timeout.

If the storage backend is struggling, the loop checks the wall clock. As long as the elapsed time is under 8 seconds, it fires another HTTP request. The worst-case scenario is an attempt starting right before the 8-second budget expires. That final attempt is allowed to block for its full 10-second timeout. The total time elapsed before the HTTP client times out and returns control to the loop is just under 18 seconds.

That 18-second worst case fits cleanly against the 22-second shutdown await the framework gives me. The process has time to observe the final failure, log it cleanly, and exit on its own terms before the operating system brings the hammer down.

The three numbers in that example are not the transferable part. The inequality is the transferable part: `(total_budget + per_attempt_timeout) < framework_shutdown_window`.

You must write that exact arithmetic in a comment next to the constants in your code. The constraint is completely invisible from any one constant on its own. The next person to review the code will see an 8-second budget, decide it is too tight for a slow backend, raise it, and unknowingly break the system. They will not know what they broke because the increased budget still looks smaller than the 22-second shutdown window on its own. They will only find out when transcripts start vanishing again.

## What I would check first

1. Check if the session was ended by a function tool calling a teardown routine rather than a normal caller hangup. A tool tearing down the session immediately can cut off final audio and disrupt the lifecycle sequence; if this is the case, arming a pending closure fixes it, as detailed in /blog/voice-agent-end-call-tool-truncates-the-final-sentence-or-loses-the-callers-last/.
2. Check the HTTP response codes returned by the downstream API just before the data loss. An HTTP 400 Bad Request indicates a malformed payload that failed instantly, ruling out timeout starvation. If the downstream storage service returned an HTTP gateway error, the request completed over the network rather than being terminated mid-flight.
3. Check whether the worker process suffered an unexpected crash or host termination prior to the shutdown callback running. Check process supervisor logs to verify whether the process ran until the framework graceful shutdown timer expired.

## Instrumenting the teardown path

Fixing the timeouts stops the data loss, but you need instrumentation to prove the fix holds over time. A teardown sequence is invisible to standard request tracing because the client has already disconnected.

I record the shutdown callback wall-clock duration per call and emit it as a structured log event. This allows me to query the duration and compare it against the framework's window across the entire fleet. If the persistence operation starts creeping closer to the 18-second worst case, I can see the degradation before it starts dropping records.

Additionally, give an incomplete transcript write its own named failure code on the call's end event. Do not let it land as a generic internal error. When a failure code specifically names the persistence timeout on the end event, you isolate a wall-clock write failure immediately instead of investigating it as a generic error.

## Generalizing the pattern

This failure mode generalises to every teardown path that performs a network write. Whether you are flushing metrics, saving an LLM context window, or writing billing records, the dynamics are identical.

Anything you write at teardown that exists nowhere else in the system needs a wall-clock budget derived directly from the window you were given.

## Common questions

**Why do my agent transcripts randomly go missing on call end?**

Your persistence logic is likely taking longer than the graceful shutdown window provided by the framework. When the retry loop exceeds this timer, the process is forcefully killed, and the in-memory transcript is destroyed before it can be saved.

**How long do I have in a shutdown hook before SIGKILL?**

It depends entirely on your framework or container orchestrator. You must check the specific graceful shutdown timeout configured in your environment.

**Should I use a background queue for transcript persistence?**

Offloading persistence to a separate queue still requires a network write before the worker process is killed. However, you still have to successfully write the payload to the queue during the shutdown window, so the wall-clock constraint applies to the queue publisher as well.

**Why does setting my HTTP retry count lower not fix the issue?**

A low attempt count still relies on the per-attempt HTTP timeout. If a single HTTP timeout can exceed the framework shutdown window, lowering the attempt count still leaves the write vulnerable to being killed mid-flight.

Canonical URL: https://www.dreamchasers.lt/blog/bounding-shutdown-retries-by-attempt-count-strands-the-transcript
