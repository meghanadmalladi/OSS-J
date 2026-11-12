# Delivery plan

See [SUMMARY.md](SUMMARY.md) for the complete plan and [EXPERIMENTS.md](EXPERIMENTS.md) for commands.

## Deadline and effort

Complete by November 27, 2026, IST. Cap personal effort at 3–4 hours per week. Updated November 12. Weekly allocations include work already spent that week; remaining capacity must be checked rather than restarting the budget.

| Week | Work | Minutes |
|---|---|---|
| Nov 9–15 | Verify two DUTs, kernel/mode support, and session duration | 30 |
| Nov 9–15 | Validate sockperf, placement, logs, and traffic pacing | 75 |
| Nov 9–15 | Prepare repeatable run commands and slide skeleton | 75 |
| Nov 9–15 | Contingency | 60 |
| Nov 16–22 | Run 18-condition matrix with repeats and collect evidence | 100 |
| Nov 16–22 | Limited scheduling-policy follow-up | 20 |
| Nov 16–22 | Analyse and plot results | 45 |
| Nov 16–22 | Freeze conclusions and evidence | 15 |
| Nov 16–22 | Rerun/diagnostic contingency | 60 |
| Nov 23–27 | Complete slides and speaker notes | 150 |
| Nov 23–27 | Rehearse and revise | 60 |
| Nov 23–27 | Export and inspect final deliverables | 30 |

Total 10 planned hours plus 2 contingency hours. This assumes a working kernel and usable boards. Optional tests consume contingency only after core evidence and slide progress are secure.

## Milestones

By November 15

- [ ] Confirm exact boards, kernels, NIC drivers, three-mode support, accepted title, and session duration.
- [ ] Validate two-DUT setup and fixed server configuration.
- [ ] Verify application threads, IRQ/NAPI placement, and baseline scheduling.
- [ ] Validate RTT/sample logs and select deadline and sustained rate.
- [ ] Prepare repeatable runs and deck skeleton.

By November 22

- [ ] Complete 18 conditions with repeats.
- [ ] Complete a limited SCHED_OTHER/FIFO comparison.
- [ ] Save raw evidence and commands.
- [ ] Generate latency and CPU figures with loss/sample counts.
- [ ] Freeze conclusions; take a diagnostic trace only where needed.

By November 27

- [ ] Complete deck by November 25.
- [ ] Rehearse by November 26.
- [ ] Export and verify final PDF by November 27.
- [ ] Commit editable deck, PDF, notes, figures, and tested commands.
- [ ] Attach test conditions to every selection recommendation.

## Scope decisions

If kernel support requires work beyond the setup allowance, decide by November 15 whether an already available compatible image can be used. Otherwise explicitly disclose any unsupported mode as unmeasured, retaining its mechanism discussion.

Prioritize the core matrix and scheduling follow-up. Memory stress, switch stress, background network traffic, RR, payload sweeps, and moderation/budget sweeps are optional. If even core work exceeds the remaining budget, record a reduced scope before running it and preserve repeat measurements.

Status is planning complete; hardware validation, experiments, and slides are not yet reported complete. Track time spent, blockers, and evidence in issues #1–#3.
