# scrv

[![Platform](https://img.shields.io/badge/platform-Linux-0078D4?logo=linux&logoColor=white)](https://www.linux.org/)
[![Language](https://img.shields.io/badge/language-bash-4EAA25?logo=gnubash&logoColor=white)](https://www.gnu.org/software/bash/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Stars](https://img.shields.io/github/stars/Luquas95/scrv?style=social)](https://github.com/Luquas95/scrv)

A simple bash script that toggles the screensaver and screen lock on Linux.
Run it once to disable, run it again to re-enable.

## Installation

**Step 1** — Make the script executable:
```bash
chmod +x scrv
```

**Step 2** — Copy it to a system-wide location so it can be run from anywhere:
```bash
sudo cp scrv /usr/local/bin/
```

## Usage

```bash
scrv          # disable screensaver
scrv          # enable screensaver again
```

Tip: assign it to a keyboard shortcut in your DE settings for quick toggling.

---

## Dependencies

| Tool | Package (Debian/Ubuntu) | Notes |
|---|---|---|
| `bash` | pre-installed | version 4+ |
| `xset` | `x11-xserver-utils` | X11 only, silently skipped on Wayland |
| `dbus-send` | `dbus` | usually pre-installed |
| `gsettings` | `libglib2.0-bin` | GNOME/Cinnamon/MATE only |
| `systemd-inhibit` | `systemd` | available on most modern distros |
| `pkill` | `procps` | usually pre-installed |

---

## Supported environments

### X11

| Desktop | Mechanism |
|---|---|
| **Cinnamon** | gsettings + DBus + xset + polling |
| **GNOME** | gsettings + DBus + xset + polling |
| **MATE** | gsettings + DBus + xset + polling |
| **KDE Plasma** | systemd-inhibit + DBus + xset |
| **XFCE** | systemd-inhibit + DBus + xset + polling |
| **Openbox / i3 / dwm** and other WMs | xset + systemd-inhibit |

### Wayland

| Compositor / DE | Mechanism |
|---|---|
| **Hyprland** (hypridle) | SIGUSR1 → hypridle + systemd-inhibit |
| **Sway** (swayidle) | SIGUSR1/SIGUSR2 → swayidle + systemd-inhibit |
| **GNOME Wayland** | systemd-inhibit + DBus |
| **KDE Plasma Wayland** | systemd-inhibit + DBus |
| **labwc, river, niri** and other wlroots WMs | systemd-inhibit (when using swayidle) |

---

## How it works

The script applies multiple inhibition layers simultaneously, each covering a different environment:

1. **gsettings** — disables the screensaver and lock at the DE settings level (Cinnamon, GNOME, MATE)
2. **xset s off -dpms** — disables the X screensaver and DPMS blanking (X11)
3. **SIGUSR1/SIGUSR2** — directly pauses `hypridle` or `swayidle` (Wayland)
4. **systemd-inhibit** — idle inhibit at the logind level, works independently of the DE, including Wayland
5. **Polling loop (every 20 s)** — fallback that sends `SimulateUserActivity` over DBus and resets the X idle timer

The on/off state is tracked via a PID file at `/tmp/scrv-inhibit.pid`.

---

## Known issues

- **Hyprland with `ignore_system_dbus_inhibit = true` in hypridle.conf** — systemd-inhibit will be ignored; the screensaver will still be blocked via SIGUSR1, but the state won't be restored after hypridle restarts
- **Multiple instances of hypridle/swayidle** — `pkill` sends the signal to all matching processes, which may cause unexpected behavior
- **XWayland apps** — `xset` only affects the XWayland context and has no impact on the Wayland compositor itself
- **Race condition on startup** — if the screensaver activates just before the script runs, it may briefly flash before the polling loop deactivates it (only affects DEs where gsettings/inhibit don't take effect immediately)

## When it won't work

- **Non-systemd distros** (Alpine with OpenRC, Void with runit, Artix with s6...) — `systemd-inhibit` is unavailable; the script still works via xset + DBus + signals, but with weaker coverage
- **Custom idle daemons** (e.g. `xss-lock`, `swaylock` launched directly without swayidle) — the script has no way to pause them
- **Hyprland with both `ignore_dbus_inhibit = true` and `ignore_system_dbus_inhibit = true`** — both inhibit methods are intentionally disabled; SIGUSR1 to hypridle still works
- **Wayland without swayidle or hypridle** (e.g. custom solution via `wlr-idle` or another daemon) — depends on the specific daemon
- **Remote sessions (SSH, VNC without a DE)** — the DBus session bus may not be available

---

## License

MIT
