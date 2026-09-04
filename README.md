# fastfetch

[fastfetch](https://github.com/fastfetch-cli/fastfetch) — the neofetch-like
system information tool — built for jailbroken iOS and installed as
`fastfetch`. One arm64 build, packaged for both **roothide** and **rootless**
bootstraps.

## Install

From the [OwnGoal Studio repository](https://github.com/owngoal-dev/OwnGoalPackages),
or grab the `.deb` for your bootstrap from
[Releases](../../releases) and `dpkg -i` it:

| bootstrap | package architecture |
| --------- | -------------------- |
| rootless  | `iphoneos-arm64`     |
| roothide  | `iphoneos-arm64e`    |

The architecture field names the **bootstrap layout**, not the CPU — both
packages carry the same arm64 binary. If you are unsure which you have, ask the
device: `dpkg --print-architecture`.

Requires iOS 15 or later. Then run `fastfetch` in a terminal on device. Your
config goes in `~/.config/fastfetch/config.jsonc`; presets ship in
`/usr/share/fastfetch/presets` under the bootstrap prefix.

## What this repository is

Packaging, not a fork. There is no application source here: the build fetches
fastfetch at a pinned commit, applies the patches in `patches/`, cross-compiles
it with CMake against the iPhoneOS SDK, and produces the two packages.

Upstream has no iOS target, so the patches add one: iOS implementations of the
OS, display, packages (dpkg) and OpenGL detectors, no-op fallbacks for the
macOS-only ones (Wi-Fi, Bluetooth, sound, fonts, wallpaper, media, brightness,
camera), and a config search path derived from where the binary was installed
so the same build works in `/var/jb`, a RootHide jbroot, or `/`. See
`AGENTS.md`.

## Build it yourself

Needs macOS with Xcode, CMake and Ninja, plus `ldid` and `dpkg`
(`brew install cmake ninja ldid dpkg`).

```sh
make check     # scripts, config, patch set
make debs      # both packages + SHA256SUMS, into build/Packages
```

To install on an attached device over USB:

```sh
iproxy 4422:2222 &
make install
```

`make help` lists the rest.

## Credits

fastfetch is by Carter Li and contributors, MIT. This repository only packages
it; the iOS patches are MIT.
