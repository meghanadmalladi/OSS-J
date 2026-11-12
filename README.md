# OSS-J

Open Source Summit Japan 2026. Updated November 12, 2026.

**Finish by November 27, 2026 (IST), within 3–4 hours per week.**

Start with [SUMMARY.md](SUMMARY.md) for the complete current plan.

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

## Documents and tracking

- [Experiments and commands](EXPERIMENTS.md)
- [Schedule and milestones](PLAN.md)
- [Presentation outline](OUTLINE.md)
- [Tracking issues](https://github.com/meghanadmalladi/OSS-J/issues)

## Status

Planning updated. Hardware validation, measurements, charts, and slides remain pending. Board model is AM64x-HSEVM. Kernel versions, ICSSG interface/queue mapping, per-instance mode support, accepted title, and session duration need confirmation. Repository is private.
