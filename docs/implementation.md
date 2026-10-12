# Implementation: The Parts I Built

The goal was to connect an existing SIEM with a small local AI model and turn selected detections into useful notifications. Most of the engineering work was in the surrounding integrations, not the number of model parameters.

## 1. Getting Usable Alerts Out of Wazuh

I added a custom integration that forwards selected high-severity alerts through a restricted SSH tunnel to a durable SQLite queue on the Raspberry Pi. A single queue worker then submits events to the existing local Qwen gateway.

## 2. A Durable Queue Before Qwen 

As I got the original SOC pipeline working, I also had to think about what happens if several high severity alerts arrive before the Pi has finished one inference. I added a small standard-library Python service with SQLite WAL storage and one worker, rather than a parallel inference pool.

The Wazuh side uses a restricted SSH tunnel and a token-authenticated local queue endpoint. The service runs with a separate system account and stores pending events in private state. Only after a job is accepted does Qwen process it through the existing gateway. Duplicate Wazuh IDs are recognized, and uncertain delivery results are held for review rather than blindly replayed.

I preserved the previous integration script and Wazuh configuration so I can restore the original direct gateway path if needed. The control design and specific evidence are in [Durable queue](durable-queue.md), with [measured benchmark results](real-world-benchmarks.md).

## 3. Building a Local AI Gateway

I used Docker to run Qwen3-0.6B locally on the Raspberry Pi 5. The gateway accepts the selected alert, validates its input, and converts Wazuh fields into a more predictable structure before calling the model.

That normalization step turned out to be essential. Wazuh events aren't all shaped alike: a network alert may have source IP fields, while a file-integrity alert may have a SHA-256 under `syscheck`. I added conservative extraction of explicit indicators so they would survive the inference and audit path. I now also retain the original Wazuh alert timestamp, source/destination IP fields, and Wazuh rule MITRE technique metadata. Private destination addresses remain available for display without automatically becoming public threat-intelligence lookups.

I also kept a persistent JSONL audit so the notification component could inspect what happened even after the inference request completed.

## 4. Adding External Evidence

I integrated IPsum and AbuseIPDB for eligible public IP intelligence and AlienVault OTX for multiple structured indicator types, including SHA-256 hashes. Eligible lookup values are validated before querying a provider.

The distinction between the providers matters. AbuseIPDB can report an abuse confidence score and report information for an IP. OTX reports community-pulse associations for indicators. I chose not to force those into a fake shared confidence score.

## 5. Building the Phone Alert Workflow

Instead of making the model directly send notifications, I built a separate publisher that reads the gateway audit. The publisher keeps a checkpoint and recent event identifiers, enriches valid new events, assembles the alert, and publishes through ntfy.

The alert displays the original Wazuh severity, native rule MITRE technique (not an AI guess), affected Device, Timestamp, Source IP and Destination IP immediately below Device, detection explanation, provider evidence, and actions I can investigate myself. Missing/invalid values display `N/A`; the timestamp is the Wazuh event time, not receipt time.

## 6. Hardening the System

The gateway runs under a non-root identity with a read-only root filesystem, dropped Linux capabilities, and controlled mounts. I retained a restricted network path to the gateway and kept model storage read-only while allowing the audit to persist.

The publisher has a dedicated non-root systemd identity and protected credentials. Versioned updates and rollback configurations allowed me to change the live alert format while protecting the existing delivery state.

## 7. What I Evaluated Instead of Deploying

I ran CPU-only LoRA training experiments on the Pi. The training ran successfully, but a small held-out comparison did not demonstrate better classification than the original model. The production gateway continues to use the base model. For security recommendations, I introduced evidence-first investigation instructions aligned with NIST SP 800-61 Rev. 3. During a controlled SSH test, I found the revised guidance more useful than the original approach.

See [Lessons learned](lessons-learned.md) for the tradeoffs.
