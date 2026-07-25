# zen-soplos — Zen kernel patchset for Soplos Linux

Rebase of the [zen-kernel/zen-kernel](https://github.com/zen-kernel/zen-kernel)
patchset (BMQ/PDS scheduler + assorted driver/filesystem/network tweaks),
kept building against current Soplos kernel point releases when upstream
hasn't published a matching release yet.

> Not yet wired into **soplos-kernel-installer** — `core/downloader.py` still
> fetches Zen from zen-kernel's GitHub releases directly. This repo exists so
> a working patch is ready the moment it's needed. See "Status" below.

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

No `v7.1.5-zen*` release existed upstream at the time of this rebase (7.1.5
was released 2026-07-24; zen-kernel typically publishes within days of each
stable point release — check upstream before assuming this rebase is still
needed).

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

## Status

- Verified with `patch -p1 --dry-run` against the real kernel source tree
  (kernel.org, tag `v7.1.5`, stable branch) — all 113 files (101 modified +
  12 new) apply clean, no fuzz, no rejects, exit code 0.
- **Not build-tested.** This is a much larger surface than `bore-soplos` or
  `x3d-soplos` (a full alternate scheduler plus unrelated driver/network
  code) — a clean `patch` apply says nothing about whether it compiles.
- **Not boot-tested.**
- The GPU driver hunks (`amdgpu`, `i915`, `ttm`) and TCP/BBR3 hunks were not
  individually reviewed beyond the automated apply check — they touch
  subsystems unrelated to the scheduler and were not the reason this rebase
  was needed.

Do not package a `soplos-zen` kernel from this patch until it has been
compiled and booted at least once. This patchset also conflicts with BORE
and RT by design (BMQ/PDS replaces CFS entirely) — same mutual-exclusion
rule `soplos-kernel-installer`'s patch selector already enforces.

---

## Applying the patch manually

```bash
cd /path/to/linux-7.1.5
patch -p1 < /path/to/patches/0001-zen-7.1.patch
```

---

## License

GPL-2.0 (inherited from the Linux kernel and zen-kernel's original patchset).
