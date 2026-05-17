# MagicMirror² Setup Guide — Raspberry Pi 4 (2GB)
> A complete step-by-step guide based on a real installation, including every error encountered and how to fix it.

---

## System Specs
- **Hardware:** Raspberry Pi 4 Model B — 2GB RAM
- **OS:** Raspberry Pi OS Bookworm 64-bit Desktop (Full)
- **Username:** `admin` (replace with your own username wherever you see `admin`)
- **MagicMirror version:** v2.36.0
- **Node.js version:** v22.22.2
- **npm version:** 10.9.7

---

## Part 1 — OS Installation

### What to flash
Use **Raspberry Pi Imager** and flash:
- **Raspberry Pi OS Bookworm 64-bit (Full/Desktop)**
- Do NOT use Lite version — MagicMirror needs a desktop environment (Electron won't run without it)
- Do NOT use Ubuntu Desktop — too heavy for 2GB RAM

### First boot
- Connect to WiFi
- Set timezone (Asia/Kolkata for India)
- Change default password
- The system should auto-login to desktop on reboot — confirm this works before moving forward

### ⚠️ What NOT to do
- Do not use Raspberry Pi OS Lite — MagicMirror will fail
- Do not use 32-bit OS — 64-bit is more efficient for 2GB RAM
- Do not skip WiFi setup — Node.js and MagicMirror need internet to install

---

## Part 2 — System Update

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
sudo reboot
```

This takes 5–15 minutes. Let it finish completely before doing anything else.

---

## Part 3 — Check RAM & Swap

```bash
free -h
```

On Raspberry Pi OS Bookworm, swap is automatically configured to match your RAM (1.8GB swap for 2GB RAM). You do not need to manually set up swap — Bookworm handles it.

### ⚠️ Note
Older guides tell you to run `sudo dphys-swapfile swapoff` — this will say **command not found** on Bookworm. That is normal. Ignore it.

---

## Part 4 — GPU Memory (Bookworm method)

On Bookworm, the GPU memory option is removed from `raspi-config`. Set it manually:

```bash
sudo nano /boot/firmware/config.txt
```

Add this line at the very bottom:

```
gpu_mem=128
```

Save with `Ctrl+O` → Enter → `Ctrl+X`. Reboot.

### ⚠️ Note
In raspi-config, Performance Options will show:
- P2 Overlay File System
- P3 Fan

There is NO GPU memory option — this is normal on Bookworm. Use the method above instead.

---

## Part 5 — Install Node.js

MagicMirror² requires a specific Node.js version. Check what version is needed before installing.

### Step 1 — Install Node.js 22

```bash
curl -sL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### Step 2 — Verify

```bash
node -v   # Should show v22.x.x
npm -v    # Should show 10.x.x
```

### ⚠️ Common Error — `engine notSupported`
If you install Node 20 first and get this error when installing MagicMirror:

```
error code EBADENGINE
error: engine: node>=22.21.1<23||>=24
```

Fix it by upgrading Node:

```bash
sudo apt remove -y nodejs
curl -sL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node -v  # Must show v22.x.x
```

---

## Part 6 — Install MagicMirror²

### Step 1 — Clone the repository

```bash
cd ~
git clone https://github.com/MagicMirrorOrg/MagicMirror.git
```

### ⚠️ Note
The one-line bash installer (`curl | bash`) may return a 404 error. Use `git clone` instead — it always works.

### Step 2 — Install dependencies

```bash
cd ~/MagicMirror
npm run install-mm
```

This takes 5–10 minutes. Successful output looks like:

```
added 380 packages in 46s
no husky installed
```

### Step 3 — Create config file

```bash
cp ~/MagicMirror/config/config.js.sample ~/MagicMirror/config/config.js
```

### ⚠️ Common Error — Config not found
If you see:
```
[ERROR] Could not find config file: /home/admin/MagicMirror/config/config.js
```
It means you forgot to copy the sample config. Run the `cp` command above.

### Step 4 — Test MagicMirror manually

```bash
cd ~/MagicMirror && ~/MagicMirror/node_modules/.bin/electron ~/MagicMirror/js/electron.js
```

You should see the MagicMirror interface with clock, date, weather, and news.

### ⚠️ Important — Always run from MagicMirror directory
MagicMirror needs to be launched from its own directory. It looks for `index.html` relative to where it runs.

- ✅ WORKS: `cd ~/MagicMirror && electron js/electron.js`
- ❌ FAILS: `electron /home/admin/MagicMirror/js/electron.js` (without cd first)

If you skip the `cd`, you will get:
```
Error: ENOENT: no such file or directory, open 'index.html'
```

---

## Part 7 — Autostart on Boot

This was the most challenging part. Here is what works on **Raspberry Pi OS Bookworm with labwc window manager**.

### Check your window manager

```bash
echo $XDG_CURRENT_DESKTOP   # Shows: labwc:wlroots
echo $XDG_SESSION_TYPE       # Shows: wayland
```

### Check your Wayland display socket

```bash
ls /run/user/1000/wayland*
# Output: /run/user/1000/wayland-0
```

### Step 1 — Create a startup script

```bash
nano /home/admin/start_mirror.sh
```

Paste this:

```bash
#!/bin/bash
cd /home/admin/MagicMirror && WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/1000 /home/admin/MagicMirror/node_modules/.bin/electron /home/admin/MagicMirror/js/electron.js
```

Save with `Ctrl+O` → Enter → `Ctrl+X`.

Make it executable:

```bash
chmod +x /home/admin/start_mirror.sh
```

### Step 2 — Add to labwc autostart

```bash
mkdir -p ~/.config/labwc
nano ~/.config/labwc/autostart
```

Add this single line:

```
/home/admin/start_mirror.sh &
```

Save. Reboot and MagicMirror will start automatically.

### ⚠️ Things that did NOT work (don't waste time on these)
| Method | Why it failed |
|---|---|
| `~/.config/autostart/magicmirror.desktop` | labwc doesn't use GNOME autostart folder |
| `pm2 startup` | pm2 starts before Wayland display is ready |
| `systemd service` | System services can't access user Wayland session |
| `crontab @reboot` | Same Wayland display access problem |
| `WAYLAND_DISPLAY=wayland-1` | Wrong socket name — it's `wayland-0` on this system |
| Running electron without `cd` first | MagicMirror can't find `index.html` |

### ⚠️ Common Error — Wayland connection failed
```
Failed to connect to Wayland display
Failed to initialize Wayland platform
```
This means the script is running before the Wayland display socket is ready, OR the `WAYLAND_DISPLAY` or `XDG_RUNTIME_DIR` variables are wrong.

Fix: Make sure your script has both:
```bash
WAYLAND_DISPLAY=wayland-0
XDG_RUNTIME_DIR=/run/user/1000
```

---

## Part 8 — Killing MagicMirror

MagicMirror runs fullscreen. To close it:

```bash
pkill electron
```

Or press **Alt + F4** if you have a keyboard connected.

---

## Quick Reference — Working Commands

| Task | Command |
|---|---|
| Start MagicMirror manually | `cd ~/MagicMirror && ~/MagicMirror/node_modules/.bin/electron ~/MagicMirror/js/electron.js` |
| Stop MagicMirror | `pkill electron` |
| Check RAM | `free -h` |
| Check Node version | `node -v` |
| Check autostart file | `cat ~/.config/labwc/autostart` |
| Check Wayland socket | `ls /run/user/1000/wayland*` |
| Check if electron is running | `ps aux | grep electron` |

---

## Next Steps
- [ ] PIR sensor — motion detection (home/idle state)
- [ ] USB camera — hand gesture detection (MediaPipe)
- [ ] Voice control — Google Assistant + relay control
- [ ] Relay module — 12V light control via voice

---

*Guide based on real installation — Raspberry Pi 4 Model B 2GB, May 2026*
