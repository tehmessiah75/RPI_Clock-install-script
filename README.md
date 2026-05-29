# 🕐 Waveshare 3.5" LCD (A) + rpi_clock — Automated Setup

> **Turn a Raspberry Pi 2B or Pi Zero v1.3 into a dedicated clock & weather display using a Waveshare 3.5" SPI touchscreen — with one script.**

![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-2B%20%7C%20Zero%20v1.3-red?logo=raspberrypi)
![OS](https://img.shields.io/badge/OS-RPi%20OS%20Lite%2032--bit-blue)
![Bookworm](https://img.shields.io/badge/Bookworm%20%7C%20Trixie-supported-green)
![License](https://img.shields.io/badge/License-GPL--3.0-yellow)

---

## 📖 Overview

This project provides a **single Bash script** that takes a fresh Raspberry Pi OS Lite (32-bit) install and automatically:

- Installs and configures the **Waveshare 3.5" LCD (A)** (ILI9486 SPI + ADS7846 touch)
- Installs [texadactyl/rpi_clock](https://github.com/texadactyl/rpi_clock) — a Python 3 / Tkinter clock & weather display
- Configures a **systemd service** for clean boot, auto-restart on crash, and hardware watchdog
- Moves logs, temp files, and swap to **RAM (tmpfs)** to extend SD card lifespan
- Runs the clock fullscreen on the LCD at 480×320 resolution

### The end result

Plug in power → Pi boots → clock appears on the LCD → weather updates automatically.

No desktop environment. No manual steps after the script runs. Survives crashes and power cuts.

---

## 🛠️ Hardware Requirements

| Component | Notes |
|---|---|
| **Raspberry Pi 2 Model B** or **Pi Zero v1.3** | Tested targets. Should work on Pi 3/Zero W too. |
| **Waveshare 3.5" RPi LCD (A)** | 480×320, SPI, resistive touch (ILI9486 + ADS7846) |
| **MicroSD card** (8GB+) | Endurance card recommended (Samsung PRO Endurance / SanDisk MAX Endurance) |
| **5V 2.5A power supply** | Reliable power prevents SD corruption |
| **Internet connection** | Required for weather API. See [Connectivity Note](#-connectivity-note-pi-2b--pi-zero-v13) below. |

---

## 💾 Software Requirements

| Item | Details |
|---|---|
| **Raspberry Pi OS Lite (32-bit)** | Bookworm or Trixie. Download from [raspberrypi.com](https://www.raspberrypi.com/software/operating-systems/) |
| **OpenWeatherMap API key** | Free tier: [openweathermap.org/api](https://openweathermap.org/api) |

---

## ⚠️ Why the Old Guides Break

The original [rpi_clock preparation notes](https://github.com/texadactyl/rpi_clock/blob/master/docs/preparation_notes.txt) were written for **Raspbian Stretch (2018)** and the [Waveshare wiki](https://www.waveshare.com/wiki/3.5inch_RPi_LCD_%28A%29) has been updated but still contains legacy paths. Key changes:

| What changed | Old way | Current way |
|---|---|---|
| Boot config location | `/boot/config.txt` | `/boot/firmware/config.txt` (Bookworm) |
| Display driver | Legacy framebuffer (automatic) | KMS/DRM enabled by default — **must be disabled** for SPI LCDs on Pi 2/Zero |
| Display compositor | X11 default | Wayland default (Bookworm) — **must switch to X11** |
| Python packages | `pip install` globally | Virtual environments required (but we use system packages) |
| LCD driver scripts | `goodtft/LCD-show` | Waveshare `.dtbo` overlay + manual config (cleaner, less invasive) |

This script handles all of the above automatically.

---

## 🚀 Quick Start

### 1. Flash the OS

Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to flash **Raspberry Pi OS Lite (32-bit)** to your MicroSD card.

During setup in the Imager:
- ✅ Set hostname, username, and password
- ✅ Enable SSH
- ✅ Configure Wi-Fi (if applicable)

### 2. Boot and connect

Insert the SD card, power on the Pi, and connect via SSH:

```bash
ssh your_username@your_pi_ip
```

### 3. Download and run the script

```bash
wget https://raw.githubusercontent.com/tehmessiah75/RPI_Clock-install-script/main/setup-waveshare-clock.sh
chmod +x setup-waveshare-clock.sh
sudo ./setup-waveshare-clock.sh
```

The script will prompt you for:
- Your **OpenWeatherMap location** (e.g. `q=Adelaide,au`)
- Your **OpenWeatherMap API key**

### 4. Reboot

```bash
sudo reboot
```

The clock should appear on the Waveshare LCD within 30 seconds of boot.

---

## 📜 The Setup Script

Save this as `setup-waveshare-clock.sh`:

```bash
#!/usr/bin/env bash
set -e

# =====================================================
#  Waveshare 3.5" LCD (A) + rpi_clock
#  Pi 2B / Pi Zero v1.3 — Raspberry Pi OS Lite 32-bit
#
#  ✅ Runs from RAM (tmpfs) — extends SD card life
#  ✅ systemd service — clean start/stop
#  ✅ Auto-restart on crash
#  ✅ Hardware watchdog — reboots on full hang
# =====================================================

echo ""
echo "================================================"
echo "  Waveshare 3.5\" LCD + rpi_clock (SD-friendly)"
echo "  Pi 2B / Pi Zero v1.3 — Lite 32-bit"
echo "================================================"
echo ""

# ---- PRE-FLIGHT ----
if [ "$EUID" -ne 0 ]; then
  echo "Run with:  sudo bash $0"; exit 1
fi
if [ -z "$SUDO_USER" ] || [ "$SUDO_USER" = "root" ]; then
  echo "Run with sudo from your normal user."; exit 1
fi

USER_NAME="$SUDO_USER"
USER_HOME=$(eval echo "~$USER_NAME")

# Detect boot paths (Bookworm vs legacy)
if [ -f /boot/firmware/config.txt ]; then
  CONFIG_TXT="/boot/firmware/config.txt"
elif [ -f /boot/config.txt ]; then
  CONFIG_TXT="/boot/config.txt"
else
  echo "ERROR: Cannot find config.txt"; exit 1
fi

if [ -d /boot/firmware/overlays ]; then
  OVERLAYS_DIR="/boot/firmware/overlays"
elif [ -d /boot/overlays ]; then
  OVERLAYS_DIR="/boot/overlays"
else
  echo "ERROR: Cannot find overlays directory"; exit 1
fi

echo "Config:   $CONFIG_TXT"
echo "Overlays: $OVERLAYS_DIR"
echo "User:     $USER_NAME ($USER_HOME)"
echo ""

# =====================================================
# 1. UPDATE + INSTALL MINIMAL PACKAGES
# =====================================================
echo "[1/9] Installing packages..."
apt update && apt upgrade -y

apt install -y \
  xserver-xorg xserver-xorg-video-fbdev xinit \
  python3 python3-tk \
  git wget unzip \
  xinput-calibrator xserver-xorg-input-evdev

# =====================================================
# 2. INSTALL WAVESHARE OVERLAY
# =====================================================
echo "[2/9] Installing Waveshare 3.5\" LCD (A) overlay..."
cd /tmp
rm -f Waveshare35a.zip waveshare35a.dtbo 2>/dev/null
wget -q https://files.waveshare.com/wiki/common/Waveshare35a.zip
unzip -o Waveshare35a.zip
cp waveshare35a.dtbo "$OVERLAYS_DIR/"

# =====================================================
# 3. CONFIGURE DISPLAY (config.txt)
# =====================================================
echo "[3/9] Configuring display..."

# Disable KMS/FKMS overlays (incompatible with SPI LCD on Pi 2/Zero)
sed -i 's/^dtoverlay=vc4-kms-v3d/#dtoverlay=vc4-kms-v3d/' "$CONFIG_TXT"
sed -i 's/^dtoverlay=vc4-fkms-v3d/#dtoverlay=vc4-fkms-v3d/' "$CONFIG_TXT"

# Add config block (idempotent)
if ! grep -q "BEGIN WAVESHARE LCD" "$CONFIG_TXT"; then
  cat >> "$CONFIG_TXT" <<'CFGEOF'

# === BEGIN WAVESHARE LCD + WATCHDOG ===
dtparam=spi=on
dtoverlay=waveshare35a
hdmi_force_hotplug=1
hdmi_group=2
hdmi_mode=87
hdmi_cvt 480 320 60 6 0 0 0
hdmi_drive=2
display_rotate=0

# Disable KMS — required for SPI LCD on Pi 2/Zero
disable_fw_kms_setup=1

# Hardware watchdog — auto-reboot on full system hang
dtparam=watchdog=on
# === END WAVESHARE LCD + WATCHDOG ===
CFGEOF
fi

# =====================================================
# 4. SD CARD PROTECTION — RUN FROM RAM
# =====================================================
echo "[4/9] Configuring SD card protection..."

# 4a. Disable swap completely
if systemctl is-enabled dphys-swapfile 2>/dev/null; then
  dphys-swapfile swapoff 2>/dev/null || true
  dphys-swapfile uninstall 2>/dev/null || true
  systemctl disable dphys-swapfile 2>/dev/null || true
  echo "  → Swap disabled"
fi

# 4b. Add noatime to root partition (reduces inode write overhead)
if ! grep -q 'noatime' /etc/fstab; then
  sed -i '/\/ .*ext4/ s/defaults/defaults,noatime,commit=600/' /etc/fstab
  echo "  → Root filesystem: noatime + commit=600"
fi

# 4c. Add tmpfs mounts (conservative sizes for Pi Zero 512MB)
add_tmpfs() {
  local mount="$1" size="$2"
  if ! grep -q "tmpfs.*${mount} " /etc/fstab; then
    echo "tmpfs  ${mount}  tmpfs  defaults,noatime,nosuid,nodev,size=${size}  0  0" >> /etc/fstab
    echo "  → ${mount} → tmpfs (${size})"
  fi
}

add_tmpfs "/tmp" "50M"
add_tmpfs "/var/tmp" "10M"
add_tmpfs "/var/log" "20M"

# 4d. journald — volatile (RAM only), tiny footprint
mkdir -p /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/sd-protect.conf <<'EOF'
[Journal]
Storage=volatile
RuntimeMaxUse=8M
RuntimeMaxFileSize=2M
EOF
echo "  → journald: volatile (RAM only, 8M max)"

# 4e. Hardware watchdog via systemd (reboots on full hang)
if ! grep -q "^RuntimeWatchdogSec" /etc/systemd/system.conf; then
  sed -i 's/^#RuntimeWatchdogSec=.*/RuntimeWatchdogSec=14/' /etc/systemd/system.conf
  grep -q "^RuntimeWatchdogSec" /etc/systemd/system.conf || \
    echo "RuntimeWatchdogSec=14" >> /etc/systemd/system.conf
  echo "  → Hardware watchdog: 14s timeout"
fi

# =====================================================
# 5. CONFIGURE X11
# =====================================================
echo "[5/9] Configuring X11..."

# Allow service user to start X
cat > /etc/X11/Xwrapper.config <<'EOF'
allowed_users=anybody
needs_root_rights=yes
EOF

# Point X at SPI framebuffer (fb0 when KMS is disabled)
cat > /usr/share/X11/xorg.conf.d/99-fbdev.conf <<'EOF'
Section "Device"
    Identifier  "SPI LCD"
    Driver      "fbdev"
    Option      "fbdev" "/dev/fb0"
EndSection
EOF

# Touch calibration (evdev priority + base values)
cp /usr/share/X11/xorg.conf.d/10-evdev.conf \
   /usr/share/X11/xorg.conf.d/45-evdev.conf 2>/dev/null || true

if [ ! -f /usr/share/X11/xorg.conf.d/99-calibration.conf ]; then
  cat > /usr/share/X11/xorg.conf.d/99-calibration.conf <<'EOF'
Section "InputClass"
    Identifier      "calibration"
    MatchProduct    "ADS7846 Touchscreen"
    Option  "Calibration"   "3932 300 294 3801"
    Option  "SwapAxes"      "1"
EndSection
EOF
fi

# =====================================================
# 6. INSTALL RPI_CLOCK
# =====================================================
echo "[6/9] Installing rpi_clock..."

if [ -d "$USER_HOME/rpi_clock/.git" ]; then
  sudo -u "$USER_NAME" bash -c "cd '$USER_HOME/rpi_clock' && git pull --ff-only" || true
else
  sudo -u "$USER_NAME" git clone https://github.com/texadactyl/rpi_clock "$USER_HOME/rpi_clock"
fi

cp "$USER_HOME/rpi_clock/bin/sample_config.txt" \
   "$USER_HOME/rpi_clock/bin/rpi_clock.cfg"

# ---- Interactive config ----
echo ""
read -rp "OpenWeatherMap location (e.g. q=Adelaide,au): " LOCATION
read -rp "OpenWeatherMap API key: " APIKEY
echo ""

sed -i "s|^LOCATION.*|LOCATION = ${LOCATION}|"         "$USER_HOME/rpi_clock/bin/rpi_clock.cfg"
sed -i "s|^OWM_API_KEY.*|OWM_API_KEY = ${APIKEY}|"     "$USER_HOME/rpi_clock/bin/rpi_clock.cfg"
sed -i "s|^TEMP_UNITS.*|TEMP_UNITS = metric|"           "$USER_HOME/rpi_clock/bin/rpi_clock.cfg"
sed -i "s|^FLAG_WINDOWED.*|FLAG_WINDOWED = False|"      "$USER_HOME/rpi_clock/bin/rpi_clock.cfg"

chown -R "$USER_NAME:$USER_NAME" "$USER_HOME/rpi_clock"

# =====================================================
# 7. CREATE .xinitrc (what X runs)
# =====================================================
echo "[7/9] Creating .xinitrc..."

cat > "$USER_HOME/.xinitrc" <<'EOF'
#!/bin/bash

# Disable screen blanking + power saving
xset s off
xset -dpms
xset s noblank

# Run clock — logs to /tmp (tmpfs = RAM, no SD writes)
cd ~/rpi_clock/bin
python3 rpi_clock.py rpi_clock.cfg >> /tmp/rpi_clock.log 2>&1
EOF

chmod +x "$USER_HOME/.xinitrc"
chown "$USER_NAME:$USER_NAME" "$USER_HOME/.xinitrc"

# =====================================================
# 8. CREATE SYSTEMD SERVICE
# =====================================================
echo "[8/9] Creating systemd service..."

cat > /etc/systemd/system/rpi-clock.service <<EOF
[Unit]
Description=rpi_clock Kiosk Display
After=multi-user.target network.target
Wants=network.target

[Service]
Type=simple
User=${USER_NAME}
Environment=HOME=${USER_HOME}

# Start X on virtual terminal 1, run .xinitrc
ExecStart=/usr/bin/xinit ${USER_HOME}/.xinitrc -- :0 vt1

# Auto-restart on crash (10 second cooldown)
Restart=always
RestartSec=10
TimeoutStartSec=30

# Logging to journald (in RAM via volatile config)
StandardOutput=journal
StandardError=journal
SyslogIdentifier=rpi-clock

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable rpi-clock.service

# Free up tty1 for the kiosk service
systemctl disable getty@tty1.service 2>/dev/null || true

# =====================================================
# 9. CLEANUP OLD METHODS
# =====================================================
echo "[9/9] Cleaning up..."

# Remove any .bash_profile startx / FRAMEBUFFER lines from previous attempts
sed -i '/startx/d'       "$USER_HOME/.bash_profile" 2>/dev/null || true
sed -i '/FRAMEBUFFER/d'  "$USER_HOME/.bash_profile" 2>/dev/null || true
sed -i '/rpi_clock/d'    "$USER_HOME/.profile" 2>/dev/null || true

echo ""
echo "================================================"
echo "  ✅ SETUP COMPLETE"
echo "================================================"
echo ""
echo "  What happens on reboot:"
echo "    1. Pi boots to console (no desktop)"
echo "    2. systemd starts rpi-clock.service"
echo "    3. X starts on the Waveshare LCD"
echo "    4. rpi_clock shows clock + weather"
echo ""
echo "  SD card protection:"
echo "    /tmp         → RAM (50M tmpfs)"
echo "    /var/log     → RAM (20M tmpfs)"
echo "    /var/tmp     → RAM (10M tmpfs)"
echo "    Swap         → disabled"
echo "    journald     → volatile (RAM, 8M max)"
echo "    Root fs      → noatime, commit=600"
echo ""
echo "  Auto-recovery:"
echo "    App crash    → service restarts in 10s"
echo "    System hang  → hardware watchdog reboots"
echo ""
echo "  View live logs (in RAM):"
echo "    journalctl -u rpi-clock -f"
echo "    cat /tmp/rpi_clock.log"
echo ""
echo "  Touch calibration (after first boot):"
echo "    DISPLAY=:0 xinput_calibrator"
echo "    Update /usr/share/X11/xorg.conf.d/99-calibration.conf"
echo ""
echo "  Reboot now:  sudo reboot"
echo "================================================"
```

---

## 🔍 What the Script Does (Step by Step)

| Step | What | Why |
|---|---|---|
| **1** | `apt update && upgrade` + install minimal X11, Python 3, Tkinter, git | Smallest possible GUI stack — no desktop environment |
| **2** | Download `waveshare35a.dtbo` from Waveshare, copy to overlays | Device tree overlay tells Linux about the SPI LCD hardware |
| **3** | Edit `config.txt`: add SPI + LCD overlay, disable KMS, enable watchdog | KMS is incompatible with SPI LCDs on Pi 2/Zero; watchdog auto-reboots on hang |
| **4** | Disable swap, add tmpfs mounts, set journald to volatile, add `noatime` | **All writes go to RAM** — SD card only written on package installs/config changes |
| **5** | Configure Xwrapper, fbdev driver, touch calibration | Allows the service user to start X, points X at the SPI framebuffer |
| **6** | Clone rpi_clock, create config from template, set location/API key | The actual clock application |
| **7** | Create `.xinitrc` (disables screen blanking, launches the clock) | This is what X executes when it starts |
| **8** | Create `rpi-clock.service` with `Restart=always` | systemd manages the lifecycle — no login-based hacks |
| **9** | Remove old `.bash_profile` / `.profile` hacks | Clean slate if you ran previous setup attempts |

---

## 💾 SD Card Protection Details

All of these work together to **dramatically reduce writes** to the MicroSD card:

| Protection | Detail |
|---|---|
| **tmpfs /tmp** (50MB) | All temp files in RAM |
| **tmpfs /var/log** (20MB) | All log files in RAM |
| **tmpfs /var/tmp** (10MB) | Persistent temp in RAM |
| **Swap disabled** | No swap partition writes |
| **journald volatile** | System journal stored only in RAM (8MB max) |
| **noatime** | Filesystem doesn't update access timestamps on every read |
| **commit=600** | Filesystem syncs to disk every 10 minutes instead of every 5 seconds |

### ⚡ RAM usage on Pi Zero (512MB)

tmpfs allocations are **lazy** — they only use RAM when data is written. Typical usage:

| Mount | Allocated | Typical use |
|---|---|---|
| /tmp | 50M | ~2MB (rpi_clock log) |
| /var/log | 20M | ~5MB (journal + system) |
| /var/tmp | 10M | ~0MB |
| **Total** | **80M reserved** | **~7MB actual** |

This leaves ~400MB+ free for the OS and application — plenty for Pi Zero.

---

## 🔄 Auto-Recovery Layers

```
┌─────────────────────────────────────────┐
│  rpi_clock.py crashes                   │
│  → xinit exits → service exits          │
│  → systemd restarts in 10 seconds       │
├─────────────────────────────────────────┤
│  Whole system hangs (kernel/X freeze)   │
│  → systemd can't pet hardware watchdog  │
│  → BCM watchdog reboots Pi after 14s    │
├─────────────────────────────────────────┤
│  Power loss                             │
│  → Logs were in RAM (no SD corruption)  │
│  → noatime + commit=600 = fewer writes  │
│  → Pi boots clean, service starts again │
└─────────────────────────────────────────┘
```

---

## 🔧 Post-Install

### View live logs

```bash
# systemd journal (in RAM)
journalctl -u rpi-clock -f

# rpi_clock application log (in RAM)
cat /tmp/rpi_clock.log
```

### Touch calibration

The script installs default calibration values. To fine-tune for your specific screen:

```bash
# SSH in, then run:
DISPLAY=:0 xinput_calibrator
```

Copy the output values into:
```
/usr/share/X11/xorg.conf.d/99-calibration.conf
```

Then reboot.

### Service management

```bash
# Check status
sudo systemctl status rpi-clock

# Stop the clock
sudo systemctl stop rpi-clock

# Start the clock
sudo systemctl start rpi-clock

# Restart the clock
sudo systemctl restart rpi-clock

# Disable autostart
sudo systemctl disable rpi-clock

# Re-enable autostart
sudo systemctl enable rpi-clock
```

### Screen rotation

Edit `config.txt` and change `display_rotate=`:

| Value | Rotation |
|---|---|
| `0` | 0° (landscape, default) |
| `1` | 90° |
| `2` | 180° |
| `3` | 270° |

Then reboot. You may need to re-calibrate touch after rotating.

### Edit clock configuration

```bash
nano ~/rpi_clock/bin/rpi_clock.cfg
sudo systemctl restart rpi-clock
```

Key settings (from [sample_config.txt](https://github.com/texadactyl/rpi_clock/blob/master/bin/sample_config.txt)):

| Key | Example | Notes |
|---|---|---|
| `LOCATION` | `q=Adelaide,au` | [OWM format](https://openweathermap.org/current) |
| `OWM_API_KEY` | `abc123...` | Free tier works |
| `TEMP_UNITS` | `metric` | `metric` / `imperial` / `Kelvin` |
| `FORMAT_DATE` | `%%d/%%m/%%Y` | Python strftime (double `%%`) |
| `FORMAT_TIME` | `%%H:%%M %%Z` | Python strftime (double `%%`) |
| `BG_COLOR_ROOT` | `blue` | Background colour |
| `FG_COLOR_NORMAL` | `white` | Text colour |

---

## 🌐 Connectivity Note (Pi 2B / Pi Zero v1.3)

Neither the Pi 2B nor the Pi Zero v1.3 has built-in Wi-Fi or Bluetooth.

| Pi Model | Internet options |
|---|---|
| **Pi 2 Model B** | Ethernet port (built-in) **or** USB Wi-Fi dongle |
| **Pi Zero v1.3** | USB OTG adapter + USB Wi-Fi dongle **or** USB OTG adapter + USB Ethernet adapter |

> **Without internet:** The clock still displays the correct date and time (via the system clock), but weather data will not update until connectivity is restored.

---

## 🐛 Troubleshooting

### LCD stays white / no display after reboot

- **Check overlay:** `ls /boot/firmware/overlays/waveshare35a.dtbo` (or `/boot/overlays/`)
- **Check config.txt:** `cat /boot/firmware/config.txt | grep waveshare`
- **Check SPI is enabled:** `ls /dev/spidev*` — you should see `spidev0.0` and `spidev0.1`
- **Check KMS is disabled:** Make sure `dtoverlay=vc4-kms-v3d` is commented out

### Service won't start

```bash
# Check service status
sudo systemctl status rpi-clock

# Check full logs
journalctl -u rpi-clock --no-pager -n 50
```

### HDMI not working after setup

This is expected — the SPI LCD becomes the primary display. To re-enable HDMI temporarily:

1. Remove the SD card, mount on another computer
2. Edit `config.txt` — comment out the `# === BEGIN WAVESHARE LCD` block
3. Re-insert SD card and boot with HDMI

### Clock shows but no weather data

- Check internet connectivity: `ping -c 3 api.openweathermap.org`
- Check your API key: `curl "https://api.openweathermap.org/data/2.5/weather?q=Adelaide&APPID=YOUR_KEY"`
- Check the log: `cat /tmp/rpi_clock.log`

### Touch not responding or inverted

Run calibration:
```bash
DISPLAY=:0 xinput_calibrator
```

Then update `/usr/share/X11/xorg.conf.d/99-calibration.conf` with the new values.

---

## 🙏 Credits & References

| Resource | Link |
|---|---|
| **rpi_clock** (Richard Elkins) | [github.com/texadactyl/rpi_clock](https://github.com/texadactyl/rpi_clock) |
| **rpi_clock preparation notes** | [docs/preparation_notes.txt](https://github.com/texadactyl/rpi_clock/blob/master/docs/preparation_notes.txt) |
| **Waveshare 3.5" LCD (A) Wiki** | [waveshare.com/wiki/3.5inch_RPi_LCD_(A)](https://www.waveshare.com/wiki/3.5inch_RPi_LCD_%28A%29) |
| **Waveshare Bookworm simplified setup** (caliston) | [github.com/caliston/waveshare-LCD35-bookworm](https://github.com/caliston/waveshare-LCD35-bookworm) |
| **Waveshare 3.2/3.5 Trixie driver** (katzenjens) | [github.com/katzenjens/lcd32](https://github.com/katzenjens/lcd32) |
| **Raspberry Pi config.txt documentation** | [raspberrypi.com/documentation/computers/config_txt](https://www.raspberrypi.com/documentation/computers/config_txt.html) |
| **SD card write reduction guide** (Chris Dzombak) | [dzombak.com/blog/2024/04/pi-reliability-reduce-writes](https://www.dzombak.com/blog/2024/04/pi-reliability-reduce-writes-to-your-sd-card/) |
| **OpenWeatherMap API** | [openweathermap.org/api](https://openweathermap.org/api) |

---

## 📄 License

This setup script is provided under the [GPL-3.0 License](https://www.gnu.org/licenses/gpl-3.0.en.html), consistent with the [rpi_clock project license](https://github.com/texadactyl/rpi_clock/blob/master/LICENSE).

---

*Last updated: 27 May 2026*
