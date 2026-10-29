# Refresh rate

- **Apps start at 72 Hz.** SteamVR runs each app at its own per-application rate (VR Settings > Video > Per-Application Video Settings, "Frame Limit / Refresh Rate Override"), and unset means 72 Hz. The headset-wide rate does not apply to apps.
- `xrRequestDisplayRefreshRateFB` returns `XR_SUCCESS` and does nothing. `XR_FB_display_refresh_rate`'s offered list is unreliable: one value (120 or 72), changing between launches, unrelated to the real pacing.
- The runtime can change pacing mid-session on its own while the system UI is up (seen going 72 to 120, 96 and 108 Hz).
- `vrpreferences.json` in a Devkit upload is not read, and in a Steam install it was not read either, for an app Steam doesn't flag as VR (see [steamworks.md](steamworks.md)).
- "Reset to Default" in Per-Application Video Settings did not remove a rate the app had written.

## What works: set the per-app rate through OpenVR

A child process connects to SteamVR as an OpenVR `VRApplication_Utility` client (an OpenVR SDK v2.15.6 loader; SteamVR provides `/opt/steamvr/bin/linuxarm64/vrclient.so`), finds the app's key and calls `IVRSettings::SetInt32(<appkey>, "preferredRefreshRate", 120)`. No sudo.

- **Write it before the app's own `xrCreateInstance`**, and run nothing during the session. Written then, the same start runs at 120.00 Hz (applied about 5 s in), and Steam's quit still arrives as SIGINT. Written during a session it can apply live, but see the process problems below.
- App keys: a Steam-launched title is `steam.app.<SteamAppId>` (or from `GetApplicationKeyByProcessId`). An ssh-started OpenXR app is `system.generated.openxr.<XR app name>.<binary>`, found only through `GetApplicationKeyByIndex`. SteamVR files every client from one program file under the key it gave the *first* client from that file, until SteamVR restarts, so a utility client that connects first can be filed as `system.generated.<binary>` and the app's session too. Write both keys for ssh starts.
- Rates are stored in `~/.config/openvr/config/steamvr.vrsettings`, in sections named by app key. Keep one stable Devkit title name, since the key is per title.
- **The utility child must not inherit the Steam game's environment.** With `SteamAppId` inherited, SteamVR registered the helper as `steam.overlay.<appid>`, Steam tracked it as one of the game's processes (`~/.local/share/Steam/logs/gameprocess_log.txt`), and quitting the game from Steam stopped reaching the app. Strip `Steam*`, `STEAM_COMPAT_APP_ID` and `LD_PRELOAD`. Even then, a helper running during the session could be taken by SteamVR as the app itself (`SetApplicationPid` to the helper's pid), and at its exit SteamVR dropped the app's process. Hence: write before the session.
- Measured once the per-app rate was set: 90.001 Hz and 120.005 Hz. For video, 120 Hz divides 24, 30 and 60 evenly; 72 judders 60 fps; 144 suits 24/48 only. 144 Hz per app: unverified.
