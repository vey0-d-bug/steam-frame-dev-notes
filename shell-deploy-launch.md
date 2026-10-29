# Shell, deploy and launch

## Shell

- Developer Mode (Steam Settings > System > Enable Developer Mode, then Developer > Set User Password) enables ssh, adb and rdp. `ssh steamos@<hostname>` (the hostname is set in Settings > System > Hostname) or `adb shell` over USB. Key auth works once a key is installed.
- Helpers on the headset: `mesa_version`, `cdl` (Steam logs at `~/.local/share/Steam/logs`), `lepton` (the Android container tool).
- firewalld is active. A server run as `steamos` on an unprivileged port (no root, no firewall change) is reachable from the LAN at the headset's Wi-Fi address, at about 50 to 67 MB/s each way.
- For an ssh-started OpenXR app, set `DISPLAY=:1`, `XDG_RUNTIME_DIR=/run/user/1000` and Steam's `VK_INSTANCE_LAYERS`.

## Deploy with the Devkit Client

- The SteamOS Devkit Client uploads to `~/devkit-game/<name>/` and keeps execute bits from Windows. Title names must match `^[A-Za-z_][A-Za-z0-9_.]+$`. The Start Command accepts arguments.
- **Pick "Steam Linux Runtime 4.0 ARM64"** as the title's runtime. Steam then ran the binary directly on the host, with no container (the SLR 4 ARM64 container wasn't installed). "Steam Linux Runtime 3.0 (sniper)" is the x86 runtime and goes through FEX-Emu: a script start command gets x86-only Vulkan drivers (`XR_ERROR_RUNTIME_FAILURE`), and an ARM64 start command dies in FEX before `main`.
- A Steam-launched app has no visible stdout, so log to a file. Quitting from Steam sends SIGINT.

## Launching through Steam from ssh

Useful for automated checks of the real library path, without touching the headset:

- `steam steam://rungameid/<id>`, with a Devkit title's shortcut id from `shortcuts.vdf`, starts the title exactly as the library does: in the headset's local session, on the host, with `nmcli`, `loginctl` and `systemd-inhibit` on its PATH.
- `steam -applaunch <appid> <options>` hands the launch and its options to the running Steam client ("Steam is already running, exiting (command line was forwarded)"), starts no second client, and the options reach the launch script. No dialog appeared for the options.
- To check which path ran, have the app log how it was launched (for example from `SteamAppId` and the runtime variables) and read `~/.local/share/Steam/logs/gameprocess_log.txt` and SteamVR's `vrserver.txt` afterwards.

## polkit, sleep and power

- polkit's admin group is `wheel` (`50-default.rules`). No rule in `/usr/share/polkit-1/rules.d` grants `steamos` any NetworkManager action; the only `steamos`-specific rule is Valve's `com.steampowered.atomupd1`. `/etc/polkit-1/rules.d` is readable only by root and polkitd.
- Over ssh (a non-local session) `nmcli general permissions` reports NetworkManager's `settings.modify.*` and `network-control` as `auth`. An app launched through the headset's Steam runs in the seat's local active session and gets `network-control`, `settings.modify.own`, `settings.modify.system` and `wifi.scan` as `yes`. For example `nmcli connection modify --temporary <profile> 802-11-wireless.bssid <ap>` re-activated the connection in about 0.6 s and stopped background scans.
- **Sleep inhibit:** `systemd-inhibit --what=sleep --mode=block` is refused over ssh ("Interactive authentication required") but allowed for an app launched from the library. Unworn with the proximity sensor uncovered, the headset did not suspend in 11 minutes. logind: IdleAction `ignore`, HandlePowerKey `ignore` (powerbuttond handles the button).
- **Sleep from ssh:** neither `steam steam://shortpowerpress` nor `systemctl suspend` works; one short press of the power button does, and a second press wakes it.
- **Power off from ssh:** `systemctl poweroff` as `steamos` is refused ("Interactive authentication required"); logind answers `challenge` to `CanPowerOff`. A polkit rule for `org.freedesktop.login1.power-off` and `org.freedesktop.login1.power-off-multiple-sessions`, installed into `/etc/polkit-1/rules.d` with sudo, allows it. See [polkit/](polkit/).
