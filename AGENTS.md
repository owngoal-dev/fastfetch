# fastfetch — Agent Notes

[fastfetch](https://github.com/fastfetch-cli/fastfetch) — the neofetch-like
system information tool — built for jailbroken iOS 15+ and installed as
`fastfetch`, for both **roothide** and **rootless** bootstraps.

This repository holds **no application source**. It fetches fastfetch at a
pinned commit, applies `patches/`, cross-compiles with CMake against the
iPhoneOS SDK, and packages. Everything runs through `Scripts/`, so CI and a
local checkout execute the same code.

## Hard rules

- **Not a fork.** Never vendor fastfetch source here. Every change to it is a
  patch in `patches/`, applied by `Scripts/prepare-source.sh` to a fresh
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
- **Versions live in `Configuration/version.txt` only.** `X.Y.Z` must equal
  upstream's `project(fastfetch VERSION …)`; `prepare-source.sh` refuses a
  mismatch. `X.Y.Z-N` is a packaging-only respin.
- **Do not link libvroot.** This is a plain C binary talking to libSystem; the
  path derivation above replaces vroot's rewriting for the few paths that
  matter.

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
  `opengl_ios.c` (unsupported).
- `0003-ios-sdk-guards` — `TARGET_OS_IPHONE` / `__has_include` guards in
  shared Apple files (AppleScript, OpenGL/OpenCL headers, Metal's
  `MTLCopyAllDevices` which is iOS 18+, KextManager) plus the upstream
  `sound_nosupport.c` signature fix.
- `0004-ios-bootstrap-config-dir` — `<bootstrap>/etc/` added to the config
  search path, derived from the executable path.

Everything else Apple-flavoured (CPU, GPU via IOKit + Metal, memory, battery,
power adapter, disks, network, processes, terminal/shell walk) compiles and
links unchanged — the iPhoneOS SDK just omits the headers. `build-ios.sh`
symlinks the missing headers (libproc, routing sysctls, most of IOKit) from
the macOS SDK into a shim include dir. Only headers the iOS SDK lacks go in
the shim, so the iOS SDK's own declarations keep winning.

Two SDK traps, both handled in `build-ios.sh`:

- The iOS 27 SDK declares `pipe2`, so CMake's `check_function_exists` says
  yes, but no device before iOS 27 has it. `HAVE_PIPE2=0` is pinned.
- Any API newer than `MIN_IOS` is weak-linked and NULL on older devices. The
  build script lists weak imports; every one must be null-checked in source
  (currently only `VTRegisterSupplementalVideoDecoderIfAvailable`, which is).

Detectors that are intentionally `nosupport` on iOS: Wi-Fi, Bluetooth,
sound, font, wallpaper, media, brightness, camera, kernel modules, OpenGL,
OpenCL. Adding one back means an iOS implementation, not a macOS framework.

## Layout

```
Configuration/upstream.env   pinned ref, program name, iOS floor
Configuration/version.txt    package version
patches/NNNN-*.patch         applied in sorted order to a pristine checkout
Packaging/DEBIAN/control     control template (@PLACEHOLDER@ substituted)
Packaging/fastfetch.entitlements  what the signed binary carries, and why
Packaging/release-notes.md   GitHub Release body template
Scripts/prepare-source.sh    fetch + patch (idempotent, stamped)
Scripts/build-ios.sh         SDK shim + cmake + verify Mach-O + install payload
Scripts/package-deb.sh       stage + ldid + dpkg-deb + verify
Scripts/install-device.sh    install over SSH and smoke-test (dev only)
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

`Follow upstream` runs every Monday at 00:00 UTC: pin to the newest stable
`fastfetch-cli/fastfetch` `X.Y.Z` release, `make source` to prove `patches/`
still apply, then commit and tag `vX.Y.Z` as `bot <bot@owngoal.dev>`.
`Release` builds that tag. OwnGoalPackages fetches it at 04:00 UTC.
