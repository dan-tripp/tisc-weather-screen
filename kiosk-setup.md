# Sailing Club Weather Kiosk — Setup & Rationale

A Raspberry Pi configured as an unattended, maintenance-free kiosk that displays a
weather webpage full-screen on a monitor in the club common area. It is designed to
survive crashes, power cuts, and stale data on its own, with **zero** intervention
from non-technical users.

---

## 1. Hardware & system

| Item | Value |
|------|-------|
| Board | Raspberry Pi 3 Model B Rev 1.2 |
| Network | Onboard 2.4 GHz WiFi (`brcmfmac` driver) |
| OS | Raspberry Pi OS, pre-Bookworm era |
| Session | X11 + LXDE / Openbox |
| Network stack | `dhcpcd` (NetworkManager **inactive**) |
| Boot config path | `/boot/config.txt` |
| Displayed URL | `https://dan-tripp.github.io/tisc-weather-screen/` |

These facts matter because they determine which config mechanisms apply (X11/LXDE
autostart rather than Wayland/labwc; `dhcpcd` rather than NetworkManager; the old
`/boot/config.txt` location rather than `/boot/firmware/config.txt`).

---

## 2. Design philosophy — layered recovery

The system is built as four independent safety layers, each covering a different
failure mode. Any one failing still leaves the others intact.

1. **Crash recovery (instant)** — a `while` loop respawns the browser the moment it
   dies. Event-driven, so recovery is near-instant rather than waiting on a poll.
2. **Periodic refresh (every 4 hours)** — a cron job kills the browser; layer 1
   immediately relaunches it with a fresh page load. Keeps data current and clears
   memory bloat (the Pi 3B has only 1 GB RAM and Chromium leaks over time).
   NOTE: currently this is disabled. The nightly restart shoudl be enough for memory leaks.
3. **Nightly reboot (4 AM)** — a full OS-level reset of the network stack, kernel,
   and everything else. Recovers from any wedged state.
4. **Power-loss resilience** — an overlay (read-only) filesystem means a yanked power
   cord can't corrupt the SD card. The Pi always boots back to a known-good image.
   NOTE: Currently this is disabled. Once we have the system running in prod for a while
   can enable the readl only file system. 

The guiding principle for everything below: **nothing should change itself.** The
system is frozen in a known-good state, and updates happen only when a human chooses,
in person, with a test afterward.

---

---

## 3. Setup steps

### 3.1 Auto-login to the desktop

```
sudo raspi-config
```

`System Options → Boot / Auto Login → Desktop Autologin`.

**Why:** the Pi must boot straight into the graphical session with no human logging in —
this is what makes both unattended boot and power-cut recovery possible. On X11/LXDE,
the desktop session is also what reads the autostart file in step 4.3.

Also set `Display Options → Screen Blanking → Off` so the screen never sleeps.

### 3.2 The kiosk launcher script

Create `~/kiosk.sh`:

```bash
#!/bin/bash
xset s off
xset -dpms
xset s noblank

URL="https://dan-tripp.github.io/tisc-weather-screen/"

while true; do
  chromium-browser --kiosk --disable-gpu --mute-audio --log-level=3 --noerrdialogs --disable-infobars --no-first-run --disable-session-crashed-bubble --disable-features=TranslateUI --check-for-update-interval=31536000 "$URL" >/dev/null 2>&1
  sleep 2
done
```

```
chmod +x ~/kiosk.sh
```

**Why a `while` loop instead of relying on LXDE's `@` respawn:** the loop is more
predictable and makes the 20-minute refresh deterministic — when cron kills Chromium,
the loop *always* relaunches it. The `sleep 2` prevents a tight crash-loop from pegging
the CPU if the browser ever can't start at all.

> **Note:** keep the `chromium-browser ... "$URL"` invocation on a **single physical
> line**. A stray line break (or a trailing space after a `\` continuation) splits the
> command — the browser launches with no URL (shows its homepage) and the leftover
> flags error out as "command not found." Writing the file via `cat > ~/kiosk.sh << 'EOF'`
> avoids editor line-wrap mistakes.

#### Chromium flag reference

| Flag | Purpose |
|------|---------|
| `--kiosk` | Full-screen, no toolbars/chrome, no exit UI |
| `--disable-gpu` | Force software rendering. The Pi 3's GPU stack fails to initialize ("Exiting GPU process due to errors"); software rendering is plenty fast for a weather page and far more stable |
| `--mute-audio` | Guarantees silence — the page can't emit sound if the browser is muted (covers every output path, including HDMI) |
| `--log-level=3` | Suppresses info/warning console spam (e.g. the harmless "GPU stall due to ReadPixels" performance message from the embedded radar's WebGL) |
| `--noerrdialogs` | No error pop-ups |
| `--disable-infobars` | No info bars |
| `--no-first-run` | Skip first-run setup wizard |
| `--disable-session-crashed-bubble` | Critical: stops the "Chrome didn't shut down correctly — Restore pages?" banner that would otherwise sit on screen forever after every kill/restart with no one to dismiss it |
| `--disable-features=TranslateUI` | No "translate this page?" prompt |
| `--check-for-update-interval=...` | Effectively disables the browser's own update nagging |
| `>/dev/null 2>&1` | Discards all output. Add this only after confirming everything works — while debugging you want to see real errors |

### 3.3 Autostart at boot

The X11/LXDE autostart file (this is the X11 equivalent of a Wayland `labwc/autostart`):

```
sudo nano /etc/xdg/lxsession/LXDE-pi/autostart
```

Add at the bottom:

```
@/home/<USER>/kiosk.sh
```

**Why:** LXDE runs this file once when the desktop session starts (which, thanks to
auto-login, is at every boot). The `@` prefix tells LXDE to relaunch the entry if it
exits — a second layer of respawn protection on top of the loop inside the script.

### 3.4 Cron jobs

**User crontab** (`crontab -e`) — the 20-minute refresh:

```
*/20 * * * * /usr/bin/pkill chromium >/dev/null 2>&1
```

This kills only the Chromium processes; the `kiosk.sh` loop survives and relaunches the
browser within ~2 seconds with fresh data.

**Root crontab** (`sudo crontab -e`) — nightly reboot and WiFi power-save:

```
0 4 * * * /sbin/reboot
@reboot sleep 20 && /sbin/iw dev wlan0 set power_save off
```

- The 4 AM reboot is layer 3 — a full clean reset every night.
- The `@reboot` line disables WiFi power-saving (see 4.6). The `sleep 20` gives the
  wireless interface time to come up first; it re-applies on every boot.

### 3.5 Disable automatic updates

```
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer
sudo systemctl mask apt-daily.service apt-daily-upgrade.service
dpkg -l | grep unattended-upgrades        # if present:
sudo apt purge unattended-upgrades
```

**Why:** Debian's daily apt timers wake at random times to refresh package lists and
(if configured) install updates — causing network/disk/CPU churn and, worse, the chance
of an update silently changing behavior at 3 AM with no one to notice. We update on our
own schedule only (see Maintenance). Leave `systemd-timesyncd` (NTP) running, though —
the page stamps "Loaded at [time]," so the clock must stay correct.

### 3.6 WiFi power-save off

Already configured via the `@reboot` cron line in 4.4. Confirm it works:

```
sudo iw dev wlan0 set power_save off
iw dev wlan0 get power_save        # should report: Power save: off
```

(If `iw` is missing: `sudo apt install -y iw`.)

**Why:** the onboard radio defaults to a power-save mode that lets it doze when idle.
On a display that only touches the network every 20 minutes, the link can sleep and the
next refresh hangs. Disabling power-save keeps the connection alive. The nightly reboot
is a backstop if it ever drops anyway.

> **Note:** this is the `dhcpcd` approach. On a NetworkManager system you'd instead use
> `/etc/NetworkManager/conf.d/wifi-powersave-off.conf` with `wifi.powersave = 2`. That
> file does nothing here because NetworkManager is inactive.

### 3.7 Audio mute (hardware-level, optional)

`--mute-audio` in the launcher already guarantees silence. As optional extra insurance
you can disable the onboard audio device entirely:

```
sudo nano /boot/config.txt
```

Set `dtparam=audio=on` → `dtparam=audio=off` (or add the line). Reboot to apply.

**Caveat:** if the display is an HDMI TV/monitor, audio can travel over a separate HDMI
path this doesn't always cover — which is exactly why `--mute-audio` is the reliable
guarantee. The hardware disable is belt-and-suspenders only.

### 3.8 Overlay filesystem — DO THIS LAST

After confirming everything survives a test reboot:

```
sudo raspi-config
```

`Performance Options → Overlay File System → Enable`.

**Why:** makes the SD card read-only, with all writes going to a RAM layer discarded on
every reboot. This means:

- A yanked power cord can't corrupt the card (the #1 killer of unattended Pis).
- Logs, cache, and any cruft accumulate only in RAM and clear nightly.
- Even if an update somehow slipped through, it can't persist — the next reboot returns
  the Pi to the exact known-good image.

**Trade-off:** while overlay is on, *no changes survive a reboot*. To change anything,
disable overlay → make the change → re-enable (see Maintenance). Enable it **last** for
this reason.

---

## 4. Maintenance guide

### Exiting the kiosk for maintenance

If launched from a terminal: **Alt+Tab** back to the terminal, then **Ctrl+C**.

Otherwise switch to a text console: **Ctrl+Alt+F2**, log in, then (in this order):

```
pkill -f kiosk.sh      # kill the loop FIRST, or it just relaunches the browser
pkill chromium
```

Return to the desktop with **Ctrl+Alt+F1** (or **F7**).

### Changing the displayed URL

1. If overlay is on: `raspi-config → Performance Options → Overlay File System → Disable`, reboot.
2. Edit the `URL=` line in `~/kiosk.sh`.
3. Reboot and confirm.
4. Re-enable overlay, reboot.

### Updating the system (do this deliberately, every few months)

1. `raspi-config` → disable Overlay File System → reboot.
2. `sudo apt update && sudo apt full-upgrade`
3. Reboot; confirm the weather screen comes up and refreshes correctly.
4. Re-enable overlay → reboot.

Updates then only ever happen with a human present to catch a problem — never
unattended.

### Common issues

| Symptom | Likely cause / fix |
|---------|--------------------|
| Browser shows homepage, not the weather page | URL not passed — Chromium command got split across lines. Keep it on one physical line |
| `--<flag>: command not found` | Same line-break problem; rewrite the script with the `cat > ... << 'EOF'` heredoc |
| GPU / EGL errors at launch | Expected on Pi 3; `--disable-gpu` forces stable software rendering |
| "GPU stall due to ReadPixels" | Harmless performance *warning* from the radar's WebGL; silenced by `--log-level=3`. Not an error |
| Can't exit kiosk | The `while` loop relaunches the browser — kill `kiosk.sh` first, then `chromium` |
| Changes don't persist after reboot | Overlay filesystem is enabled — disable it to make changes |

---

## 5. Files & settings touched

| Path / setting | Purpose |
|----------------|---------|
| `~/kiosk.sh` | Launcher: screen-blank disable + browser respawn loop |
| `/etc/xdg/lxsession/LXDE-pi/autostart` | Starts `kiosk.sh` at boot (`@` respawn) |
| user `crontab` | 20-minute browser refresh |
| root `crontab` | Nightly 4 AM reboot + `@reboot` WiFi power-save off |
| `raspi-config` | Desktop Autologin; Screen Blanking off; Overlay FS (last) |
| `apt-daily*` timers/services | Disabled + masked (no auto-updates) |
| `unattended-upgrades` | Purged if present |
| `/boot/config.txt` | Optional `dtparam=audio=off` |
