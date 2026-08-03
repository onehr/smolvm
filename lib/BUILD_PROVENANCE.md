# libkrun binary provenance

*Recorded 2026-08-03. Read this before rebuilding, replacing, or trusting any dylib in this directory.*

## Why this file exists

Until 2026-08-03 the only binary implementing the nex-env filesystem effect gate existed as an
**uncommitted working-tree file**. `git status` showed `M lib/libkrun.dylib`, while the LFS pointer at
HEAD referenced a *different* build that does not contain the gate at all. The branch holding the
corresponding source (`fs-effect-policy`) had no upstream and did not exist on the remote. A disk failure
would have destroyed both the binary and its source, and the toolchain was unpinned so neither could be
reproduced.

## The binaries

All hashes are sha256 of the file as it existed on 2026-08-03.

| File | Size | sha256 | FS effect gate | S6 epoch fence | GPU (virgl/epoxy) |
|---|---|---|---|---|---|
| `libkrun.dylib` **← the one that works** | 5,927,056 | `a1e64d59821cc83f0631d333a33f4c0ecae905ef87be5415d60ef708e4a1e383` | **yes** | no | no |
| `libkrun.dylib.pre-s6-bak` | 5,927,056 | `a1e64d59…` *(byte-identical to the above)* | yes | no | no |
| `libkrun.dylib.june6.bak` | 6,070,704 | `02ed2bc2914e613385085d6f7345760c0746f3d87cfdf9701ffa2c884bde22b6` | **no** | no | **yes** |
| `libkrun.dylib.gpu-bak` | 6,070,704 | `02ed2bc2…` *(byte-identical to june6.bak)* | no | no | yes |
| `../libkrun/target/release/libkrun.dylib` | 5,940,832 | `aebeeaa47e7a66e469a75b8d25bd1aaefaf4badd9e1113951819bed948249e9a` | yes | **yes** | no |
| `libkrunfw.5.dylib` | 13,315,472 | `d1d7091c8067c9219f793b18dc673b47688966f12f947f9ec083c518498cd0d2` | n/a | n/a | n/a |

Presence of the FS gate was determined by `strings <file> | grep NEX_FS_AUDIT_LOG`; the S6 epoch fence by
`grep NEX_FS_EPOCH`; GPU linkage by `otool -L | grep -i 'virglrenderer\|epoxy'`.

**The gated demo requires `libkrun.dylib` = `a1e64d59…`.** Any other binary in this directory silently
disables filesystem gating: the VM still boots and the demo still appears to pass, but no FS effect is
mediated and no FS audit journal is produced.

`a1e64d59…` links only Hypervisor.framework, libiconv and libSystem — no GPU libraries.

## Correction: the "GPU broke the build" explanation is false

`lessons.md:6618` and `production_gaps.md:161` record the S6 fork-liveness regression as blocked on a
libkrun GPU/rutabaga build-environment problem. The hashes above refute this:

- the **working** FS-gate build (`a1e64d59…`) has **no** GPU links;
- the **regressed** S6 build (`aebeeaa4…`) also has **no** GPU links;
- only `june6.bak` / `gpu-bak` (`02ed2bc2…`) carry virglrenderer/libepoxy — and neither contains the FS
  gate, so neither has ever been the deployed gated binary.

GPU-vs-non-GPU therefore cannot be the difference. Both the known-good and the regressed builds are
non-GPU, and they differ by 13,776 bytes whose only new strings are S6's.

**The remaining prime suspect is S6's own code.** `epoch_guard`
(`libkrun/src/devices/src/virtio/fs/macos/passthrough.rs:935`) calls `read_epoch_file` (`:952`), which
performs a synchronous `std::fs::read_to_string` on **every write-capable `open`** (`:1789`). The
virtio-fs daemon is single-threaded, so this puts a blocking host filesystem read on the FUSE hot path
during runner boot — a plausible cause of the fork's runner never coming online. This is a hypothesis, not
a conclusion; it is settled by the bisect described below.

## Reproducibility: preserved, not reproducible

`rust-toolchain.toml` was `channel = "stable"` (floating) when `a1e64d59…` was built in June 2026. The
current toolchain is `rustc 1.94.0 (4a4ef493e 2026-03-02)`, which is **not** necessarily the compiler that
produced the preserved binary, and the original version was never recorded.

The channel is now pinned so that future builds are at least reproducible *going forward*. Understand the
consequence: **`a1e64d59…` can be preserved but cannot be recreated.** Treat it as an artifact, not as
build output.

## The bisect that unblocks Rust work

Any change to the FS gate, the TSI allowlist, or the epoch fence requires a rebuild, and every rebuild
will be blamed for the unexplained S6 regression until this is run:

1. Rebuild libkrun at the pre-S6 commit with the now-pinned toolchain.
2. Run the live gated-VM suite (`fs_effect_gate_real_test.go`, `fs_effect_record_real_test.go`,
   `hermes_full_demo_test.go`) and the fork end-to-end.

- **Pass** ⇒ the regression is S6's code, most likely `read_epoch_file` on the hot path. Fix by caching the
  epoch in an atomic refreshed by a watcher rather than reading the file per `open`.
- **Fail** ⇒ the regression is toolchain drift. The known-good binary cannot be reproduced at all, only
  preserved, and every Rust change is blocked until the build is made reproducible.

Timebox this to one day. An ambiguous result counts as a failure for planning purposes.

### RESULT, 2026-08-03: PASS. Rust work is not blocked.

Built libkrun at the pre-S6 commit `b9e3745` with the now-pinned rustc 1.94.0 and `BLK=1 NET=1`
(matching the deployed binary's shape, **not** the `GPU=1` in the repo's `build-libkrun` task, which
has never been what shipped). Result: `23b682d4...`, 5,943,472 bytes — FS gate present, no S6 epoch
strings, no GPU links.

It does **not** byte-reproduce `a1e64d59...` (5,927,056 bytes), which is expected: a different
compiler. That is not the question the bisect asks.

Swapped it in and ran the full fork -> probe -> pause live test, which exercises the exact symptom S6
regressed ("the fork's runner never came online"). **It passed**, including *"the forked world booted
(a live, separate gated agent)"*, the original being paused only after the fork's gate probe returned
OK, and the serving epoch advancing 0 -> 1. The deployed binary was restored afterwards; `lib/` is
untouched.

**Conclusions.**

1. **Toolchain drift is not the cause.** A fresh build with today's pinned compiler, at the commit
   that produced the known-good binary, yields a working binary with live forks.
2. **The S6 fork-liveness regression is S6's own code.** The remaining suspect stands:
   `epoch_guard` (`passthrough.rs:935`) calls `read_epoch_file` (`:952`), a synchronous
   `std::fs::read_to_string`, on **every write-capable `open`** (`:1789`) — a blocking host filesystem
   read on the single-threaded FUSE hot path, during runner boot. The fix is to read the epoch once
   into an atomic refreshed by a watcher, not per-`open`.
3. **Everything downstream is unblocked**: the DAX `setupmapping` guard, `(ip, port, proto)` TSI
   granularity, the signed egress-denial journal, FS shadow mode, and the Linux port were all
   scheduled behind an unexplained build failure that does not exist.

A rebuilt binary is now known to be viable and reproducible, which the preserved one is not. Promoting
the rebuild to the deployed binary is reasonable but should clear the full live suite first, not just
the fork test.

## Off-repo backup

A copy of every binary above, plus a `git bundle --all` of the whole repository (which preserves the
otherwise-unpushed `fs-effect-policy` branch), was written to `~/Backups/smolvm-substrate-20260803/` with a
`SHA256SUMS.txt`. **That directory is on the same machine and is not a backup until it is copied off it.**
