# OSS-J

Open Source Summit Japan 2026. Updated November 12, 2026.

**Finish by November 27, 2026 (IST), within 3–4 hours per week.**

Start with [SUMMARY.md](SUMMARY.md) for the complete current plan.

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

## Documents and tracking

- [Experiments and commands](EXPERIMENTS.md)
- [Schedule and milestones](PLAN.md)
- [Presentation outline](OUTLINE.md)
- [Tracking issues](https://github.com/meghanadmalladi/OSS-J/issues)

## Status

Planning updated. Hardware validation, measurements, charts, and slides remain pending. Board models, kernel versions, accepted title, and session duration need confirmation. Repository is private.
