# Delivery plan

Finish by November 27, 2026, IST. Allocate at most four hours in each week below. Today, November 8, is planning/setup by the assistant. User execution starts November 9.

## Weekly schedule

| Dates | Task | Minutes |
|---|---|---|
| Nov 9–15 | Confirm board, kernel, NIC driver, three-mode support, and talk duration | 30 |
| Nov 9–15 | Install tools and establish sockperf UDP echo over Ethernet | 60 |
| Nov 9–15 | Record mode switching, CPU placement, fixed settings, and a smoke test | 60 |
| Nov 9–15 | Create slide skeleton from OUTLINE.md | 30 |
| Nov 9–15 | Setup contingency | 60 |
| Nov 16–22 | Run the nine-cell comparison with repeats and save metadata | 90 |
| Nov 16–22 | Generate latency and CPU charts, check losses and anomalies | 45 |
| Nov 16–22 | Test one targeted tuning change or take one short diagnostic trace | 30 |
| Nov 16–22 | Write measured conclusions and freeze results | 15 |
| Nov 16–22 | Rerun contingency | 60 |
| Nov 23–27 | Build the complete slide deck and speaker notes | 150 |
| Nov 23–27 | Rehearse once and correct timing and unclear claims | 60 |
| Nov 23–27 | Export PDF, verify figures, and commit final deliverables | 30 |

Total planned work is 10 hours, plus 2 hours reserved for contingency. Week totals are 4 hours maximum each. Benchmark runtime is included in the measurement allocation.

## Milestone 1 — runnable setup by November 15

- [ ] Record exact kernel/config, board, driver, NIC, link speed, and available NAPI controls.
- [ ] Confirm all three modes actually run on the target.
- [ ] Record IRQ, NAPI thread, application CPU affinity, and scheduling policy.
- [ ] Complete a short sockperf run and verify full RTT interpretation and sample export.
- [ ] Agree on a demonstrative RTT deadline before inspecting comparative results.
- [ ] Create the deck skeleton.

If threaded busy polling requires a kernel backport or rebuild that cannot fit the setup budget, decide by November 15 whether to use an already available compatible image. Otherwise measure the supported modes and explicitly mark threaded busy polling as conceptual/unmeasured. Preserve its discussion in the talk, and record this scope change.

## Milestone 2 — evidence frozen by November 22

- [ ] Complete the matrix in EXPERIMENTS.md.
- [ ] Save exact commands/settings and raw output for each run.
- [ ] Produce a tail-latency comparison and a per-core CPU comparison.
- [ ] Check losses, sample counts, repeat variability, and unexpected results.
- [ ] Demonstrate one useful tuning change when the evidence supports it.
- [ ] Write three conclusions, each tied to a figure.

Prefer one clean result over expanding to extra boards, frame sizes, or priority sweeps. Poll budget and moderation receive a concise trade-off discussion unless there is spare measurement time.

## Milestone 3 — presentation complete by November 27

- [ ] Complete slides by November 25.
- [ ] Rehearse and revise by November 26, using the confirmed session length.
- [ ] Export and inspect the final PDF by November 27.
- [ ] Save editable slides, PDF, speaker notes, charts, and reproduction commands.
- [ ] Link measured recommendations to their test conditions.
- [ ] State maximum observed latency with run duration/sample count.
- [ ] Keep a recorded trace or static figures available for the demonstration.

## Tracking

Use the three repository issues as weekly checklists. Add time spent, evidence paths, blockers, and the next action to the relevant issue after each work session.
