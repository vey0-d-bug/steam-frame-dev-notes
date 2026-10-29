# Device

- **CPU:** aarch64, 8 cores, a mix of Cortex-A520 and Cortex-A720 (flags include sve2, i8mm, bf16; ARMv8 crypto: aes, pmull, sha1/2/3/512, sm3, sm4). Snapdragon 8 Gen 3 class. 15 GiB RAM, 7.6 GiB swap, 4 KiB pages. [ssh: `lscpu`, `free -h`, `/proc/cpuinfo`]
- **OS:** SteamOS 0.6.0, `VARIANT_ID="vr"`, Arch-based. Host glibc 2.43. [ssh: `/etc/os-release`, `ldd --version`]
- **Root filesystem** is btrfs and read-only for package installs: `pacman` fails with "Read-only file system" until `sudo steamos-readonly disable`. `sudo` over ssh needs `ssh -t`. (A `findmnt` check reads it as rw, which is misleading.) Whether installed packages survive an OS update: unverified.
- **Toolchain on the device:** gcc, g++, clang, cmake, ninja, pkg-config and git are in `/usr/bin`. Cross-compiling against an old glibc instead (a 2.31 sysroot from Ubuntu 20.04 arm64, clang + lld, static libstdc++, dynamic only against glibc and libvulkan) gives binaries that run on the host and inside Steam's runtimes.
- **Battery and power:** `/sys/class/power_supply` has `max1720x_bat_7-36` (capacity, status), `pm8550b-charger` and a USB-C `tcpm-source-psy-...` entry. The battery temperature is also a thermal zone.
- **Heat:** sustained software HEVC 4K60 10-bit decode (about 4 cores busy) holds the hottest CPU sensor at 94 to 96 °C, with big-core clocks dipping up to about 15 %, and no runaway over 20 minutes. Hardware decode of 8K30 sits at 55 to 59 °C.

## Panels

- One DSI connector, 4320x2160 = two 2160x2160 panels on a dual DSI link, driver `deckard-panel`. Modes 72, 80, 90, 96, 108, 120 and 144 Hz. No EDID and no "max bpc" property. [ssh: `/sys/class/drm`, `modetest -M msm -c`]
- **8 bits per colour.** The kernel's `drivers/gpu/drm/panel/panel-deckard.c` sets `MIPI_DSI_FMT_RGB888` and DSC with `bits_per_component = 8`. Valve publishes the kernel source at `holo-packages.steamos.cloud/archlinux-deckard-hotfixes/source/`; an unofficial mirror is `github.com/evlaV/deckard-linux` (branch `sm8650-7.2`). Whether SteamVR dithers a 10-bit swapchain down to 8: unverified.
- SteamVR recommends 1728x1728 per eye, below the native 2160.

## Audio

- PipeWire 1.6.8 with WirePlumber. The graph runs at 48000 Hz only (`clock.allowed-rates [ 48000 ]`). With `node.latency` 512/48000 it ran a 480-frame (10 ms) quantum; `pw_time.delay` was a steady 1504 samples, about 31.2 ms from callback to device. A new stream reports a delay of 0 for its first cycles; the real value shows 9 to 41 ms later.
- An app started over ssh gets working PipeWire with `XDG_RUNTIME_DIR=/run/user/1000`.
- **SteamOS spatializes every output stream by default** while the default sink is the built-in speakers: WirePlumber's `/etc/wireplumber/scripts/spatial-node-config.lua` inserts a `<name>.<serial>.spatialize_filter_chain` (LADSPA plugin `/usr/lib/ladspa/vraudiocompositor.so`) for each stream. The switch is `node.stream.enable-spatial-audio` in the `sm-settings` metadata. There is no per-stream opt-out property. Headphones and Bluetooth get no spatializer. What the spatializer does to the sound: unverified.
- Hardware: `hw:LPASS,0` in S16LE at 48 kHz, `node.force-quantum 480`.

## Fonts

- `/usr/share/fonts` holds about 460 MB in 4515 files: the Noto families (`noto/`, 624 files), the Noto CJK collections, Adwaita and X11 bitmap fonts. fontconfig 2.17.1, 583 families. `sans-serif` resolves to Noto Sans; Arial, DejaVu Sans and Liberation Sans map to Noto Sans. No `~/.fonts` or `~/.local/share/fonts` by default.
- An app's own static fontconfig building a cache over those folders takes about 2.3 s the first time.
