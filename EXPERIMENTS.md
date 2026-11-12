# Experiment guide

Updated November 12, 2026. See [SUMMARY.md](SUMMARY.md) for the complete plan.

## Objective and scope

For a given traffic pattern and CPU budget, which NAPI mode delivers predictable application latency, and when does scheduling policy change the answer?

Compare standard NAPI, threaded NAPI, and threaded NAPI with busy polling on PREEMPT_RT. Define optimum as meeting a preselected RTT deadline with acceptable loss and the lowest CPU cost. Recommendations must identify the tested workload and hardware conditions.

Teach the mechanisms briefly, then give the audience measured selection guidance. Socket SO_BUSY_POLL is deferred to later work.

| Mode | Mechanism | Question |
|---|---|---|
| Standard NAPI | Interrupt schedules softirq-based polling | How does the default receive path handle different loads? |
| Threaded NAPI | Dedicated kernel thread executes scheduled polling | Does control over poller scheduling improve the tail? |
| Threaded NAPI busy polling | Dedicated kernel thread continuously checks the queue | Is the latency benefit worth the CPU cost? |

On PREEMPT_RT, standard NAPI may execute in IRQ-thread context or ksoftirqd. Dedicated threaded NAPI has its own napi/<interface>-<id> thread. Ordinary threaded NAPI normally completes polling and returns to interrupts when idle. Timer-based IRQ deferral is separately configurable. A blocking recvfrom()/recvmsg() returns available data or sleeps inside the call until data or another terminating condition arrives. After processing, the application calls receive again.

## Setup and CPU placement

Use two DUTs connected by Ethernet. Run sockperf client on the evaluated DUT and sockperf server on the peer. Change modes, stress, and scheduling only on the client during the initial study. Hold the server configuration fixed and record it.

Starting layout on each dual-core board

| Core | Work |
|---|---|
| Core A / CPU0 | Ethernet IRQs and NAPI processing |
| Core B / CPU1 | Benchmark application and housekeeping |

Set dedicated NAPI-thread affinity explicitly. taskset supplies affinity; cpusets/isolation and IRQ/workqueue placement are also needed to reduce unrelated work on Core A. Verify actual placement and RPS settings. Some per-CPU kernel activity remains.

Start applications and dedicated NAPI threads with SCHED_OTHER for the mode comparison. Record the actual IRQ-thread and standard-NAPI execution policies. Inspect sockperf sender/receiver threads and achieved rate before applying RT policy to the application. Multiple runnable threads sharing CPU1 can affect pacing.

RTT uses the client clock, requiring no inter-machine PTP. Both DUTs contribute to RTT; the result is end-to-end application latency. A focused DUT trace can help attribute a difference.

## Tool choice

| Tool | Role and rationale |
|---|---|
| sockperf | Primary measurement tool using ordinary UDP sockets, rate/burst controls, RTT statistics, and sample logs |
| stress-ng | Controlled CPU, memory, or scheduler interference |
| mpstat / pidstat | Per-core and per-thread CPU use |
| trace-cmd / KernelShark | One short diagnostic trace explaining a result |
| iperf3 | Optional background network traffic |
| rtc-testbench | Deferred because its industrial workload setup and the user's synchronization workflow exceed the desired effort |
| xsk_rr | Deferred because AF_XDP setup and experimental benchmark validation add work and characterize a different receive path |
| pwru | Optional investigation of a specific packet path or drop |

Sockperf offers a manageable ordinary-socket experiment. Software rate control is nominal pacing. Validate achieved rates; avoid claiming a hardware-timed control cycle.

## Traffic patterns

Use 64-byte sockperf messages initially. All values below are proposed starting points.

| Pattern | Initial settings | Workload analogy | Question |
|---|---|---|---|
| Sparse periodic | 1,000 messages/s, burst 1 | Cyclic control/status | Wakeup costs and CPU cost during gaps |
| Bursty | Burst 16 at the same average 1,000 messages/s | Sensors reporting together or event bursts | Queue buildup and batch draining |
| Sustained small packets | Brief calibration at 5k, 10k, 20k messages/s as supported | Telemetry aggregation/gateway | Capacity and latency growth |

Select one sustainable common high rate for the main matrix. Report actual rate and losses. Compare periodic and bursty traffic at equal average message rates. An optional overload point can show degradation.

## Core matrix

3 modes × 3 traffic patterns × 2 load conditions = 18 conditions.

The load conditions are idle and CPU contention. Use one fixed stress placement for the core matrix, initially CPU1 with the application. Repeat each condition three times for 60 seconds of measurement with 10 seconds of warmup. This is 54 minutes measured time and about 63 minutes including warmup, before setup and collection. Alternate mode order across repeats.

Warmup must be consistently excluded from analysis; validate the installed tool's behavior. Do not claim a 60-second file automatically excludes a separately intended warmup.

## Scheduling follow-up

Use one revealing traffic pattern and CPU contention condition to compare SCHED_OTHER with SCHED_FIFO for the two dedicated NAPI modes. Keep application and IRQ-thread policies unchanged. Record the chosen FIFO priority and its relation to competing threads.

If needed, move the stress worker to Core A for this separate experiment and collect an SCHED_OTHER baseline at that exact placement first. Reuse results only when all other settings match. Reserve roughly 20 minutes for this limited follow-up.

Standard NAPI has no dedicated per-instance scheduling policy. Changes to IRQ-thread or ksoftirqd policy affect a broader execution context and belong in a separately identified experiment.

SCHED_RR is optional and most informative with an equal-priority runnable RT competitor. FIFO and RR may behave similarly without one. A FIFO busy poller may starve ordinary work on its core. Record achieved stress-worker progress as well as networking latency.

## Stress workloads

These reproduce resource pressure, with application analogies rather than claims of exact workload emulation. Run one stressor at a time.

| Stress | Approximation | Priority |
|---|---|---|
| CPU at partial load | Analytics/compression/background computation | Core matrix |
| Memory access | Image/telemetry buffer processing, shared cache/memory contention | Optional |
| Context switching | IPC-heavy cooperating services | Optional |
| Network background | Diagnostics/logging/software transfer | Optional |

Example commands for CPU1; change placement only in a documented experiment.

~~~bash
taskset -c 1 stress-ng --cpu 1 --cpu-method int64 \
    --cpu-load 70 --cpu-load-slice 10 --timeout 90s --metrics-brief

taskset -c 1 stress-ng --vm 1 --vm-bytes 64M --vm-keep \
    --timeout 90s --metrics-brief

taskset -c 1 stress-ng --switch 1 --switch-freq 10000 \
    --timeout 90s --metrics-brief
~~~

Select memory size within available RAM. Requested CPU load and switch rate may differ from achieved values. Keep stress active through warmup and measurement; coordinate startup accordingly.

Optional network extension uses iperf3 toward the client DUT while periodic probes continue. Record achieved background throughput/loss and verify RX queue mapping.

## Sockperf procedure

Check version/help on both DUTs. Use ordinary kernel sockets and disable socket busy polling for this study. Record net.core.busy_read and net.core.busy_poll and ensure the test sockets have no socket busy-poll setting. Save original settings for restoration.

Server remains fixed.

~~~bash
taskset -c 1 sockperf server -i <SERVER_IP> -p 11111
~~~

Bring-up only.

~~~bash
taskset -c 1 sockperf ping-pong -i <SERVER_IP> -p 11111 \
    -m 64 -t 30 --full-rtt
~~~

Main periodic comparison.

~~~bash
taskset -c 1 sockperf under-load -i <SERVER_IP> -p 11111 \
    -m 64 --mps 1000 --reply-every 1 -t 60 \
    --full-rtt --full-log periodic.csv
~~~

Bursty comparison.

~~~bash
taskset -c 1 sockperf under-load -i <SERVER_IP> -p 11111 \
    -m 64 --mps 1000 --burst 16 --reply-every 1 -t 60 \
    --full-rtt --full-log burst.csv
~~~

For sustained traffic, change --mps to the calibrated common rate. Use unique filenames for mode, pattern, load, policy, and repeat. These commands are starting templates and have not been executed on the target.

Under-load supports offered-rate testing while ping-pong waits for replies and can reduce offered load during stalls. Explicit --reply-every 1 requests a reply to every message; the inspected under-load source defaults to 100. --full-rtt avoids the half-RTT presentation convention. Validate full-log units, sample counts, and actual sending cadence.

## Metrics and controls

Choose an RTT deadline before examining comparative results, justified by the intended application or labelled as a demonstration threshold. A 1 ms send interval does not establish the deadline.

Collect

- Full RTT median, p99, p99.9, and maximum observed.
- Late-reply fraction and missing replies separately; state denominators.
- Per-core and per-thread CPU use, including quiet-link busy-poll cost.
- Achieved message rate, loss, run duration, sample counts, and repeat variability.
- Stress-worker CPU use/progress and any optional background throughput.
- Exact hardware, kernel/config, driver, tool versions, affinity, policy/priority, queue mapping, link speed, MTU, offloads, frequency policy, moderation, and poll budget.

Hold moderation and poll budget fixed initially. Check thermal/frequency changes across runs. Derive losses from supported counters/sequence accounting, not latency records alone. Exclude missing replies from successful-latency percentiles but show their count prominently. Tail estimates require enough samples; maximum observed is not a proven hard bound.

Take one short diagnostic trace in a separate run. Keep tracing disabled for headline measurements.

## Evidence and final outputs

Create data paths when measurements exist

- results/<run-id>/metadata.md
- results/<run-id>/sockperf.log and samples.csv
- results/<run-id>/cpu.log and stress.log
- results/summary.csv
- figures/latency-tail.png and cpu-cost.png
- scripts/ containing tested reproduction commands
- Editable presentation, final PDF, and speaker notes

Final recommendation table should map traffic pattern, contention, CPU budget, and scheduling policy to measured latency, loss, and CPU cost. Findings remain pending.

## Sources

- [NAPI documentation](https://docs.kernel.org/networking/napi.html)
- [PREEMPT_RT execution differences](https://docs.kernel.org/core-api/real-time/differences.html)
- [Threaded busy-poll proposal](https://lwn.net/Articles/1034942/)
- [Sockperf](https://github.com/Mellanox/sockperf)
- [Sockperf options and pacing](https://github.com/Mellanox/sockperf/blob/sockperf_v2/src/sockperf.cpp)
- [Linux scheduling](https://man7.org/linux/man-pages/man7/sched.7.html)
- [stress-ng manual](https://manpages.ubuntu.com/manpages/noble/man1/stress-ng.1.html)
- [trace-cmd](https://github.com/rostedt/trace-cmd)
