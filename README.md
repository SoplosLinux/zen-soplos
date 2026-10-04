# zen-soplos — Zen kernel patchset for Soplos Linux

Rebase of the [zen-kernel/zen-kernel](https://github.com/zen-kernel/zen-kernel)
patchset (BMQ/PDS scheduler + assorted driver/filesystem/network tweaks),
kept building against current Soplos kernel point releases when upstream
hasn't published a matching release yet.

> Wired into **soplos-kernel-installer** as a fallback: `core/downloader.py`
> tries zen-kernel's own GitHub releases first, and only falls back to this
> repo (`_download_zen_soplos()`) when upstream hasn't published a patch that
> applies cleanly to the requested kernel version yet — verified with a
> dry-run apply before either source is accepted, not just a successful
> download.

---

## What this patchset is

It is **not** a small, focused patch like `bore-soplos` or `x3d-soplos`. The
upstream zen-kernel release bundles roughly 21,000 lines across 113 files:
the BMQ/PDS alternative CPU scheduler (`kernel/sched/alt_*.c`, a near-total
fork of `kernel/sched/core.c`), AMD/Intel GPU driver tweaks, TCP congestion
control additions (BBR3), a virtual SCSI HBA driver (`vhba`), cgroup/mm
changes, and various sysctl/Kconfig defaults. Soplos does not author or
maintain this code — only rebases it against newer point releases when
zen-kernel hasn't published one yet.

---

## Patch files

| File | Kernel versions | Base |
|------|-----------------|------|
| `patches/0001-zen-7.1.patch` | Linux 7.1.5 | zen-kernel `v7.1.4-zen1` release |
| `patches/0001-zen-7.2.patch` | Linux 7.2 | zen-kernel `v7.1.8-zen1` release |
| `patches/0001-zen-7.2.9.patch` | Linux 7.2.9 | zen-kernel `v7.1.8-zen1` rebase for Linux 7.2.9 |

No `v7.1.5-zen*` or `v7.2-zen*` release existed upstream at the time these
rebases were made — zen-kernel typically publishes within days of each
stable release; check upstream before assuming a rebase is still needed.

---

## Why this rebase exists (7.1.5)

Of ~700 hunks across 113 files in the `v7.1.4-zen1` release, only **2 hunks**
failed to apply against real Linux 7.1.5 sources:

1. **`Makefile`** — the `SUBLEVEL = 4` version-banner context line no longer
   matches (7.1.5's Makefile already says `SUBLEVEL = 5`, which is simply
   correct for that release). Not a real conflict — the actual payload of
   that hunk (`EXTRAVERSION`, `NAME`) applied fine on its own via normal
   offset tolerance. No manual fix was needed.

2. **`kernel/sched/alt_core.c`** (a copy-and-heavily-modify of
   `kernel/sched/core.c`) — one large hunk wholesale-replaces mainline's
   `sched_core_sysctls[]` / `sched_fork()` / `wake_up_new_task()` with
   BMQ/PDS's own versions. Between 7.1 and 7.1.5, upstream `core.c` gained
   one line inside `sched_fork()`'s `sched_reset_on_fork` branch:
   `p->timer_slack_ns = p->default_timer_slack_ns;`. That line sits inside
   the block BMQ discards entirely and replaces with its own fork-time
   logic (BMQ doesn't use policy/static_prio-based reset-on-fork), so it
   has **no functional relevance to BMQ/PDS** — it only needed to be added
   to the hunk's deletion list so `patch` could still locate and remove the
   whole superseded mainline block byte-for-byte.

**No BMQ/PDS scheduling logic was written, changed, or reinterpreted by
Soplos.** Both fixes are additions to *what upstream mainline already
changed*, applied to a block that gets replaced wholesale either way — not
new scheduler behavior.

Every other file/hunk is byte-identical to zen-kernel's release and applies
against 7.1.5 with plain line-offset only.

---

## Why this rebase exists (7.2)

A much bigger jump than 7.1→7.1.5: 7.2 is a new kernel line, not a point
release, and upstream added substantial new scheduler subsystems that
didn't exist in 7.1.8 — `CONFIG_SCHED_PROXY_EXEC` (priority inheritance via
task migration for blocked mutex owners), and expanded `CONFIG_SCHED_CORE`
/ `CONFIG_SCHED_CACHE` (LLC balancing) support.

A plain 2-way `patch` apply of `alt_core.c` (BMQ/PDS's ~8,000-line fork of
`core.c`) failed 11 of 58 hunks — over 5,000 rejected lines — not because of
genuine overlapping conflicts, but because the combined drift (BMQ's own
edits + mainline's independent changes) exceeds what a 2-way diff can
re-locate via context matching alone.

**Fix:** rebased with a real 3-way merge (`git merge-file`) instead —
BASE = `core.c` at v7.1.8, MINE = `alt_core.c` freshly derived from
zen-kernel's own v7.1.8-zen1 release patch (verified clean against that
base first), THEIRS = `core.c` at v7.2. This cut the real conflict surface
down to 21 individually-reviewed conflicts. Every single one followed the
same pattern already used throughout this file: mainline added or expanded
a CFS/EEVDF-only subsystem BMQ/PDS has never implemented, so BMQ/PDS's own
(unmodified) side was kept and mainline's new CFS-only content was
excluded — consistent with how `sched_core_sysctls[]` already excludes
`CONFIG_UCLAMP_TASK`/`CONFIG_NUMA_BALANCING` entries. **No BMQ/PDS
scheduling logic was written, changed, or reinterpreted.**

Six other files needed small hand-reconciliations (same category as the
7.1.5 fix — an anchor moved or a field was added nearby, Zen's own additions
unchanged): `Makefile` (version banner only), `init/init_task.c` (new
`.exec_state` field), `include/linux/sched.h` (task_struct fields repacked
into a u8 cluster), `kernel/sched/stats.h` (an internal struct renamed),
`kernel/sched/debug.c` (new LLC-balancing debugfs block, same `#ifndef
CONFIG_SCHED_ALT` guard as its neighbors), `include/linux/vtime.h`
(`static inline` → `static __always_inline`, no logic change). Two files
needed no changes at all — `block/bio.c` and `drivers/dma-buf/udmabuf.c`
turned out to already have Zen's exact intended change merged into
mainline independently between 7.1.8 and 7.2.

Full details, including the exact conflict-by-conflict reasoning, are in
the patch file's own commit message.

---

## Why this rebase exists (7.2.9)

Linux 7.2.9 introduced minor context drift in `kernel/Kconfig.preempt` (`SCHED_CLASS_EXT` gained `select GENERIC_ALLOCATOR`) and updated `SUBLEVEL = 9` in the top-level `Makefile`.

Additionally, upstream zen-kernel's official releases include `arch/x86/Makefile` changes for `CONFIG_X86_64_VERSION`, which conflict with Soplos Linux's ISA level selection patch (`0001-x86-64-isa-level-7.x.patch`).

**Fix:**
1. Re-aligned `SUBLEVEL = 9` in the top-level `Makefile` hunk.
2. Updated context matching for `SCHED_CLASS_EXT` in `kernel/Kconfig.preempt`.
3. Stripped the `arch/x86/Makefile` modification so that Soplos' ISA-level patch applies cleanly for V1, V2, V3, and V4 builds without collision.

---

## Status

- **7.1.5:** verified with `patch -p1 --dry-run` (all 113 files clean, no
  fuzz, no rejects), **compiled and boot-tested on real hardware** —
  `CONFIG_SCHED_ALT`/`CONFIG_SCHED_PDS` confirmed active at runtime, not
  just compiled in.
- **7.2:** verified with `patch -p1 --dry-run` (all 107 files clean, no
  fuzz, no rejects), **compiled, packaged, and boot-tested** — published as
  `linux-soplos-zen-v1` through `v4` (7.2.5) in the stable repository,
  following the same validation process as every other kernel variant
  before release.
- **7.2.9:** verified with `patch -p1 --dry-run` (all 113 files 100% clean, 0
  rejects) against official Linux 7.2.9 sources. Stripped `arch/x86/Makefile`
  hunk to prevent ISA level (V2/V3/V4) build conflicts.
- The GPU driver hunks (`amdgpu`, `i915`, `ttm`) and TCP/BBR3 hunks were not
  individually reviewed beyond the automated apply check — they touch
  subsystems unrelated to the scheduler and were not the reason either
  rebase was needed.
- The 7.2 rebase went through a substantially more invasive reconciliation
  (3-way merge, 21 manually-reviewed conflicts) than 7.1.5 did (1 line
  addition), which is exactly why it needed four extra fixes after the
  initial merge that 7.1.5 didn't (see CHANGELOG 1.1.1–1.1.5).

This patchset also conflicts with BORE and RT by design (BMQ/PDS replaces
CFS entirely) — same mutual-exclusion rule `soplos-kernel-installer`'s patch
selector already enforces.

---

## Applying the patch manually

```bash
cd /path/to/linux-7.1.5
patch -p1 < /path/to/patches/0001-zen-7.1.patch

# or, for 7.2:
cd /path/to/linux-7.2
patch -p1 < /path/to/patches/0001-zen-7.2.patch

# or, for 7.2.9:
cd /path/to/linux-7.2.9
patch -p1 < /path/to/patches/0001-zen-7.2.9.patch
```

---

## License

GPL-2.0 (inherited from the Linux kernel and zen-kernel's original patchset).
