# Presentation structure

Updated November 12, 2026. See [SUMMARY.md](SUMMARY.md) for the complete plan.

## Presentation outline

Working title — Choosing a NAPI Mode for Predictable Networking on PREEMPT_RT. Confirm accepted title and talk duration.

| Slide | Content | Learning |
|---|---|---|
| 1 | Central question | Select by deadline, traffic, and CPU budget |
| 2 | Problem | CPU/network contention can affect packet delivery |
| 3 | Standard NAPI on non-RT and RT | Execution context and scheduling |
| 4 | Three NAPI mechanisms | Trigger, polling context, empty-queue behavior |
| 5 | Hypotheses | Benefits to test, with no assumed winner |
| 6 | Fixed dual-core AM64x-HSEVM setup | CPU0 networking, CPU1 application/housekeeping, fixed peer |
| 7 | Periodic, bursty, sustained traffic | Arrival shape and rate both matter |
| 8 | Idle versus CPU contention results | Tail latency, late/missing replies |
| 9 | CPU cost | Cost of each mode, including idle polling |
| 10 | Explain one result | Focused trace relating poll execution to application delivery |
| 11 | Evidence-based selection table | Which mode fits which tested condition |
| 12 | Reproduction and takeaways | Repeatable procedure and practical limits |

Keep mechanisms concise and use diagrams showing IRQ thread, ksoftirqd where relevant, dedicated NAPI thread, socket queue, and application receive. A socket wakeup makes the receiver runnable; it still needs CPU time.

Backup material covers exact settings, repeat statistics, and optional tests. SO_BUSY_POLL and scheduling sweeps are deferred. Populate findings only from measurements.

## Sources

- [NAPI documentation](https://docs.kernel.org/networking/napi.html)
- [PREEMPT_RT execution differences](https://docs.kernel.org/core-api/real-time/differences.html)
- [Threaded busy-poll proposal](https://lwn.net/Articles/1034942/)
- [Sockperf](https://github.com/Mellanox/sockperf)
- [Sockperf options and pacing](https://github.com/Mellanox/sockperf/blob/sockperf_v2/src/sockperf.cpp)
- [Linux scheduling](https://man7.org/linux/man-pages/man7/sched.7.html)
- [stress-ng manual](https://manpages.ubuntu.com/manpages/noble/man1/stress-ng.1.html)
- [trace-cmd](https://github.com/rostedt/trace-cmd)
