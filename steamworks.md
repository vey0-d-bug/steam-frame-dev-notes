# Steamworks and Steam installs

## Getting a Steam app into the headset's Library

For an unreleased app (a Playtest, or a store app before release) the headset only listed it after:

- the app's General > Supported OS set to Linux + SteamOS (it was empty), and the depot's OS set to Linux + SteamOS too;
- store and library art published;
- a headset reboot.

An unreleased Playtest stays "Not Playable"; a developer comp package with a Release State Override lets the owner install and run it anyway. The install lands in `~/.local/share/Steam/steamapps/common/<install folder>/` with execute bits kept. Changing the install folder in Steamworks later does not move an installed copy.

## The runtime of a Steam app

- Set the app's Linux runtime to **Steam Linux Runtime 4.0 for arm64** (`SteamLinuxRuntime_steamrt4-arm64` in the launch config), and leave the launch option's CPU Architecture at **"Any"**. With no runtime, or with CPU Architecture "64-bit OS", the headset launches the app through `fex-compat-tool` and the x86 soldier/scout runtime: `uname` says x86_64, only x86 Vulkan drivers are listed and `vkCreateInstance` fails with `VK_ERROR_INCOMPATIBLE_DRIVER`.
- With the runtime set it runs through `SteamLinuxRuntime_4-arm64/_v2-entry-point` on aarch64 inside pressure-vessel (`steamrt4_platform_4.0.20261021.265997`), with the aarch64 freedreno ICD and the OpenXR manifest at `/usr/lib/pressure-vessel/overrides/...`. A check for "the headset's own runtime" has to accept that path.
- **The headset's Steam maps an app to its runtime only when Steam itself starts** (`compat_log.txt`: "Mapping AppID ... to tool ..."). A runtime change published in Steamworks reaches the cached appinfo within minutes, but launches keep the old mapping until Steam restarts (a reboot did it).
- SteamVR identifies the Steam-launched process by its host pid under `steam.app.<appid>` (`SetApplicationPid` in `vrserver.txt`).

## vrpreferences.json

`vrpreferences.json` with `preferMinRefreshRate` 120 at the package root was installed but **SteamVR never read it**: no "Loading app-provided preferences" line in `vrserver.txt` (a Steam VR game gets one at every SteamVR start), and "Failed to get app info for steam.app.<appid> when getting app settings". Likely reason, unverified: SteamVR reads it only for apps in Steam's VR list (`~/.local/share/Steam/config/steamapps.vrmanifest`), which needs the store page's VR support declared; a Playtest has none of its own. The per-app rate route in [refresh-rate.md](refresh-rate.md) works instead.

## Steamworks SDK on ARM64

Steamworks SDK 1.63 added `linuxarm64` libraries, 1.64 set them to a 4 KB max page size, and 1.65 added `ISteamUtils::IsRunningOnSteamHardware()` with a Steam Frame value. Whether `SteamAPI_Init`, lobbies, Steam Datagram Relay and Steam Voice work from an ARM64 binary on the headset: unverified.
