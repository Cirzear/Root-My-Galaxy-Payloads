# SM-S9260 / S9260ZCU5DZDP adaptation notes

This target was derived from the supplied CHC factory firmware and is intended
only for the exact `SM-S9260` / `S9260ZCU5DZDP` build. The target header,
page-zero fingerprint, payload, KernelSU module, and embedded-module `ksud`
pass the offline consistency checks described below. Hardware diagnostics from
2026-08-10 reached page-zero preparation with an earlier production profile,
but did not acquire root. The current physical-P0-oracle revision therefore
remains hardware-debugging-in-progress until it passes a clean-boot retest.

## Firmware identity and provenance

- Product/device: `SM-S9260`, `e2q`, China `CHC`
- AP/CSC: `S9260ZCU5DZDP` / `S9260CHC5DZDP`
- Runtime/system fingerprint: `samsung/e2qzcx/e2q:16/BP4A.251205.006/S9260ZCU5DZDP:user/release-keys`
- OTA metadata fingerprint: `samsung/e2qzcx/qssi_64:16/BP4A.251205.006/S9260ZCU5DZDP:user/test-keys`
- SDK/security patch: `36` / `2026-02-05`
- Kernel release: `6.1.145-android14-11-3254743-abS9260ZCU5DZDP`
- Kernel build banner: `#1 SMP PREEMPT Fri Apr 24 01:49:38 UTC 2026`

The payload header uses the runtime/system `user/release-keys` fingerprint
reported by the connected device. The OTA tooling value is retained as
provenance only. The value reported by `adb shell getprop ro.build.fingerprint`
must still be checked before each hardware run. Do not bypass the application's
model and kernel checks if the live values differ.

| Supplied or extracted input | Size | SHA-256 |
| --- | ---: | --- |
| Factory ZIP | 14,594,955,441 | `349f05e74de1f817ab0e1ba43a9587fcd5b4f18c5f265bba1db70626663dac1e` |
| AP tar.md5 | 16,769,198,203 | `fbbc88a368922379465dc54c563b100fb6b601bec5456f4e09aad0a732150edb` |
| BL tar.md5 | 103,823,473 | `bc7372707bac8fbc8846594d30dc722d15cf01ebf7ccf23841771cf47d0b758f` |
| `boot.img.lz4` | 22,111,139 | `301adceb7725808ff5135c99ec6a6a4384542e95e347a512c9671fdd688631d2` |
| Decompressed `boot.img` | 100,663,296 | `a9d2910332527240040e73926926a73c8fe493231032950ab50e8a283f607e23` |
| Raw kernel Image | 38,005,248 | `c21ba3a191ac389c3e33b0e85a236bb905712f8615a963e134d51537d1c0b98d` |
| Recovered `vmlinux.elf` | 43,072,392 | `e6a1fb2f17287ee3c875fe22fb415c6134a94ba5c2c1538a482917a34e4d50f1` |
| `abl.elf` | 2,441,528 | `6e895f0b81e209a47444105359f4c3a32bab963e5a6b68d372203ed32df26cd9` |

## Recovered target constants

Offsets below are relative to `KIMAGE_TEXT_BASE = 0xffffffc008000000`.

| Symbol/object | Offset |
| --- | ---: |
| `call_usermodehelper_exec_work` | `0x000d39cc` |
| `worker_thread` return after `schedule()` | `0x000db1a0` |
| `noop_llseek` | `0x003a14e4` |
| `generic_file_splice_read` | `0x003ef340` |
| `configfs_read_iter` | `0x004712a4` |
| `configfs_bin_write_iter` | `0x004717d4` |
| `ashmem_ioctl` / `compat_ioctl` | `0x00d3a314` / `0x00d3ac4c` |
| `ashmem_mmap` / `open` / `release` | `0x00d3aca4` / `0x00d3aed0` / `0x00d3af58` |
| `ashmem_show_fdinfo` | `0x00d3b078` |
| `anon_pipe_buf_ops` | `0x01219d90` |
| `ashmem_fops` | `0x013d1140` |
| `kmalloc_caches` | `0x0176cbb8` |
| `system_unbound_wq` | `0x0223ae60` |
| `loggers` / `nfulnl_logger` | `0x02242968` / `0x02242a20` |
| `init_task` | `0x0224f8c0` |
| `random_table` / boot-ID pointer slot | `0x023761e8` / `0x023762f0` |
| `ashmem_miscs.fops` | `0x023bb5b0` |
| `root_task_group` | `0x0244cd80` |
| `selinux_state.enforcing` | `0x02521588` |
| `sysctl_bootid` | `0x026046e8` |

The recovered BTF blob is the validated raw-Image interval
`[0x180b384, 0x1dbfdc6)`, size `5,982,786`, SHA-256
`9b26dd8cf87273da0141b42d93f30a732cef9f9660550a17eb1eddfaab5f3db2`.
It was used to verify the `file_operations`, `task_struct`, RT-mutex,
configfs, workqueue, page/slab, and `skb_shared_info` layouts encoded by the
target header. The configuration confirms 4 KiB pages, MTE, hardware-tag
KASAN, BTF, and module versioning.

The BTF-derived `mm_struct` size for this exact kernel is `0x3c0`; the
original app profile incorrectly used `0x500`, which made KernelSnitch scan
the wrong object stride. The target profile now uses `MM_STRUCT_SZ=0x3c0`
and `KERNELSNITCH_MTE_ENABLED=1`, matching the BTF/configuration evidence.

The ARM64 Image header reports `text_offset = 0`, `image_size = 0x26f0000`,
and flags `0xa`. The ABL LinuxLoader contains the exact tuple
`[0x80000, 0x5600000, 0x3c00000, 0x8000]`; together with the vendor boot
`gunyah_hyp_region@80000000` data this yields `P0_PHYS_OFFSET = 0x80000000`
and `P0_KERNEL_PHYS_LOAD = 0x80080000`.

## Generated artifacts

| Artifact | Size | SHA-256 |
| --- | ---: | --- |
| `artifacts/e2q-S9260ZCU5DZDP/cve-2026-43499-app.so` | 104,128 | `83b77dbdf35a00d2098bebd75e05c1915f9a5919bd847637673bb180d3b10678` |
| `kernelsu/android14-6.1_kernelsu-e2q-S9260ZCU5DZDP-kdp.ko` | 400,152 | `ce9140b70f83edfd1d61ed9c7b8a9a6c47a4b6f9f546d8bd596a03b8a5c97bed` |
| `kernelsu/ksud-e2q-S9260ZCU5DZDP-kdp` | 4,780,112 | `76b5c52f955a50f244941f24bf9bd4a62040e9da5c64fffa8015b40bf8d221a2` |

The current physical-P0-oracle payload was compiled for AArch64 with Android
NDK r28c and padded to its published feed size. The KernelSU pair is based on KernelSU `v3.2.5` commit
`b0bc817b4e966aa6aa830834eaf6ef765d821d40` plus the repository's Samsung
KDP/RKP/DEFEX patch. The module carries the exact target `vermagic`; all 209
undefined module symbols were found in the recovered target symbol table,
with zero missing symbols, zero version entries, and zero CRC mismatches. The
`ksud` binary embeds that exact module and reports version code `32525`.

The module is a mechanically specialized same-KMI E3Q build, not a module
compiled from a released Samsung E2Q kernel source tree. The exact vermagic
and symbol audit reduce obvious incompatibility risk but cannot prove that
Samsung KDP/RKP behavior or runtime structure use is identical. A clean-boot
hardware test is therefore required before this target can be called
device-verified.

## Hardware diagnostic status

The 2026-08-10 application logs came from the older
`app-production74-slide8-fops8` profile. Across independent attempts it found
collision candidates and once recovered an `mm_struct` plus a page-zero pipe
oracle. The following fresh kernel-page preparation repeatedly returned no
usable nonzero source pointer, so the run stopped before the fops overwrite,
bootstrap root, or KernelSU late-load stages.

The published payload uses the exact DZDP offsets and the generated 32-slide
fingerprint table. The root-first path now fails closed at every proof step:
P0 leak misses are discarded before reclaim data is used, physical read/write
proofs are checked before the next kernel write, and root success is reported
only after the uid-0 socket check.

The previous retry5 run produced three `SYSTEM_LAST_KMSG_*_KP` records; the
latest was at `2026-08-15 17:25:03` and followed repeated physical P0 attempts.
After the single-attempt hardening, the device stayed connected with no new
kernel-panic record through APK installation and application launch.

The subsequent 17:52 run exposed a second retry source in the Android wrapper:
`InstallViewModel` explicitly exported `EXPLOIT_ATTEMPTS=24`, overriding the
native target default. Its log confirmed `attempts=24` and the device recorded
`SYSTEM_LAST_KMSG_15_20260815_175527_KP`. The wrapper now exports
`EXPLOIT_ATTEMPTS=1` as well.

The 18:57, 19:02, and 19:21 hardware attempts produced new
`SYSTEM_LAST_KMSG_16_20260815_185751_KP` and
`SYSTEM_LAST_KMSG_17_20260815_190223_KP` and
`SYSTEM_LAST_KMSG_18_20260815_192103_KP` records. The corrected profile
reached `p0 pipe oracle prepared` and then entered KernelSnitch kernel-page
preparation; the run stopped during that stage before any physical P0 write.
The in-range pipe candidate filter did not address the fault, so the current APK has a hard
`APP_P0_PREPARE_ONLY` circuit breaker during that diagnostic run. The current
root-first build is separate from the prepare-only artifact and is not claimed
as hardware-rooted until a clean execution reaches the uid-0 check.

On 2026-08-15, the physical profile caused another hard reset while the device
was at boot uptime 83 seconds. The new prepare-only diagnostic completed after
171 seconds, reached a valid pipe page once, exhausted eight kernel-page
preparation attempts, and returned status `0xff00` without another reset. The
device remained online afterward. The safe artifact is
`build/e2q-S9260ZCU5DZDP/cve-2026-43499-app.p0-prepare-only-diagnostic.so`.

The latest failed run reached the P0 preparation retry loop and the device then
dropped off ADB, consistent with the reboot observed during that run. Its direct cause
before the disconnect was a P0
KernelSnitch miss: all 32 attempts found collision sets but no `mm_struct`
candidate in the then-active window
`[ffffff8000000000, ffffff8600000000)`, so the child returned status `255`
before physical preparation, any kernel write, or the uid-0 check.

The next root-first artifact keeps all valid mm-slab object indexes `0..33`
and keeps the P0 pipe-page window bounded at
`[ffffff8000000000, ffffff8f80000000)` so each fresh-session budget is
not consumed by an unnecessarily large scan. This includes the previously observed
pipe-page candidate near `ffffff89252b0000` while still excluding the known
high-end fault region near `ffffff8ffd...`. The subsequent run did prepare a
valid P0 oracle, then the slide kernel-page stage still searched only object
indexes `27..30` and returned no candidate; the slide-page search now uses the
complete valid slab range `0..33`.
at `ffffff80b3a78000`, then failed in kernel-page preparation because the fops
search still excluded slab object indexes below `24`. The fops search is now expanded to `0..33`, matching the observed candidates at indexes `12`, `19`, and `21`. The P0 threshold remains at the default `10`, while the kernel-page threshold is `6`. The native supervisor now enforces sixteen fresh processes with 600/660-second P0/overall budgets because the wrapper was still exporting 1/240/300. The kernel-page and reclaim upper bound is now the full direct-map end `ffffff9000000000`; validation remains responsible for rejecting non-source candidates. Kernel-page collision threshold is set to `6` while P0 retains the default `10`; kernel-page setup uses two collision anchors to widen the candidate set before source-pointer validation. The kernel-page and reclaim filters now use the same bounded
`ffffff8f80000000` upper limit so the observed `ffffff89252b0000` candidate
is not rejected before validation.
The fresh2x16 retry profile limits each exploit process to two P0 page-preparation
passes, adds a 500 ms quiet interval between failed passes, and lets the native
supervisor use sixteen fresh processes. This keeps a failed P0 session from repeatedly
reusing the same allocator state while retaining multiple clean-boot opportunities.
The optional pipe-shaping path is skipped when its descriptor arrays are not
allocated, avoiding an accidental `F_SETPIPE_SZ` call on fd 0. The kernel-page
and reclaim filters use the same upper bound. The APK embedded asset and the
app-private payload were updated and verified at SHA-256
`83b77dbdf35a00d2098bebd75e05c1915f9a5919bd847637673bb180d3b10678`.

## First hardware validation boundary

Before executing the payload, capture these values after a clean reboot:

```sh
adb shell getprop ro.product.model
adb shell getprop ro.build.fingerprint
adb shell uname -r
adb shell uname -a
```

Proceed only when the model is `SM-S9260`, the build is DZDP, and the kernel
release exactly matches the value above. Preserve the full application log.
Do not substitute the regular `CSC` package for `HOME_CSC` or flash any
firmware as part of this payload test; firmware flashing is outside this
adaptation and is unnecessary for the first validation.

## 2026-08-16 CVE/P0 and gate boundary

The CVE-2026-43499 hypothesis is relevant to the P0 stage: repeated fresh
allocator sessions are probabilistic, and a clean P0-only run hit the pipe
oracle on its second P0 pass. The current e2q root trace then reproduced the
same mechanism in child 2, reached a valid kernel-page candidate, and passed
the pselect readiness checks. It did not pass the physical root-gate proof:
`gate_hits=0 changed=0`. The run produced no uid-0 proof and was followed by a
delayed device reboot. The complete run record is
`build/e2q-S9260ZCU5DZDP/root32trace-run-summary.txt` and the final UI capture
is `build/e2q-S9260ZCU5DZDP/root32trace-final-ui.xml`.

The immediate safety change is a separately built diagnostic artifact,
`build/e2q-S9260ZCU5DZDP/root32-safe-diag-signed.apk`, which keeps the P0 and
kernel-page preparation but compiles with
`APP_P0_DIAGNOSTIC_THEN_ROOT=0`; it therefore stops before the physical gate,
probe, restore, and root writes. The physical route remains disabled until
the e2q bank geometry and gate verification agree.
