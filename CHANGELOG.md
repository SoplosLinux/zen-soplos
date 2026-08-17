# Changelog — zen-soplos

## 1.1.0 — 2026-08-17

### Added

- `0001-zen-7.2.patch` — rebase of zen-kernel's `v7.1.8-zen1` release
  (109 files, ~21,600 lines) so it applies against Linux 7.2.

### Fixed

- `kernel/sched/alt_core.c`: a plain 2-way `patch` apply failed 11 of 58
  hunks (>5,000 rejected lines) because of accumulated drift between BMQ's
  own edits and mainline's independent 7.1.8→7.2 changes — not genuine
  overlapping conflicts. Rebased with a proper 3-way merge (`git
  merge-file`, BASE=v7.1.8 core.c / MINE=v7.1.8-based alt_core.c /
  THEIRS=v7.2 core.c) instead, reducing this to 21 individually-reviewed
  conflicts. Every one was mainline adding/expanding a CFS/EEVDF-only
  subsystem BMQ/PDS doesn't implement (`CONFIG_SCHED_PROXY_EXEC`,
  `CONFIG_SCHED_CORE`, `CONFIG_SCHED_CACHE`, generic NUMA-balancing state,
  `CONFIG_UCLAMP_TASK` sysctls) — BMQ/PDS's own side kept unmodified in
  every case, consistent with patterns already present elsewhere in the
  file. One placement bug (an orphaned `#endif` from a `CONFIG_SCHED_SMT`
  block mainline dropped) was found and fixed during self-review by direct
  comparison against BMQ/PDS's own v7.1.8-based source.
- `init/init_task.c`: upstream added `.exec_state = &init_task_exec_state,`
  between `.active_mm` and `.restart_block` — kept as shared context,
  BMQ/PDS's own fields (`.on_cpu`, `.prio`, `.sq_node`, etc.) unchanged.
- `include/linux/sched.h`: upstream repacked `on_cpu`/`on_rq` into a new u8
  cluster (`is_blocked`, `__pad` added) and moved `on_rq`'s position —
  BMQ/PDS's own `task_struct` fields re-anchored after the new cluster.
- `kernel/sched/stats.h`: upstream renamed `struct sched_entity_stats` to
  `struct cfs_tg_state` — kept the existing `#ifndef CONFIG_SCHED_ALT`
  guard around the (renamed) content.
- `kernel/sched/debug.c`: upstream added a new `CONFIG_SCHED_CACHE`
  (LLC balancing) debugfs block — included inside the same `#ifndef
  CONFIG_SCHED_ALT` region as every other CFS-only debugfs entry in that
  function, matching the existing pattern.
- `include/linux/vtime.h`: upstream changed several functions from
  `static inline` to `static __always_inline` (compiler hint only) —
  re-applied the same qualifier change; function bodies untouched (one is
  now named `vtime_generic_enabled_cpu` instead of
  `context_tracking_enabled_cpu`, upstream's rename, not ours).
- `Makefile`: version banner only, no real conflict.

### Notes

- `block/bio.c` and `drivers/dma-buf/udmabuf.c` needed **no changes** —
  the exact change Zen's patch makes was independently upstreamed into
  mainline between 7.1.8 and 7.2 (verified byte-for-byte identical,
  including doc comments).
- Verified with `patch -p1 --dry-run` against kernel.org tag `v7.2`, all
  107 touched files, exit code 0, no fuzz, no rejects.
- Not build-tested, not boot-tested. This rebase is substantially more
  invasive than 7.1.5's (3-way merge + 21 manual conflicts vs. a single
  line addition) — treat with proportionally more caution.
- Not submitted upstream. Soplos Linux rebase of zen-kernel/zen-kernel.

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
