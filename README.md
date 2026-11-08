# OSS-J

Open Source Summit Japan 2026 presentation preparation.

**Deadline — November 27, 2026 (IST).** Personal effort capped at 3–4 hours per week.

## Central question

Which NAPI mode meets a packet deadline under competing load, and how much CPU does it cost?

## Problem

Latency-sensitive applications need predictable packet delivery even when other tasks and network traffic compete for resources. Compare standard NAPI, threaded NAPI, and threaded NAPI busy polling on PREEMPT_RT, and explain when placement or scheduling changes help.

## Audience takeaway

A practical way to select a NAPI mode using measured tail latency, deadline misses, and CPU consumption. Recommendations must follow the results, with the workload and hardware conditions attached.

## Documents

- [Work plan](PLAN.md) — time budget, milestones, completion checks, and fallback scope.
- [Presentation outline](OUTLINE.md) — slide sequence and intended learning.
- [Experiment plan](EXPERIMENTS.md) — minimal matrix, measurements, and reproducibility.

## Working assumptions

One existing PREEMPT_RT board and a Linux laptop connected by Ethernet. Use sockperf for UDP RTT, iperf3 for background receive traffic, stress-ng for CPU contention, and mpstat for CPU use. Use trace-cmd only for a focused diagnostic run if needed. RTT uses the client's clock and requires no PTP.

Kernel, driver, board model, talk duration, and threaded busy-poll availability are still to be confirmed. The November 27 target is the user's completion deadline. Repository visibility was chosen as private.

## Current status

Planning complete. Hardware setup, measurements, charts, and slides remain pending.
