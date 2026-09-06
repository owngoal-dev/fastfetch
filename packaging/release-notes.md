[fastfetch](https://github.com/fastfetch-cli/fastfetch) — the neofetch-like system information tool — built for jailbroken iOS and installed as `fastfetch`.

## Which one do I download?

The architecture field names the **bootstrap layout, not the CPU**. Both packages carry the same arm64 binary.

| your bootstrap | download |
| -------------- | -------- |
| rootless (Dopamine, palera1n rootless) | `@PACKAGE_ID@_@VERSION@_iphoneos-arm64.deb` |
| roothide (RootHide Dopamine) | `@PACKAGE_ID@_@VERSION@_iphoneos-arm64e.deb` |

Not sure? Ask the device: `dpkg --print-architecture`.

Requires **iOS @MIN_IOS_MAJOR@ or later**. Or add the [OwnGoal Studio repository](https://github.com/owngoal-dev/owngoal-packages) and let your package manager pick.

## Usage

Run `fastfetch` in a terminal on device. Your config goes in `~/.config/fastfetch/config.jsonc`; presets ship in `/usr/share/fastfetch/presets` (under the bootstrap prefix). Packages are counted from dpkg.

## About this build

Upstream [`fastfetch-cli/fastfetch@@UPSTREAM_SHORT@`](https://github.com/fastfetch-cli/fastfetch/commit/@UPSTREAM_REF@), plus the patches that port it to iOS: an iOS CMake target, iOS implementations of the OS, display, packages, CPU/GPU naming and chassis detectors, and no-op fallbacks for the macOS-only ones (Wi-Fi, Bluetooth, sound, fonts, wallpaper, media, brightness, camera, OpenGL). See [`patches/`](https://github.com/owngoal-dev/fastfetch/tree/@TAG@/patches).

Verify your download against `SHA256SUMS`.

**Full changelog**: https://github.com/owngoal-dev/fastfetch/commits/@TAG@

This packaging revision updates RootHide compatibility checks and signing.
CLI startup passes bootstrap paths to payloads that use the physical filesystem;
RootHide virtual-filesystem utilities retain their official import rewriting.
RootHide device validation is pending; a successful build is not a claim that
all interactive runtime paths have been tested.
