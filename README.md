# omarchy-system-config-apple-display

The Apple Studio Display's intermittent grey flashing on an Omarchy (Arch + Hyprland) laptop is fixed by pinning its DisplayPort link to 4 lanes at HBR3 on every boot and hotplug.

## Background

The Apple Studio Display flashed grey at random because amdgpu kept
renegotiating the DisplayPort link rate. Lowering the resolution and
blocking the display's USB hub (USBGuard) didn't help; pinning the link to
4 lanes @ HBR3 (8.1 Gb/s per lane) stopped it immediately (diagnosed 2026-09-20).

The setting lives in debugfs, which resets on every reboot and reconnect,
so three pieces re-apply it:

| File | Installs to | Role |
|---|---|---|
| `dp-lock-hbr3.sh` | `/usr/local/bin/` | Finds the display by EDID and writes `4 0x1e` to its connector's `link_settings` in `/sys/kernel/debug/dri`. Retries for up to 30 s. |
| `dp-lock-hbr3.service` | `/etc/systemd/system/` | Oneshot service, enabled at boot, that runs the script. |
| `99-dp-lock-hbr3.rules` | `/etc/udev/rules.d/` | Starts the service on every DRM hotplug. |

The display is matched by EDID because its connector name changes with port
and plug order (DP-4, DP-2, ...).

## Install

```sh
sudo ./install.sh
```

## Check

```sh
journalctl -t dp-lock-hbr3
```

Expect `locked DP-x to 4 lanes @ HBR3` after boot and each replug.
`no enabled Studio Display connector found` while the display is unplugged is
harmless. Each hotplug runs the script ~3 times (udev sends several change
events); also harmless.

If the display isn't matched, check its EDID name with
`edid-decode < /sys/class/drm/card*-DP-*/edid` and set `DP_LOCK_EDID_NAME`
in the service (`Environment=DP_LOCK_EDID_NAME=...`).

## Caveats

- Relies on amdgpu's debugfs `link_settings`, which is not a stable API. If
  flashing returns after a kernel or amdgpu update, check the journal first.
- These files were rebuilt from the write-up after the originals were lost
  in a system restore.
