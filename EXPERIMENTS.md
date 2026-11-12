# Experiment guide

Updated November 12, 2026. See [SUMMARY.md](SUMMARY.md) for the complete plan.

## Objective and scope

Under one fixed AM64x-HSEVM configuration, which NAPI mode delivers predictable application latency for each traffic pattern and CPU budget?

Compare standard NAPI, threaded NAPI, and threaded NAPI with busy polling on PREEMPT_RT. Define optimum as meeting a preselected RTT deadline with acceptable loss and the lowest CPU cost. Recommendations must identify the tested workload and hardware conditions.

Teach the mechanisms briefly, then give the audience measured selection guidance. Socket SO_BUSY_POLL and scheduling-policy/priority sweeps are deferred to later work.

| Mode | Mechanism | Question |
|---|---|---|
| Standard NAPI | Interrupt schedules softirq-based polling | How does the default receive path handle different loads? |
| Threaded NAPI | Dedicated kernel thread executes scheduled polling | How does a dedicated poller affect the tail under fixed scheduling? |
| Threaded NAPI busy polling | Dedicated kernel thread continuously checks the queue | Is the latency benefit worth the CPU cost? |

On PREEMPT_RT, standard NAPI may execute in IRQ-thread context or ksoftirqd. Dedicated threaded NAPI has its own napi/<interface>-<id> thread. Ordinary threaded NAPI normally completes polling and returns to interrupts when idle. Timer-based IRQ deferral is separately configurable. A blocking recvfrom()/recvmsg() returns available data or sleeps inside the call until data or another terminating condition arrives. After processing, the application calls receive again.

## Setup and CPU placement

Use two AM64x-HSEVM boards, each with two Cortex-A cores, running the same PREEMPT_RT kernel build. Use one ICSSG Ethernet port on each board with a direct cable. ICSSG is the selected driver for this study; confirm interface names and RX queue/NAPI mapping during setup.

### Layouts considered and selected layout

With only two cores, networking, the application, and housekeeping cannot each have a separate CPU.

| Layout | CPU0 | CPU1 | What it emphasizes |
|---|---|---|---|
| **1 — selected** | Isolated CPU for Ethernet IRQs/NAPI | Application + housekeeping | Comparing NAPI modes with reduced unrelated interference on the polling CPU |
| 2 — alternative | Ethernet IRQs/NAPI + housekeeping | Isolated CPU for the application | Comparing networking under system activity while reducing interference on application execution |

**Choose layout 1: CPU0 isolated for IRQ/NAPI; CPU1 for the application and housekeeping.** The mechanism under comparison executes on CPU0, so this layout prioritizes controlling unrelated scheduling interference there. It also separates the continuously polling NAPI thread from the application. Keep this allocation on both boards across all three modes, with the server's NAPI mode fixed.

This is an experimental design choice, not a measured claim that layout 1 always gives lower latency. Layout 2 is also valid, but answers a different question about NAPI sharing CPU time with housekeeping. We will not add a layout sweep to the core matrix.

Isolation here means excluding ordinary unrelated task placement and moving movable housekeeping away from CPU0, alongside explicit IRQ/NAPI affinity. It does not eliminate all per-CPU kernel activity. Verify the actual polling context and CPU rather than assuming affinity alone establishes isolation.

The trade-off is that sockperf shares CPU1 with housekeeping and, in loaded client runs, stress-ng. Report end-to-end RTT on this fixed two-core system. Application scheduling and the peer's response time remain part of the measurement, so RTT differences alone do not isolate NAPI execution time. CPU1 stress tests application-side contention and shared-resource effects; it does not directly stress the NAPI CPU.

### Published xsk_rr benchmark as context

The [threaded NAPI busy-poll proposal](https://lwn.net/Articles/1034942/) describes two machines and the following configuration:

| Component | Published placement/settings |
|---|---|
| Ethernet IRQs | CPU2 |
| Busy-polling NAPI thread | CPU2, SCHED_FIFO priority 50 |
| Client and server applications | Allowed on CPUs 3–5, SCHED_FIFO priority 50 |
| Client sender | Separate thread |
| Other controls | SMT disabled, one RX and one TX queue, RPS/RFS disabled, hardware interrupt moderation disabled |

The application affinity is an allowed CPU set, not a published one-thread-per-core assignment. Four distinct CPU IDs are allocated across networking and application execution, but the total physical core count and simultaneous utilization are unspecified. The post does not establish a complete housekeeping/isolation setup or that PREEMPT_RT was enabled. Its results labelled “NAPI threaded” refer to threaded NAPI busy polling, not ordinary threaded NAPI.

Our two-core sockperf experiment therefore does not reproduce that benchmark's CPU allocation or AF_XDP receive path. It evaluates the three NAPI modes within the AM64x resource constraint.

### Fixed configuration

| Setting | Client DUT — evaluated board | Server DUT — fixed peer |
|---|---|---|
| CPU0 / Core A | Ethernet RX/TX IRQs and NAPI processing | Ethernet RX/TX IRQs and NAPI processing |
| CPU1 / Core B | All sockperf threads and housekeeping | sockperf server and housekeeping |
| Test RX NAPI mode | Standard, threaded, or threaded busy polling | Standard throughout |
| Application scheduling | SCHED_OTHER | SCHED_OTHER |
| Dedicated NAPI scheduling | SCHED_OTHER when enabled | Not applicable to the standard baseline |
| IRQ-thread scheduling | Kernel defaults, recorded | Kernel defaults, recorded |

Within each traffic/load condition, change only the client RX NAPI mode. Traffic pattern and idle/loaded state are the planned workload variables. Keep the server configuration fixed and free of added stress.

Configure once and retain across all runs

- Reserve CPU0 from ordinary task placement with a cpuset isolated partition. Explicitly assign networking work there. taskset alone supplies affinity and does not reserve a CPU.
- Place movable unrelated IRQs, tasks, and workqueues on CPU1. Some per-CPU kernel activity remains.
- Pin relevant Ethernet IRQs to CPU0 and prevent irqbalance from overriding assignments.
- Explicitly pin each dedicated test NAPI thread to CPU0 when created; verify SCHED_OTHER.
- Disable RPS/RFS on the test receive path.
- Disable socket busy polling and optional software IRQ deferral. Record net.core.busy_read, net.core.busy_poll, gro_flush_timeout, and napi_defer_hard_irqs.
- Retain one fixed hardware interrupt moderation setting, MTU, offload configuration, queue configuration, poll budget, and CPU-frequency policy. Record actual values.

ICSSG has distinct RX and TX NAPI processing. Use per-NAPI controls to switch the RX instance carrying the test flow. Keep TX NAPI in its baseline mode. Avoid enabling busy polling device-wide and creating multiple continuously polling TX/RX threads on CPU0. Verify the exact kernel's per-instance control support before the main run.

Record the actual standard-NAPI execution context and policy on RT. It may run in IRQ-thread context or ksoftirqd. Fixed settings do not make those contexts identical to a dedicated SCHED_OTHER NAPI thread.

Run the fixed CPU stressor only on client CPU1. It tests application-side contention and shared-resource effects, rather than direct CPU competition with the poller on CPU0.

Validate actual RX polling placement and achieved sockperf rate once before the matrix. Sender and receiver threads share CPU1 under SCHED_OTHER. Keep this placement for all modes.

RTT uses the client clock, requiring no inter-machine PTP. Both boards contribute to RTT. Record server CPU/load and loss so peer bottlenecks can be recognized. A focused diagnostic trace may help attribute a difference.

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

The load conditions are idle and CPU contention. Use one fixed stress placement for the core matrix, client CPU1 with the application throughout. Repeat each condition three times for 60 seconds of measurement with 10 seconds of warmup. This is 54 minutes measured time and about 63 minutes including warmup, before setup and collection. Alternate mode order across repeats.

Warmup must be consistently excluded from analysis; validate the installed tool's behavior. Do not claim a 60-second file automatically excludes a separately intended warmup.

## Fixed scheduling

Keep applications and dedicated NAPI threads at SCHED_OTHER. Preserve IRQ-thread defaults and record them. Scheduling-policy and priority comparisons are deferred so the study focuses on NAPI modes under one fixed setup.


## Stress workloads

These reproduce resource pressure, with application analogies rather than claims of exact workload emulation. Run one stressor at a time.

| Stress | Approximation | Priority |
|---|---|---|
| CPU at partial load | Analytics/compression/background computation | Core matrix |
| Memory access | Image/telemetry buffer processing, shared cache/memory contention | Optional |
| Context switching | IPC-heavy cooperating services | Optional |
| Network background | Diagnostics/logging/software transfer | Optional |

CPU stress is fixed to client CPU1. Memory and switch examples are future extensions using the same placement.

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

Final recommendation table should map traffic pattern, contention, and CPU cost under the fixed scheduling configuration to measured latency, loss, and CPU cost. Findings remain pending.

## Sources

- [NAPI documentation](https://docs.kernel.org/networking/napi.html)
- [PREEMPT_RT execution differences](https://docs.kernel.org/core-api/real-time/differences.html)
- [Threaded busy-poll proposal](https://lwn.net/Articles/1034942/)
- [Sockperf](https://github.com/Mellanox/sockperf)
- [Sockperf options and pacing](https://github.com/Mellanox/sockperf/blob/sockperf_v2/src/sockperf.cpp)
- [Linux scheduling](https://man7.org/linux/man-pages/man7/sched.7.html)
- [stress-ng manual](https://manpages.ubuntu.com/manpages/noble/man1/stress-ng.1.html)
- [trace-cmd](https://github.com/rostedt/trace-cmd)
