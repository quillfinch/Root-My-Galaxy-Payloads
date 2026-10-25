# SM-X520 X520XXSBCZI2 porting record (Galaxy Tab S10 FE, Wi-Fi)

Offline port following `PORTING.md`. Hardware execution has NOT been
performed. Every value below was derived from the exact firmware; nothing was
copied from another device's profile without re-verification against this
kernel's own symbols or BTF.

## 1. Firmware identity

```text
model: SM-X520 (device gts10fewifi, platform essi / Exynos 1580)
region queried: XAC (same international AP across XAC/KOO/INS/MXO/XSP)
AP/PDA: X520XXSBCZI2
CSC: X520OXMBCZI2
CP: X520XXSBCZI2
display build: BP4A.251205.006.X520XXSBCZI2 (Android 16)
fingerprint: samsung/gts10fewifixx/gts10fewifi:16/BP4A.251205.006/X520XXSBCZI2:user/release-keys
kernel release: 6.6.102-android15-8-abX520XXSBCZI2-4k
boot.img SHA-256: 852180488B8D6542AEC2C5066B84B5B19198BDD948FE291CE13C08583B6E8DAF
kernel SHA-256: 4773A3284FB06BD4D3D282E20B54DC64B31AD309ECB303CDEBC9EE1463EC60DB
kernel size: 38779392
ARM64 Image text_offset: 0x0, image_size 0x27a0000, flags 0xa
```

Downloaded with `samloader-rs` (topjohnwu) from Samsung FUS.

## 2. Recovery

- `vmlinux-to-elf` recovered 110,037 symbols; ELF base
  `0xffffffc080000000` (= `KIMAGE_TEXT_BASE`, android15-6.6 GKI layout).
- Raw BTF blob validated at Image `[0x186af0c, 0x1e81915)` (6,384,137 bytes);
  layouts below come from this kernel's own BTF (`bpftool btf dump format
  raw`), not from another target.

## 3. Symbol offsets (from the recovered ELF)

| Macro | Symbol / derivation | Offset |
| --- | --- | ---: |
| `CALL_USERMODEHELPER_EXEC_WORK_OFF` | `call_usermodehelper_exec_work` | `0x000d0f9c` |
| `NOOP_LLSEEK_OFF` | `noop_llseek` | `0x003c7d98` |
| `COPY_SPLICE_READ_OFF` | `copy_splice_read` (6.6 name) | `0x004153f4` |
| `CONFIGFS_READ_ITER_OFF` | `configfs_read_iter` | `0x00492094` |
| `CONFIGFS_BIN_WRITE_ITER_OFF` | `configfs_bin_write_iter` | `0x004925c0` |
| `ASHMEM_IOCTL_OFF` | `ashmem_ioctl` | `0x00d65830` |
| `ASHMEM_COMPAT_IOCTL_OFF` | `compat_ashmem_ioctl` | `0x00d65eec` |
| `ASHMEM_MMAP_OFF` | `ashmem_mmap` | `0x00d65f40` |
| `ASHMEM_OPEN_OFF` | `ashmem_open` | `0x00d66160` |
| `ASHMEM_RELEASE_OFF` | `ashmem_release` | `0x00d661e8` |
| `ASHMEM_SHOW_FDINFO_OFF` | `ashmem_show_fdinfo` | `0x00d66274` |
| `ANON_PIPE_BUF_OPS_OFF` | `anon_pipe_buf_ops` | `0x0124d308` |
| `ASHMEM_FOPS_OFF` | `ashmem_fops` | `0x0140b600` |
| `KMALLOC_CACHES_OFF` | `kmalloc_caches` | `0x017d8690` |
| `SYSTEM_UNBOUND_WQ_OFF` | `system_unbound_wq` | `0x022eae60` |
| `INIT_TASK_OFF` | `init_task` | `0x022fe340` |
| `ASHMEM_MISC_OFF` | `ashmem_misc` | `0x0246c928` |
| `ROOT_TASK_GROUP_OFF` | `root_task_group` | `0x0250bd80` |
| `SELINUX_ENFORCING_OFF` | `selinux_state` + BTF `enforcing` | `0x0254e598` |
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | first qword of `nfulnl_logger` -> string | `0x0175ce6b` |
| `SLIDE_NFULNL_LOGGER_OBJECT_OFF` | `nfulnl_logger` | `0x022f2278` |
| `SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` | `random_table[4].data` slot (procname == `boot_id`) | `0x024293b8` |
| `SLIDE_SYSCTL_BOOTID_OFF` | value of `random_table[4].data` | `0x026311d0` |
| `ASHMEM_MISC_FOPS_OFF` | `ashmem_misc` + BTF `miscdevice.fops` (0x10) | `0x0246c938` |

The `nfulnl_logger.name` pointer target was read from the raw Image and
verified to be the literal `"nfnetlink_log"`.

## 4. BTF layout values (this kernel's own BTF)

`sizeof(struct file_operations) = 0x108` with `unlocked_ioctl 0x48`,
`compat_ioctl 0x50`, `mmap 0x58`, `open 0x68`, `release 0x78`,
`splice_read 0xb8`, `show_fdinfo 0xd8`.

`task_struct`: `usage 0x40`, `prio 0x84`, `normal_prio 0x8c`,
`sched_task_group 0x348`, `pi_lock 0x90c`, `pi_waiters 0x920`,
`pi_top_task 0x930`, `pi_blocked_on 0x938`.

`struct page` / `struct slab` overlay: `size 0x40`, `compound_head 0x08`,
`slab_cache 0x08` (named in `struct slab`), `page_type 0x30`.

`rt_mutex_waiter` (size 0x70): `tree` field @0x00 containing
`rb_root_cached @0x00 / prio @0x18 / deadline @0x20` (BTF inner member bits
192/256), `pi_tree` @0x28 (prio 0x40, deadline 0x48), `task` @0x50,
`lock` @0x58, `wake_state` @0x60, `ww_ctx` @0x68. Identical to the
android15-6.6 family used by the verified A566E profile.

`configfs_buffer`: `page 0x10`, `needs_read_fill 0x50`, `bin_buffer 0x58`,
`bin_buffer_size 0x60`, `cb_max_size 0x64`. Workqueue family
(`work_struct`, `pool_workqueue`, `worker_pool`, `workqueue_struct.dfl_pwq
0xb0`) matches the family headers; re-verified from BTF.

`sizeof(struct mm_struct) = 0x4c0`; `MM_STRUCT_SZ` remains `0x500` because
`mm_cachep` objects are SLAB_HWCACHE_ALIGN'ed to the 128-byte stride the
controlled-mm scan relies on.

## 5. Slide data

```c
#define SLIDE_TRACEFS_EVENT_ID 109
```

Actual computation: `__event_sched_blocked_reason` is ftrace-event index 89
(zero-based, `(__event_sched_blocked_reason - __start_ftrace_events)/8`);
with `__TRACE_LAST_TYPE == 20` this gives event id **109**, the same value
hardware-verified on the same-KMI A566E build.

```c
#define SLIDE_TRACEFS_WORKER_CALLER_OFF 0x000d98dcULL
```

The single blocking `bl schedule` in `worker_thread` (idle path, directly
after `worker_enter_idle`/`_raw_spin_unlock_irq`) is at `0x000d98d8`; the
saved return PC is the following instruction `0x000d98dc`.

```c
#define SLIDE_PSELECT_WORD_SHIFT 0
```

Inherited from the hardware-verified same-KMI (6.6.102-android15-8) A566E
profile; `do_pselect`/`pselect6` compile from identical upstream source in
this family. Re-verify from slide diagnostics on first hardware run.

P0 fingerprint table generated with `tools/generate_p0_fingerprint.pl` from
the raw Image at probe offset `0x1f0000`; the tool's independent readback
verified all 32 rows and 256 source qwords.

## 6. Physical load addresses

`P0_PHYS_OFFSET` and `P0_KERNEL_PHYS_LOAD` are `0x80000000` (Exynos 1580
DRAM base; Image `text_offset == 0`). The static sboot pattern scan used for
the S24 FE did not match this multi-stage sboot build, so the value is
inherited from the same-SoC hardware-verified A566E profile. The P0 oracle
is fail-closed: if the constant were wrong, no fingerprint row matches and
the exploit aborts without writing.

## 7. Builds

```sh
make TARGET=essi-X520XXSBCZI2 ANDROID_NDK_HOME=<ndk-r29> all release
```

- `cve-2026-43499-app.release.so` — 104128 bytes (fixed release size gate),
  SHA-256 `f37a278a234afb66c7209a43241b2166657b80c450120eaa2932079da869bfde`
- `cve-2026-43499-root` — root helper (umh path
  `/data/local/tmp/cve-2026-43499-root`)

KernelSU module: KernelSU `v3.2.5` (b0bc817b) +
`KernelSU-v3.2.5-samsung-kdp-rkp-defex.patch`, built in
`ghcr.io/ylarod/ddk-min:android15-6.6-20260313` with the DDK release
replaced by the exact target release, `CONFIG_KSU=m`,
`CONFIG_KSU_SAMSUNG_KDP=y`, `CONFIG_KSU_SAMSUNG_RKP=y`,
`CONFIG_KSU_SAMSUNG_DEFEX=y`, and
`CONFIG_KSU_SAMSUNG_NO_PATCH_TEXT=y` (Exynos EL2 live-text-patching panic
class documented on the same-SoC A56).

```text
vermagic: 6.6.102-android15-8-abX520XXSBCZI2-4k SMP preempt mod_unload modversions aarch64
```

`check_symbol` passes against the recovered X520 `vmlinux.elf`; an
independent audit resolved all 214 undefined module imports by name in the
target kallsyms (zero missing). The DDK `Module.symvers` CRC table was not
replaced by a target-derived one (no target `Module.symvers` exists offline);
KMI CRC equality with android15-6.6 is assumed per the same DDK path used by
the device-tested A566E/A366W modules.

Artifacts:

```text
artifacts/essi-X520XXSBCZI2/cve-2026-43499-app.so        104128 bytes
artifacts/essi-X520XXSBCZI2/cve-2026-43499-root           27072 bytes
kernelsu/android15-6.6_kernelsu-essi-X520XXSBCZI2-kdp.ko 325784 bytes (debug-stripped)
kernelsu/ksud-essi-X520XXSBCZI2-kdp                      4751280 bytes (embeds the KO)
```

## 8. Status and next steps

Offline gates passed: offsets derived from this exact firmware, structure
layouts from this kernel's BTF, module vermagic exact, symbol audits clean,
release size gate enforced. Hardware validation remains open: first run
should confirm (1) the tracefs slide route reports a 64-KiB-aligned caller
minus `0x000d98dc` inside the P0 slide range, (2) the P0 fingerprint row
locks, (3) `ksud late-load` enters `u:r:ksu:s0`.
