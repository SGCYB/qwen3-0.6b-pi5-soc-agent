# What I Learned Building a Small AI SOC Assistant

I started this project because I wanted to be able to step away from my SIEM without feeling completely blind to my homelab. I ended up learning far more than how to run a model on a Raspberry Pi.

## The AI Model Was Only One Part of the Work

Getting Qwen3-0.6B to run locally was exciting, but making the alerts useful took more engineering. Wazuh event structures vary by source. I had to normalize events, preserve important data like file hashes, check model output, maintain an audit trail, and build a separate path for phone notifications.

That taught me to think of AI as one component inside a system rather than the entire system.

## Small Hardware Makes You Pay Attention to Routing

I chose the Raspberry Pi 5 specifically because I wanted to see how much I could do with affordable hardware. The constraint forced me to make decisions about which events deserve AI analysis. Processing every log was never realistic, and blindly notifying on every change would create noise and cause alert fatigue.

I initially used lower-severity canary rules to verify the workflow. After validating the integration, I moved to a minimum Wazuh level of 13. I later added and offline-validated a narrowly scoped SSH correlation rule (`100171`). During controlled lab activity, it generated two genuine Wazuh level-13 alerts that successfully passed through the durable queue, local Qwen inference, and ntfy delivery to my iPhone.

## Threat Intelligence Is Helpful, but Context Still Matters

AbuseIPDB and OTX can provide additional evidence, but they measure different things. An IP reputation score is not the same as a community pulse count for a file hash. Likewise, a lack of reports does not mean an indicator is harmless.

I wanted the phone alert to show the underlying facts and keep the judgment where it belongs: with me. I also learned to take MITRE ATT&CK IDs from Wazuh rule metadata, not AI guesses, and to preserve the original event timestamp and structured source/destination IP addresses for follow-up in the SIEM.

## I Learned Not to Promote a Model Just Because Training Finished

I ran a small CPU-only LoRA experiment and compared its behavior with the original model. The run finished, but the evaluation was limited and did not show an improvement. Some synthetic examples also exposed overconfident classifications under uncertainty.

I kept the production inference model unchanged. The better short-term result came from clearer, NIST-aligned investigation instructions and a controlled acceptance test, not retraining. For this project, being willing to reject an unproven improvement was more valuable than claiming I had trained a better security model.

## Reliability Isn't the Same as One Successful Notification

I built persistent audit logging, publisher checkpoints, deduplication, retry behavior and a recovery policy. I then checked that the publisher was caught up with the audit stream and that the exact canary indicator had a recorded OTX lookup.

Those checks gave me confidence in the working flow.  

## A Short Burst Needs Storage More Than More Model Threads

Once I had the alert-to-phone path working, the next practical concern was multiple Wazuh alerts arriving while Qwen was already busy. The solution I tested was a durable single-worker queue, not another parallel model process. I also learned that retries need careful definitions. Acknowledging a queued alert is different from verifying an audited inference, and an uncertain timeout should not automatically trigger a duplicate phone alert. [What I tested →](durable-queue.md).

## My Biggest Takeaway

AI doesn't need to replace the analyst to make security operations easier. If I can get a timely summary of an event, understand what evidence supports it, and decide where to investigate next, that's already a ton of useful information that can support my investigation.  
