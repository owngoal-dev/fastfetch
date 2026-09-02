# fastfetch — Agent Notes

[fastfetch](https://github.com/fastfetch-cli/fastfetch) — the neofetch-like
system information tool — built for jailbroken iOS 15+ and installed as
`fastfetch`, for both **roothide** and **rootless** bootstraps.

This repository holds **no application source**. It fetches fastfetch at a
pinned commit, applies `patches/`, cross-compiles with CMake against the
iPhoneOS SDK, and packages. Everything runs through `scripts/`, so CI and a
local checkout execute the same code.

## Hard rules

- **Not a fork.** Never vendor fastfetch source here. Every change to it is a
  patch in `patches/`, applied by `scripts/prepare-source.sh` to a fresh
  checkout of `UPSTREAM_REF`. Keep patches small and single-purpose.
- **`UPSTREAM_REF` is a full commit sha**, not a branch. Bump with
  `make bump-upstream REF=…`.
- **One arm64 binary backs both packages.** arm64 runs on every arm64e device
  and the reverse is not true. The two `.deb` architectures name a *bootstrap
  layout*, not a CPU: `iphoneos-arm64` is rootless, `iphoneos-arm64e` is
  roothide. Never build an arm64e slice for the "arm64e" package. Aggregate
  release targets build the payload once, package it twice, and checksum only
  the two exact current-version outputs.
- **Never hardcode a bootstrap path in patched source.** The binary derives
  the bootstrap from its own executable path (`<bootstrap>/usr/bin/fastfetch`)
  and probes fixed candidates only as a fallback. That is how it finds
  `<bootstrap>/etc/fastfetch`, its presets, and dpkg's status file on
  rootless, RootHide and rootful alike.
- **Versions live in `configuration/version.txt` only.** `X.Y.Z` must equal
  upstream's `project(fastfetch VERSION …)`; `prepare-source.sh` refuses a
  mismatch. `X.Y.Z-N` is a packaging-only respin.
- **Do not link libvroot.** This is a plain C binary talking to libSystem; the
  path derivation above replaces vroot's rewriting for the few paths that
  matter.
- **`CLAUDE.md` is a symlink to `AGENTS.md`**, never a file of its own. One
  set of notes, two names; `make check` enforces it.
- **Review for sensitive information before anything is uploaded or
  published.** `scripts/check-sensitive.sh` scans tracked files, the staged
  package tree and the finished `.deb`s for credentials, private keys, home
  and scratch paths, device identifiers, IP addresses and e-mail addresses.
  `make check`, `package-deb.sh` and the Release workflow all run it and
  stop on a hit. A deliberate public value goes on its allowlist; a rule is
  never loosened.

## How the iOS port works

Upstream has no iOS target; `if(APPLE)` means macOS. The port is four patches:

- `0001-ios-cmake-target` — inside the Apple branch, `if(IOS)` swaps out the
  detectors that need AppKit, CoreWLAN, IOBluetooth, the CoreAudio HAL,
  CoreDisplay, KextManager or AppleScript for the existing `_nosupport`
  fallbacks, adds the iOS files, and links only frameworks the iOS SDK has.
- `0002-ios-detection-sources` — new files: `os_ios.m` (iOS/iPadOS name,
  version, build; `idLike = macos` so the Apple logo is picked),
  `displayserver_ios.c` (panel size, scale and PPI from MobileGestalt's
  unprotected `main-screen-*` keys; WM reported as SpringBoard),
  `packages_ios.c` (dpkg count from the bootstrap's status file),
  `opengl_ios.c` (unsupported), `common/apple/chip_ios.c` (SoC name from
  the device tree's `chosen/chip-id`, e.g. 0x8027 → "Apple A12Z Bionic",
  via upstream's Asahi code table extended with the A-series).
- `0003-ios-sdk-guards` — `TARGET_OS_IPHONE` / `__has_include` guards in
  shared Apple files (AppleScript, OpenGL/OpenCL headers) plus the upstream
  `sound_nosupport.c` signature fix.
- `0004-ios-bootstrap-config-dir` — `<bootstrap>/etc/` added to the config
  search path, derived from the executable path.
- `0005-ios-hardware-names` — the kernel's CPU brand string on iOS is the
  literal "Apple processor", `MTLCreateSystemDefaultDevice()` returns nil
  for a command-line process, and there is no `IOAccelerator` class. CPU
  and GPU are named from the chip id; the GPU entry is the `AGXAccelerator`
  IORegistry service (core count from `GPUConfigurationVariable/num_cores`,
  usage and memory from `PerformanceStatistics`, clock from `pmgr` like on
  macOS). Chassis is Tablet/Handset by `hw.machine`. KextManager and Metal
  are compiled out on iOS.

Everything else Apple-flavoured (memory, battery, power adapter, disks,
network, processes, host name via `IODeviceTree:/product`, terminal/shell
walk) compiles and links unchanged — the iPhoneOS SDK just omits the headers.
`build-ios.sh`
symlinks the missing headers (libproc, routing sysctls, most of IOKit) from
the macOS SDK into a shim include dir. Only headers the iOS SDK lacks go in
the shim, so the iOS SDK's own declarations keep winning.

Two SDK traps, both handled in `build-ios.sh`:

- The iOS 27 SDK declares `pipe2`, so CMake's `check_function_exists` says
  yes, but no device before iOS 27 has it. `HAVE_PIPE2=0` is pinned.
- Any C API newer than `MIN_IOS` is weak-linked and NULL on older devices. The
  build script lists weak imports; every one must be null-checked in source
  (currently only `VTRegisterSupplementalVideoDecoderIfAvailable`, which is).
  Objective-C methods newer than `MIN_IOS` are worse: no weak symbol, just an
  unrecognized-selector crash. Guard them with `@available`, never by
  silencing `-Wunguarded-availability-new`.

Verified on device: iPad Pro 11" (2nd gen, A12Z), iPadOS 18.5, rootless
Dopamine. The roothide package is the same binary in the other layout and
has not been installed on a roothide device yet.

Detectors that are intentionally `nosupport` on iOS: Wi-Fi, Bluetooth,
sound, font, wallpaper, media, brightness, camera, kernel modules, OpenGL,
OpenCL. Adding one back means an iOS implementation, not a macOS framework.

## Layout

```
configuration/upstream.env   pinned ref, program name, iOS floor
configuration/version.txt    package version
patches/NNNN-*.patch         applied in sorted order to a pristine checkout
packaging/DEBIAN/control     control template (@PLACEHOLDER@ substituted)
packaging/fastfetch.entitlements  what the signed binary carries, and why
packaging/release-notes.md   GitHub Release body template
scripts/prepare-source.sh    fetch + patch (idempotent, stamped)
scripts/build-ios.sh         SDK shim + cmake + verify Mach-O + install payload
scripts/package-deb.sh       stage + ldid + dpkg-deb + verify
scripts/install-device.sh    install over SSH and smoke-test (dev only)
build/                       everything generated; not source
```

## Build & verify

- `make check` — script syntax, config sanity, patch set, packaging inputs
- `make source` — fetch + patch; fails loudly if a patch no longer applies
- `make build` — cross-compile and verify the Mach-O is iOS
- `make debs` — both packages plus `SHA256SUMS`; what CI releases
- `make install` — install on an attached device and run `--version` and a
  full `fastfetch --pipe`. Over USB: `iproxy 4422:2222 &`

Test by installing, never by copying a binary onto `/var/mobile`. A copied
binary runs with its entitlements ignored (trustcache never saw it).

## The OwnGoalPackages contract

Same as kk: a non-draft, non-prerelease tag `vX.Y.Z`; assets whose names end
in `iphoneos-arm64.deb` / `iphoneos-arm64e.deb`; a `SHA256SUMS` of bare names.

`Follow upstream` runs every day at 00:00 UTC: pin to the newest stable
`fastfetch-cli/fastfetch` `X.Y.Z` release, `make source` to prove `patches/`
still apply, then commit and tag `vX.Y.Z` as `bot <bot@owngoal.dev>`.
`Release` builds that tag. OwnGoalPackages fetches it at 04:00 UTC.
