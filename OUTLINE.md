# Presentation outline

Working title — Choosing a NAPI Mode for Predictable Networking on PREEMPT_RT.

The accepted title and session duration remain to be confirmed. Start with 12 slides and adjust pacing after that confirmation.

| Slide | Content | Audience learning |
|---|---|---|
| 1 | Title and the central question | Which mode meets the packet deadline within a CPU budget? |
| 2 | Problem — periodic packets alongside CPU and network load | Average throughput alone cannot answer the deadline question. |
| 3 | Standard NAPI on non-RT and PREEMPT_RT | Execution context and scheduling change. Keep this to one slide. |
| 4 | Standard, threaded, and threaded busy-poll NAPI | Who runs the poll, what triggers it, and which controls become available. |
| 5 | Three hypotheses | Standard as baseline, threaded for explicit scheduling/placement, busy polling for a CPU-for-latency trade-off. |
| 6 | Simple test setup | One board, one laptop, UDP RTT, no clock synchronization. |
| 7 | Workload matrix and success criteria | Quiet link, CPU contention, and competing receive traffic. |
| 8 | Measured latency tails and late/missing replies | See where the modes differ under the tested loads. |
| 9 | CPU cost and background throughput | Evaluate the cost of the latency result. |
| 10 | One tuning experiment | Explain one result using affinity/priority or a focused trace. |
| 11 | Selection guidance from the data | Map deadline, contention, and available CPU to the measured choices. |
| 12 | Reproduction steps and takeaways | How to repeat the comparison and apply the decision method. |

## Introduction boundaries

Explain only the mechanics needed to interpret the experiment. Validate the standard-NAPI RT execution path against the actual kernel and driver. Distinguish IRQ threading from dedicated NAPI threading. Distinguish threaded NAPI busy polling from application/socket busy polling.

## Result placeholders

Populate after experiments. For each recommendation, state the mode, workload, CPU placement, latency result, CPU cost, and limitation. Keep hypotheses clearly separate from measured findings.

## Backup material

Exact mode controls, complete configuration, repeat statistics, and a short discussion of poll budget and interrupt moderation. Explain their trade-offs without presenting unmeasured benefits as results.

## References

- NAPI documentation — https://docs.kernel.org/networking/napi.html
- PREEMPT_RT differences — https://docs.kernel.org/core-api/real-time/differences.html
- Threaded busy-poll patch discussion — https://lwn.net/Articles/1034942/
- sockperf — https://github.com/Mellanox/sockperf
- trace-cmd — https://github.com/rostedt/trace-cmd
- pwru, optional packet-path debugging — https://github.com/cilium/pwru
