# SM-N975W N975WVLS8HWG1 — GhostLock (CVE-2026-43499) feasibility assessment

Galaxy Note10+ (Canadian Snapdragon, board `d2q`, SM8150). Assessment-only
record: no N975 payload exists yet. This documents what was verified on the
actual device + firmware and what a port requires.

## Device / firmware identity (read from the running device)

```text
model: SM-N975W (d2q, Snapdragon 855)
firmware: N975WVLS8HWG1 (SP1A.210812.016, Android 12)
kernel: 4.14.190-25890545-abN975WVLS8HWG1 (built 2023-08-01, clang 10.0.7)
SELinux: Enforcing; sys.oem_unlock_allowed = 0 (bootloader unlock not offered)
shell groups: no readtracefs gid, but tracefs reads AND enable writes work
boot.img SHA-256: AAB67B3D545FFC0CDDB699AAE0C61451699F50262BD3843513D7AA4F10CF6827
kernel SHA-256:  7F3133240424A63FF41CCA599BA53BBC33281EB1100CE5B8361044C107B9FE3C
```

Boot image is header v1 with a 0x4c-byte Samsung DTB prefix before the
ARM64 Image; the recovered ELF base is `0xffffff8008080000` (141,443
symbols via vmlinux-to-elf).

## Verified favorable facts

1. **The bug is present.** `remove_waiter()` in the d2q source unconditionally
   uses `current->pi_lock` / `current->pi_blocked_on` — the exact pre-2026
   GhostLock pattern (upstream fix switches to `waiter->task`). Device EOL
   2024 < fix 2026, and `remove_waiter` is present in the recovered symbols
   (`0xb3f9c`).
2. **Tracefs KASLR leak route is fully usable from adb shell**:
   `sched:sched_blocked_reason` exists (id **74**, device-read), the ring
   buffer is shell-readable, and shell can write event enables (verified by
   arming/disarming on device).
3. **Kernel config is permissive for exploitation**: no `MODULE_SIG`
   enforcement (unsigned modules load), no `SLAB_FREELIST_RANDOM`,
   no `SLAB_FREELIST_HARDENED`, `CONFIG_PREEMPT=y`, `SLUB=y`,
   `KALLSYMS=y`, `MODULE_FORCE_LOAD=n` (exact vermagic required),
   `STRICT_KERNEL_RWX=y`, `FAIR_GROUP_SCHED=y` (`root_task_group` exists —
   the fake-task priority route is available).
4. **Samsung's 4.14 rtmutex is the backported rb_root_cached + deadline
   variant** (`rb_erase_cached`, `task->pi_waiters` is `rb_root_cached`) —
   the same tree machinery as the 5.10 "legacy" profile already supported
   by the a15 target. The 4.14 waiter (`CONFIG_DEBUG_RT_MUTEXES=n`) is the
   compact shape: `tree_entry@0, pi_tree_entry@0x18, task@0x30, lock@0x38,
   prio@0x40, deadline@0x48`, size 0x50 (no `wake_state`/`ww_ctx`).
5. All leak/root objects exist as symbols (table below), including the
   4.14-specific `selinux_enforcing` global (no `selinux_state` struct in
   4.14).

## Symbol offsets (recovered, base 0xffffff8008080000)

| Symbol | Offset | | Symbol | Offset |
| --- | ---: | --- | --- | ---: |
| `remove_waiter` | `0xb3f9c` | | `kmalloc_caches` | `0x2311d60` |
| `worker_thread` | `0x56324` | | `__start_ftrace_events` | `0x267ba88` |
| `call_usermodehelper_exec_work` | `0x4fcdc` | | `__event_sched_blocked_reason` | `0x267bc48` (index 56) |
| `noop_llseek` | `0x20c0a4` | | `nfulnl_logger` | `0x2b9f4c0` |
| `generic_file_splice_read` | `0x24ff04` | | `init_task` | `0x2ba9780` |
| `ashmem_ioctl` | `0x10856ec` | | `system_unbound_wq` | `0x2b99f88` |
| `compat_ashmem_ioctl` | `0x1085f44` | | `root_task_group` | `0x2f55e80` |
| `ashmem_mmap` | `0x1085f9c` | | `ashmem_misc` | `0x2e1bb40` |
| `ashmem_open` | `0x108610c` | | `selinux_enforcing` | `0x3034d50` |
| `ashmem_release` | `0x108618c` | | `sysctl_bootid` | `0x30f2268` |
| `anon_pipe_buf_ops` | `0x17a1700` | | `ashmem_fops` | `0x19d0da0` |
| `configfs_read_bin_file` | `0x2bf7d4` | | `configfs_write_bin_file` | `0x2bf964` |

Worker-caller: the blocking `bl __schedule` in `worker_thread` (idle path,
after `_raw_spin_unlock_irqrestore`) is at `0x56370`; the saved return PC is
`SLIDE_TRACEFS_WORKER_CALLER_OFF = 0x00056374`.
`SLIDE_TRACEFS_EVENT_ID = 74` (device-verified; `__TRACE_LAST_TYPE` is 18 in
this 4.14).

## Required port work (the delta vs the 6.6 family)

Ordered by risk/effort; none of it is impossible, all of it is real:

1. **RELA-based data pointers.** This 4.14 Image stores `.data` pointers as
   `R_AARCH64_RELATIVE` relocations (file slots are zero; addends live in
   `.rela.dyn`). The `nfulnl_logger` name pointer and `random_table[]`
   `boot_id` entry values must be resolved from the RELA table instead of
   raw reads. Mechanical.
2. **Struct layouts without BTF.** 4.14 has no BTF; derive `task_struct`,
   `struct page`, `file_operations` (4.14 has ~0x1a8 bytes with
   `aio_read`/`readdir`/`setlease`… members), `rt_mutex`, `configfs_buffer`
   from the d2q source built with the device's exact IKCONFIG (available:
   `/proc/config.gz`), then pahole. Source: TeamWin `android_kernel_samsung_d2q`
   (4.14.83 base; verify layout drift against the .190 binary by
   disassembly spot-checks).
3. **Compact waiter path.** Exploit's fake-writer must target the 0x50-byte
   waiter (no `wake_state`/`ww_ctx`); tree ops already match the 5.10 legacy
   implementation.
4. **configfs ARW via the old API.** 4.14 uses `configfs_read_bin_file` /
   `configfs_write_bin_file` through `.read`/`.write` fops slots (no
   read_iter/write_iter on bin attributes). The fake-fops wiring and the
   `configfs_buffer` field offsets change accordingly.
5. **Slab-page detection.** No `page_type` in 4.14; replace with
   `PG_slab` flag-bit test (or `slab_cache != NULL`) in the pipe scanner,
   using 4.14 `struct page` offsets.
6. **pselect stack geometry.** Re-derive `SLIDE_PSELECT_WORD_SHIFT` from
   this kernel's `do_pselect` disassembly.
7. **KernelSU below 5.10 (largest item).** No DDK exists for 4.14; the
   module must be built against the d2q source (clang 10 per the banner)
   with vermagic `4.14.190-25890545-abN975WVLS8HWG1 ...`. KernelSU v3.2.5's
   kernel code has never been exercised on 4.14 in this project — expect API
   backports. KDP does not apply (Snapdragon); DEFEX handling from the
   Samsung patch does. kallsyms manual relocation works as root
   (`kptr_restrict=1`).

## Verdict

Feasible in principle — bug, leak, ARW targets, and module loading all
verified present on the running firmware — but it is a ground-up exploit
bring-up (items 1–6) plus a 4.14 KernelSU backport (item 7), comparable in
scale to the a15/5.10 legacy port or larger. Estimate: multi-week project;
not achievable as a follow-on task to the 6.6 profile work. The snapshot
above (offsets, config, source references) removes the reconnaissance
phase for whoever takes it on.

Reference source: [TeamWin/android_kernel_samsung_d2q](https://github.com/TeamWin/android_kernel_samsung_d2q)
(SM-N9750 Snapdragon base; same SM8150/d2q platform as SM-N975W).
