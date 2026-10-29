# Steam Frame developer notes

Facts about Valve's Steam Frame headset, gathered while building a native Linux ARM64 OpenXR video player for it (October 2026). Each fact was checked on a retail unit unless it says "unverified", and most say how it was checked. Device details reflect SteamOS 0.6.0 (preview branch, build 20261020), SteamVR/OpenXR 2.18.4 and kernel 7.2.0; later builds may differ.

| File | What it covers |
|---|---|
| [device.md](device.md) | CPU, OS, filesystem, panels, battery, heat, audio, fonts |
| [openxr.md](openxr.md) | The runtime, extensions, views, controllers, eye gaze, focus and presence, passthrough |
| [compositor-layers.md](compositor-layers.md) | Quad layers, layer ordering, per-eye quads |
| [refresh-rate.md](refresh-rate.md) | Why apps start at 72 Hz, and the per-app rate route that works |
| [graphics-and-video.md](graphics-and-video.md) | Vulkan, swapchains, hardware decode, 10-bit on the decoder |
| [shell-deploy-launch.md](shell-deploy-launch.md) | ssh, Devkit Client, runtimes, launching through Steam from ssh, polkit, sleep, power off |
| [steamworks.md](steamworks.md) | Getting a Steam app onto the headset, its runtime, vrpreferences.json |
| [polkit/](polkit/) | A polkit rule to allow powering the headset off over ssh |

Official docs worth reading first (Steamworks partner site, partner.steamgames.com): `doc/steamframe/debugging`, `doc/steamhardware/steamframe/loadgames`, `doc/steamframe/compatibility`, `doc/steamhardware/steamframe/engines/custom`, `doc/steamhardware/steamframe/input`, `doc/steamhardware/steamframe/vrpreferences`.
