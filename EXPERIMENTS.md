# Minimal experiment plan

## Setup

Use a direct Ethernet link between a Linux laptop running the sockperf client and the PREEMPT_RT board running the server. Measure full round-trip latency on the client clock. PTP is unnecessary for RTT. RTT includes both hosts, both network directions, and server processing.

Use ordinary kernel sockets. Record sockperf version, exact invocation, and timing behavior on the chosen architecture. Start with ping-pong to verify setup, then use rate-driven under-load mode for comparisons. Confirm reply sampling and exported sample counts in the installed version.

## Proposed fixed probe

Start with 64-byte UDP application payloads and 1,000 messages/s. These are experiment choices to validate during setup. Record actual achieved rate. Choose an RTT deadline before comparative runs based on the intended workload, or label it clearly as a demonstration threshold. A 1 ms send interval does not by itself establish a 1 ms application deadline.

## Matrix

| Condition | Standard | Threaded | Threaded busy poll |
|---|---|---|---|
| Quiet link | Pending | Pending | Pending |
| CPU contention | Pending | Pending | Pending |
| Competing receive traffic | Pending | Pending | Pending |

For CPU contention, pin a stress-ng CPU worker to the selected receive-processing CPU and record its scheduling class. For network contention, send iperf3 UDP traffic from laptop toward board on a separate port. Select one sustainable background rate during setup and hold it constant. Verify shared RX queue placement if claiming direct queue contention.

Run each cell for 60 seconds after 10 seconds of warmup, three times. This is about 32 minutes of runtime for 27 runs, leaving time for switching and collection within the 90-minute allocation. Alternate mode order between repeats where practical.

## Controls

Record and hold constant where applicable

- Board, NIC, driver, kernel version/config, link speed, MTU, offloads, and interrupt moderation.
- IRQ affinity, application placement, scheduling class/priority, and CPU frequency policy.
- RX queue mapping and any RPS/RFS configuration.
- Probe rate, payload size, background rate, stress placement, and run duration.
- NAPI mode controls and effective thread affinity/policy for each run.

Standard NAPI uses its actual RT execution context. Record that context. Dedicated-thread configuration has different controls, so document differences rather than describing every setting as equivalent.

## Measurements

- Full RTT median, p99, p99.9, and maximum observed.
- Replies exceeding the chosen deadline and missing replies, reported separately.
- Sample count, duration, and results across repeats.
- Per-core CPU utilization using mpstat, including the idle-link busy-poll cost.
- Background receive throughput/loss, plus NIC drops if available.

Do not infer loss from latency logs alone. Verify transmitted and received counters or sequence accounting. Do not count missing replies as successful low-latency samples. If the tool cannot supply a metric reliably, state that limitation.

## One follow-up experiment

Choose the clearest contention result. Change one factor, such as NAPI thread placement or priority, then repeat that condition. Explain improvement or regression in both latency and CPU terms. Record the exact before/after settings.

Use a short trace-cmd capture of scheduler, IRQ/softirq, and NAPI events available on the target if needed to explain the result. Collect benchmark charts in runs without tracing. pwru remains optional for a specific packet path or drop investigation.

## Evidence to commit

Create these paths when measurements exist

- results/<run-id>/metadata.md
- results/<run-id>/sockperf.log
- results/<run-id>/cpu.log
- results/<run-id>/background.log
- results/summary.csv
- figures/latency-tail.png
- figures/cpu-cost.png
- scripts/ — exact repeatable commands once tested

Record sample size alongside tail percentiles. Report maximum observed latency as an observation under the tested conditions. A finite run does not establish a hard worst-case bound.
