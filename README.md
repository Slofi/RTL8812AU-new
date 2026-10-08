# RTL8812AU-new — rtw88 for an RTL8812AU USB dongle on an Armbian Rock 5B, with a driver switch for monitor mode

A small, tested setup for a Rock 5B (RK3588, Armbian `current-rockchip64`, kernel 6.18) whose only Wi-Fi is an RTL8812AU USB dongle.

## Why
The widely used out-of-tree aircrack-ng `88XXau` driver (Realtek vendor code v5.6.4.2, 2019) stalled the link several times a day
(`read timeout` on long-lived TCP connections, whole-link stalls). The modern **rtw88** driver supports the RTL8812AU (`rtw88_8812au`,
mainline since 6.14), but Armbian's 6.18 kernel is built **without** `CONFIG_RTW88_8812AU`. So it is built out-of-tree from
[lwfinger/rtw88](https://github.com/lwfinger/rtw88) via DKMS.

`88XXau` is kept installed (blacklisted for auto-loading) because a Wi-Fi testing app was built around it; that app's monitor-mode
button switches drivers with `cd-wifi-driver monitor` and back with `cd-wifi-driver normal`.

## Contents
| Path | What |
|---|---|
| `sbin/cd-wifi-driver` | switch script: `status` · `monitor` (→ 88XXau, interface handed to the monitor app) · `normal` (→ rtw_8812au) · `old` (→ 88XXau for normal Wi-Fi, manual fallback) · `reset` (fresh reload of rtw_8812au) · `ensure` (boot check). One switch at a time (flock), logs to the journal (`-t cd-wifi-driver`), **never leaves the box without a bound driver** (falls back to the other one) |
| `systemd/cd-wifi-driver-boot.service` | runs `ensure` after every boot → the default driver is always back after a restart |
| `systemd/banshee.service.d-driver-restore.conf` | user-service drop-in: `ExecStopPost` → `normal` if the monitor app stops or crashes |
| `sudoers.d/cd-wifi-driver` | allows exactly `cd-wifi-driver monitor` / `normal` / `reset` / `old` for the apps' user |
| `rtw88/` | `dkms.conf` with the gcc pin, the modprobe files, rebuild + rollback notes |

## Gotchas this setup handles
- **Compiler pin:** Armbian's kernel is built with gcc-14, the userland default is gcc-13 → `CC=gcc-14` in `dkms.conf`, or every
  out-of-tree module fails to build/load.
- **USB 3 mode:** rtw88's default `switch_usb_mode=y` moved the dongle to USB 3; on this hub the SuperSpeed port failed
  (`Cannot enable. Maybe the USB cable is bad?`) and the dongle vanished until physically replugged → `switch_usb_mode=n`.
  USB 2 also avoids USB 3 RF noise on 2.4 GHz.
- **Two drivers, one USB ID:** both claim `0bda:8812` → blacklist the one you don't want auto-loaded or they race.
- **Module name ≠ driver name:** `88XXau` registers in sysfs as `rtl88XXau`.
- **Swap drivers over a wired link**, not over the Wi-Fi you are swapping.
- **rtw88 after hotspot (AP) mode:** AP mode works, but after it is switched off the firmware stops answering (`failed to get tx report from firmware`, empty scans) until the driver is reloaded → the dashboard's hotspot-off path runs `cd-wifi-driver reset` and waits for the client link, warning in the UI if it does not come back.

## Install (as done here)
```bash
git clone https://github.com/lwfinger/rtw88 ~/src/rtw88 && cd ~/src/rtw88 && git checkout a56bcd2
cp <this repo>/rtw88/dkms.conf.S444-cd dkms.conf          # adds CC=gcc-14
sudo dkms install $PWD
sudo cp <this repo>/rtw88/etc-modprobe.d-rtw88.conf.S444 /etc/modprobe.d/rtw88.conf
sudo cp <this repo>/rtw88/etc-modprobe.d-blacklist-88XXau-S444.conf /etc/modprobe.d/blacklist-88XXau-S444.conf
sudo install -m 755 sbin/cd-wifi-driver /usr/local/sbin/
sudo install -m 644 systemd/cd-wifi-driver-boot.service /etc/systemd/system/ && sudo systemctl enable cd-wifi-driver-boot
visudo -cf sudoers.d/cd-wifi-driver && sudo install -m 440 sudoers.d/cd-wifi-driver /etc/sudoers.d/
# then replug the dongle or reboot
```
Firmware `rtw88/rtw8812a_fw.bin` ships with `linux-firmware`. Adjust the interface name and user in the script/sudoers to your system.

## Results (2026-10-08)
rtw_8812au on USB 2: 0 % loss, ~4–5 ms to the gateway, 104 Mbit/s (MCS 13); clean monitor↔managed switching; reboot test passed
(driver, USB 2, NetworkManager, DKMS, boot check). **Packet injection** (`aireplay-ng --test`): identical on both drivers — 30/30 (100 %).
**Monitor scanning** with an explicit channel list (1–13, 30 s): rtw88 stayed on one channel (2 APs), 88XXau saw 4 APs / 6 clients on 2
channels — which is why monitor mode still swaps to 88XXau. Old driver backup: [Slofi/RTL8812AU](https://github.com/Slofi/RTL8812AU).

## Not affiliated
Uses [lwfinger/rtw88](https://github.com/lwfinger/rtw88) (GPL-2.0) unmodified apart from the `dkms.conf` compiler pin.
