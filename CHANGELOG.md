# Changelog — zen-soplos

## 1.0.0 — 2026-07-25

Initial release.

### Added

- `0001-zen-7.1.patch` — rebase of zen-kernel's `v7.1.4-zen1` release
  (113 files, ~21,000 lines: BMQ/PDS scheduler + driver/network/fs patches)
  so it applies against Linux 7.1.5.

### Fixed

- `kernel/sched/alt_core.c`: one hunk (wholesale replacement of mainline's
  `sched_core_sysctls[]` / `sched_fork()` / `wake_up_new_task()` with
  BMQ/PDS's own versions) failed because 7.1.5's `core.c` gained one line
  (`p->timer_slack_ns = p->default_timer_slack_ns;`) inside the mainline
  block being discarded. Added that line to the hunk's deletion context so
  it matches real 7.1.5 content; no BMQ/PDS logic touched.
- `Makefile`: version-banner hunk (`SUBLEVEL`) no longer matched context by
  design (7.1.4 vs 7.1.5) — resolved on its own via normal offset tolerance,
  no manual change needed.

### Notes

- Verified with `patch -p1 --dry-run` against kernel.org tag `v7.1.5`
  (stable branch), all 113 files, exit code 0.
- Not build-tested, not boot-tested.
- Not submitted upstream. Soplos Linux rebase of zen-kernel/zen-kernel.
