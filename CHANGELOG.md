# Changelog — zen-soplos

## 1.1.5 — 2026-08-18

### Fixed

- **Linker failure, not a compile error** (`vmlinux.o: en la función
  'mm_init': referencia a 'mm_init_sched' sin definir`, and the same for
  `sched_update_llc_bytes` from `drivers/base/cacheinfo.c`): unlike
  `CONFIG_SCHED_PROXY_EXEC`, mainline's new `CONFIG_SCHED_CACHE` Kconfig
  entry (`init/Kconfig`) never got a `depends on !SCHED_ALT` — the patch
  already excludes BMQ/PDS from every `CONFIG_SCHED_CACHE`-guarded code
  path, but the option itself (`default y`) stayed enabled, so generic
  kernel code outside the scheduler (`kernel/fork.c`,
  `drivers/base/cacheinfo.c`, neither of them scheduler files) still
  called into the LLC-balancing functions that only exist when
  `fair.c`/`topology.c` are compiled — which they aren't under
  `CONFIG_SCHED_ALT`. Added the missing `depends on !SCHED_ALT`, matching
  the existing pattern already used for `SCHED_PROXY_EXEC`.
  Compile-and-link verified: a full `vmlinux` was built successfully
  end-to-end with `soplos-zen-v1`'s config, not just individual objects.

## 1.1.4 — 2026-08-18

### Fixed

- `kernel/sched/debug.c`, two more gaps from the same 1.1.0 merge, both
  found via direct targeted compiles (`make kernel/sched/build_utility.o`,
  `kernel/sched/`) instead of waiting on the full 26-kernel queue:
  - `static struct dentry *debugfs_sched;` was declared inside an already-
    excluded `CONFIG_SCHED_ALT` region, but `sched_init_debug()` — which
    still runs under `CONFIG_SCHED_ALT` to register a few generic
    debugfs entries — uses it unconditionally a few lines later:
    `'debugfs_sched' undeclared`. Moved the declaration to the top of the
    file, unconditional.
  - `sched_fair_server_period_write/show/open` and `fair_server_period_fops`
    reference `DL_PERIOD`/`rq->fair_server`/`sched_server_write_common`/
    `sched_server_show_common`, all CFS/deadline-server only, but sat
    outside any `CONFIG_SCHED_ALT` guard: `invalid use of undefined type
    'struct cfs_rq'` and several implicit-declaration errors. Wrapped in
    `#ifndef CONFIG_SCHED_ALT`, matching its sibling
    `sched_fair_server_runtime_*` functions which were already guarded.
  - Also confirmed (compile-verified, not just balance-checked) that
    `kernel/sched/build_policy.o`, `alt_core.o`, `alt_debug.o` and the
    whole `kernel/sched/` built-in object build clean with only two
    pre-existing, harmless warnings (`sched_show_numa` unused,
    `task_llc` missing prototype, frame-size notice on `select_task_rq`).

## 1.1.3 — 2026-08-18

### Fixed

- `kernel/sched/debug.c`: `proc_sched_show_task()` (the CFS/mainline body)
  was missing its opening `#ifndef CONFIG_SCHED_ALT` guard — the matching
  `#else` (stub for BMQ/PDS, which provides the real implementation
  separately in `alt_debug.c`) and `#endif` were already there, but
  without the opening the CFS body compiled unconditionally under
  `CONFIG_SCHED_ALT` too. Caused three classes of error in the same
  build: `'struct task_struct' has no member named 'se'/'dl'` (CFS/DL
  scheduling-class fields that don't exist under `CONFIG_SCHED_ALT`),
  `'#else' without '#if'`, and `redefinition of 'proc_sched_show_task'`
  (both this stub and `alt_debug.c`'s real one ended up compiled at
  once). First surfaced compiling `soplos-zen-v1` a second time — this is
  a fourth, distinct bug from the same 1.1.0 merge, in a different file
  than the previous three. Added the missing `#ifndef CONFIG_SCHED_ALT`
  immediately before the function definition. Verified two ways: a
  `#if`/`#endif` nesting-balance check across the whole file (0 at EOF),
  and a fresh `patch -p1` apply against pristine v7.2 sources producing a
  byte-identical `debug.c` to the one hand-fixed directly in the build
  tree to unblock the in-progress compile.

## 1.1.2 — 2026-08-18

### Fixed

- `kernel/sched/alt_sched.h` was missing `#include <linux/sched/stat.h>` —
  not a merge mistake, this gap already existed in BMQ/PDS's own header on
  7.1 too, it just never mattered there. Mainline 7.2 added a new function
  to `kernel/sched/cputime.c` (`kcpustat_idle_stop`) that calls
  `nr_iowait_cpu()`, and that file is compiled as part of
  `build_policy.c`'s translation unit, which reaches scheduler internals
  through `sched.h` → (`CONFIG_SCHED_ALT`) → `alt_sched.h` instead of the
  normal chain that pulls in `<linux/sched/stat.h>` directly. First
  surfaced compiling `soplos-zen-v1` overnight in a 26-kernel unattended
  Stock run — `error: implicit declaration of function 'nr_iowait_cpu'`.
  Added the include next to the other `linux/sched/*` headers already
  there. Verified with a fresh `patch -p1` apply against pristine v7.2
  sources: exit 0, zero fuzz, zero rejects.

## 1.1.1 — 2026-08-18

### Fixed

- `kernel/sched/alt_core.c` failed to compile against real Linux 7.2
  (first real build test of the 1.1.0 rebase, caught mid-way through a
  6-kernel unattended Stock build): two `CONFIG_SCHED_PROXY_EXEC`/
  `CONFIG_SCHED_CACHE` mainline additions leaked into the 3-way merge
  instead of being excluded like every other CFS-only addition in the
  file.
  - `try_to_wake_up()` gained an `else if (cpu != p->wake_cpu) { ...
    p->wake_cpu = cpu; }` proxy-migration fixup. `wake_cpu` itself is
    `#ifndef CONFIG_SCHED_ALT`-guarded out of `task_struct` in mainline
    (BMQ/PDS doesn't track it), so the field didn't exist —
    `error: 'struct task_struct' has no member named 'wake_cpu'`.
    Removed the clause; the 7.1 patch's equivalent function never had it.
  - `sched_cpu_deactivate()` gained a `sched_domains_free_llc_id(cpu);`
    call. Its declaration in `sched.h` is itself inside the same
    `#ifndef CONFIG_SCHED_ALT` region as the LLC-balancing block already
    excluded elsewhere in this file, so it was never declared for a
    `CONFIG_SCHED_ALT` build — `error: implicit declaration of function
    'sched_domains_free_llc_id'`. Removed the call; confirmed absent from
    the 7.1 patch's equivalent function too.
  - Verified with a fresh `patch -p1` apply against pristine v7.2 sources:
    exit 0, zero fuzz, zero rejects, for all files.

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
