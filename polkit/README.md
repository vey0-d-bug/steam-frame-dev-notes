# Power off over ssh

As `steamos` over ssh, `systemctl poweroff` is refused ("Call to PowerOff failed: Interactive authentication required."), and logind answers `challenge` to `CanPowerOff`. polkit's admin group is `wheel`, and `/etc/polkit-1/rules.d` is readable only by root and polkitd.

`50-steamos-poweroff.rules` allows `org.freedesktop.login1.power-off` and `org.freedesktop.login1.power-off-multiple-sessions` for the `steamos` user only. Install it with sudo over `ssh -t`:

```
sudo install -m 644 50-steamos-poweroff.rules /etc/polkit-1/rules.d/
```

Then check with `busctl call org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager CanPowerOff`, which should answer `yes` instead of `challenge`. Whether the rule survives a SteamOS update: unverified. Remove the file to undo it.

This rule follows the standard polkit JavaScript rules format; the facts above were observed on the headset, but this exact file has not yet been confirmed on a unit.
