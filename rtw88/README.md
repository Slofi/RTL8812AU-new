# CD Wi-Fi driver — rtw88 (rtw_8812au) via DKMS — S444, 2026-10-08

**What runs:** `rtw_8812au` (+ `rtw_8812a`, `rtw_88xxa`, `rtw_usb`, `rtw_core`) from **lwfinger/rtw88** @ `a56bcd26e770257612a0803249cbd4095fc6feca` (2026-05-21),
DKMS package `rtw88/0.6` → `/lib/modules/<kernel>/updates/dkms/`, source `/usr/src/rtw88-0.6/`, build checkout `~/src/rtw88` on CD.
Why: the old aircrack-ng `88XXau` (Realtek vendor code v5.6.4.2 from 2019) stalled the link 1–6×/day since ≥09-26 (24 × read timeout).
Armbian's 6.18 kernel ships rtw88 but **not** `CONFIG_RTW88_8812AU` (it has `rtw88_88xxa` + `rtw88_8821au`), hence the out-of-tree build.

**Local changes (files in this folder):**
- `dkms.conf.S444-cd` = upstream `dkms.conf.upstream` + **`CC=gcc-14`** in `MAKE=` — the Armbian kernel is built with gcc-14, CD's default gcc-13
  breaks every out-of-tree module (the 2026-08-30 outage). `AUTOINSTALL=yes` → rebuilt on kernel updates. **Bump the pin if the kernel moves to gcc-15+.**
- `etc-modprobe.d-rtw88.conf.S444` = upstream `rtw88.conf` (blacklists the in-kernel `rtw88_*`) with **`options rtw_usb switch_usb_mode=n`**.
  🔴 With the upstream default `=y` the dongle switches to USB 3, the Genesys hub's SuperSpeed port fails (`Cannot enable. Maybe the USB cable is bad?`),
  the dongle drops off the bus and only a **physical replug** recovers it (happened S444). USB 2 also avoids USB3 RF noise on 2.4 GHz.
- `etc-modprobe.d-blacklist-88XXau-S444.conf` = `blacklist 88XXau` — both drivers claim `0bda:8812`; without this udev loads both and they race.

**Rebuild from scratch:** `git clone https://github.com/lwfinger/rtw88 ~/src/rtw88 && cd ~/src/rtw88 && git checkout a56bcd2` → copy `dkms.conf.S444-cd` over `dkms.conf`
→ `sudo dkms install $PWD` → copy both modprobe files to `/etc/modprobe.d/` (names without the suffix for rtw88.conf) → replug the dongle or reboot.
Firmware `rtw88/rtw8812a_fw.bin` comes from linux-firmware (already present; `make install_fw` NOT needed).

**Roll back to 88XXau:** `sudo rm /etc/modprobe.d/blacklist-88XXau-S444.conf && sudo modprobe -r rtw_8812au rtw_8812a rtw_88xxa rtw_usb rtw_core && sudo modprobe 88XXau`
(88XXau stays DKMS-installed: `realtek-rtl88xxau/5.6.4.2~20230501`). Do it **over the cable**, never over the Wi-Fi itself.

**Measured S444 (USB 2, ch 11, cable as fallback):** isolated ping 0 % loss, ~4.7 ms; tx 104 Mbit/s MCS 13.
**Monitor mode (Banshee) — updated later on 2026-10-08:** the first test ("reception nearly nil") ran while the dongle was stuck in the failing USB 3 mode. In USB 2, a 20 s 2.4 GHz scan saw 7 APs on rtw88 vs 6 on 88XXau (airodump aborted there). Decision taken: Banshee's monitor button swaps to 88XXau automatically and back (`../sbin/cd-wifi-driver`); whether that swap is still needed is open (fair test = Banshee's own scan method + an injection test, which transmits → the owner's go).
