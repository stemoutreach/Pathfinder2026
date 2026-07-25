# robot Pi OS Build Steps

**Purpose:** Create an SD card image for a Pathfinder2026 workshop robot.
**Platform:** Raspberry Pi 4 (4GB+)
**OS:** Raspberry Pi OS (64-bit), Debian 13 (Trixie)
**Image type:** Lite recommended for robot-only images; Desktop is OK if VNC is needed
**Raspberry Pi Imager:** 2.0.0
**Current Pi OS Released:** 2026-06-18
**Kernel version:** 6.18
**Last tested:** 2026-06-18
**Time:** ~30 minutes

> **OS Note:** Use the Raspberry Pi OS 64-bit release dated **2026-06-18**. Raspberry Pi lists this image as Debian 13 (Trixie) with kernel 6.18. This guide was last tested on **2026-06-18**.

---

## Overview

After completing these steps you will have an SD card that can be cloned for all workshop robots. Each robot gets an identical image; WiFi credentials are included, and the robot IP is confirmed at event time.

This guide is only for building the robot OS image. Pi 500-to-robot connection, VS Code Remote SSH, and hardware wiring tests happen during the workshop in [Connect and Test](CONNECT_AND_TEST.md).

**Materials:** see [BOM_TEAM_KIT.md](BOM_TEAM_KIT.md) in this folder for the robot/setup BOM.

---

## Step 1: Flash Base OS

### Download
- Go to https://www.raspberrypi.com/software/
- Download **Raspberry Pi Imager 2.0.0**

### Flash Settings
- **OS:** Raspberry Pi OS (64-bit), released 2026-06-18
  - Lite (no desktop) is recommended for robot-only images
  - Desktop is fine if students will use VNC
- **Storage:** Select your SD card (16GB minimum, 32GB recommended)

### Pre-Configure in Imager (gear icon / Ctrl+Shift+X)
- **Device name / hostname:** any event label you prefer; use IP addresses for all event connections
- **Enable SSH:** Yes (password authentication)
- **Username:** `robot`
- **Password:** choose your own workshop password
- **WiFi:** Configure your workshop network SSID and password
- **Locale:** Set timezone and keyboard layout

> **Event account note:** Event notes may refer to the user name as "robot"; keep the Linux account lowercase as `robot` so paths and commands match `/home/robot/pathfinder`.

### Flash
Click **Write** and wait for completion.

> **Locale tip:** If you set locale to `en_US` during imaging but see locale warnings over SSH, the locale may not be fully generated. Fix with:
> ```bash
> sudo sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
> sudo locale-gen en_US.UTF-8
> sudo update-locale LANG=en_US.UTF-8
> ```

---

## Step 2: Boot and SSH For Image Build

1. Insert SD card into Pi 4
2. Power on
3. Wait ~60 seconds for first boot (filesystem expands)
4. Find the Pi's IP address on your network:
   ```bash
   # On the robot, if you have a monitor attached
   hostname -I

   # Or check your router / DHCP client list
   ```
5. SSH in from your image-build computer using the robot IP address:
   ```bash
   ssh robot@<ROBOT_IP>
   ```

## Step 3: System Updates

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

---

## Step 4: Enable Interfaces

Enable SSH, VNC, I2C, and the UART configuration needed by the robot control board.

### Confirm SSH

SSH should already be enabled from Raspberry Pi Imager. If needed, enable it manually:

```bash
sudo raspi-config nonint do_ssh 0
```

### Enable VNC

```bash
sudo raspi-config nonint do_vnc 0
```

### Enable I2C (for motor board and sonar)
```bash
sudo raspi-config nonint do_i2c 0
```

### Enable UART
The motor board communicates via UART at 1,000,000 baud. On Pi 4, Bluetooth uses the good UART (`ttyAMA0`) by default. Enable UART and add the Bluetooth overlay so `ttyAMA0` is available for the motor board.

Open the boot config file:

```bash
sudo nano /boot/firmware/config.txt
```

In nano, find the `[all]` section and ensure these lines are present:
```
[all]
enable_uart=1

# Pathfinder2026: Free ttyAMA0 for motor board (1Mbaud)
dtoverlay=disable-bt
```

Save and exit nano:

1. Press `Ctrl+O` to write the file.
2. Press `Enter` to confirm the file name.
3. Press `Ctrl+X` to exit nano.

### Disable Bluetooth Services
After you are back at the command line, disable the Bluetooth services:

```bash
sudo systemctl disable hciuart bluetooth
sudo systemctl stop bluetooth
```

### Disable Serial Console on ttyAMA0

By default, the kernel and systemd both attach a serial console to `ttyAMA0`. This conflicts with the motor board â the port must be completely free for 1Mbaud communication.

```bash
# Remove the serial console from kernel boot parameters
sudo sed -i 's/console=serial0,[0-9]* //' /boot/firmware/cmdline.txt

# Disable the serial login prompt (getty)
sudo systemctl disable serial-getty@ttyAMA0.service
```

### Reboot
```bash
sudo reboot
```

### Verify After Reboot
```bash
# UART available (should point to ttyAMA0, NOT ttyS0)
ls -la /dev/serial0
# Should show: /dev/serial0 -> ttyAMA0

# ttyAMA0 exists
ls /dev/ttyAMA0
# Should show: /dev/ttyAMA0

# I2C available
ls /dev/i2c-1
# Should show: /dev/i2c-1

# Bluetooth inactive
systemctl is-active bluetooth
# Should show: inactive

# Serial console removed from ttyAMA0
systemctl is-active serial-getty@ttyAMA0
# Should show: inactive

# ttyAMA0 not held by any process
fuser /dev/ttyAMA0
# Should return nothing (no output)
```

> **Note:** Before adding `dtoverlay=disable-bt`, `/dev/serial0` points to `ttyS0`. After adding it and rebooting, it correctly points to `ttyAMA0`. The motor board requires `ttyAMA0`.

---

## Step 5: Install Python Dependencies

### System Packages
```bash
sudo apt-get install -y python3-pip python3-dev i2c-tools git python3-opencv python3-pygame joystick
```

> **Trixie note:** `python3-opencv` (4.10.0) is available via apt on Debian 13 Trixie and is the recommended install method â no pip needed for OpenCV.

`python3-pygame` and `joystick` support Phase 2 gamepad control with the Logitech F710.

### Python Packages
```bash
pip3 install --break-system-packages \
    pupil-apriltags \
    flask \
    pyserial \
    smbus2 \
    pyyaml \
    numpy
```

**Note:** `--break-system-packages` is required on Trixie/Bookworm (PEP 668). `numpy` and `flask` may already be present; pip will skip them if current.

If pip warns that `flask` was installed in `/home/robot/.local/bin` and that directory is not on `PATH`, that is OK for Pathfinder. The workshop code imports Flask from Python; it does not need to run the `flask` command directly. If pip reports a `types-flask-migrate` dependency conflict, ignore it unless the import checks below fail. Pathfinder does not use Flask-Migrate.

### Verify Installation
```bash
python3 -c "import cv2; print(f'OpenCV: {cv2.__version__}')"
python3 -c "from pupil_apriltags import Detector; print('AprilTags: OK')"
python3 -c "import flask; print('Flask: OK')"
python3 -c "import pygame; print('pygame: OK')"
python3 -c "import serial; print('PySerial: OK')"
python3 -c "import smbus2; print('SMBus2: OK')"
python3 -c "import yaml; print('PyYAML: OK')"
python3 -c "import numpy; print(f'NumPy: OK')"
```

All should print without errors.

> **Flask version warning:** Flask 3.1+ shows a deprecation warning when accessing `flask.__version__`. This is harmless â Flask is working correctly.

---

## Step 6: Install Pathfinder2026

### Clone Repository
```bash
mkdir -p /home/robot/pathfinder
cd /home/robot/pathfinder
git clone https://github.com/stemoutreach/Pathfinder2026.git .
```

### Verify Clone
```bash
ls skills/
# Should show skill files including strafe_nav.py, block_detect.py, etc.
```

### Create Team Code Starter Folder
```bash
mkdir -p /home/robot/team_code
cp -a /home/robot/pathfinder/team_code_starters/. /home/robot/team_code/
```

This folder gives students a safe place to edit code during the event. Keep `/home/robot/pathfinder` as the official updateable repo.

The starter folder should include:

```text
/home/robot/team_code/README.md
/home/robot/team_code/drive_practice.py
/home/robot/team_code/arm_practice.py
```

### Verify All Imports
```bash
cd /home/robot/pathfinder
python3 -c "
from lib.board import get_board, PLATFORM
print(f'Platform: {PLATFORM}')
from skills.block_detect import BlockDetector
from skills.strafe_nav import StrafeNavigator
print('All imports OK!')
"
```

Should print:
```
Platform: pi4
All imports OK!
```

---

## Step 7: Verify User Permissions

The `robot` user should already have the correct groups from Raspberry Pi Imager. Verify:

```bash
groups robot
```

Must include: `dialout` `i2c` `gpio` `video`

If any are missing:
```bash
sudo usermod -a -G dialout,i2c,gpio,video robot
# Then logout and login again
```

**No sudo needed to run robot code** if groups are correct.

---

## Step 8: Enable Startup Service

`start_robot.py` runs automatically at boot and verifies all hardware is ready.

**Checks performed:**
- **Board** -- motor controller connected
- **Battery** -- voltage with good / caution / LOW status
- **Arm** -- moves to camera-forward position, gripper open
- **Sonar** -- reads distance, LEDs show distance-zone color
- **Camera** -- opens and captures a test frame

**Feedback (no screen needed):**
- **All clear:** 2 quick beeps + green LEDs for 5 seconds then off
- **Needs attention:** 5 slow beeps + red LEDs stay on until next boot

### Install the Service

The installer copies the included service file, reloads systemd, and enables the service.

```bash
cd /home/robot/pathfinder
sudo bash scripts/setup/install_services.sh
```

### Check Last Run

```bash
journalctl -u pathfinder-startup.service
```

Do not use this OS build guide for event hardware testing. Battery, motor, servo, sonar, camera, SSH, and Pi 500 connection checks are in [Connect and Test](CONNECT_AND_TEST.md).

---

## Step 9: Clone the Image

Once one SD card is fully configured, clone it for all robots:

### On Linux/Mac
```bash
# Read from SD card
sudo dd if=/dev/sdX of=Pathfinder2026.img bs=4M status=progress

# Write to new SD card
sudo dd if=Pathfinder2026.img of=/dev/sdY bs=4M status=progress
```

### On Windows
Use **Win32 Disk Imager** or **balenaEtcher** to read/write the image.

### Per-robot changes after cloning
Robot IPs come from the workshop network DHCP and usually stay sticky. Confirm the robot IP during event setup instead of setting fixed addresses before the event.

Optional: set a unique device label for local identification, but do not use it for event connections.

WiFi credentials should already be set from Step 1. If different networks are needed:
```bash
sudo nmcli dev wifi connect "SSID" password "PASSWORD"
```

> **Trixie note:** Debian 13 Trixie uses NetworkManager by default. `wpa_supplicant.conf` may not be the right place to set WiFi â use `nmcli` or the desktop network manager instead.

---

## Quick Reference: What's Installed

| Component | Version | Purpose |
|-----------|---------|---------|
| Raspberry Pi OS | Debian 13 Trixie 64-bit, released 2026-06-18 | Base operating system |
| Kernel | 6.18 | Linux kernel |
| Python | 3.13.5 | Programming language |
| OpenCV | 4.10.0 | Computer vision (via apt) |
| pupil-apriltags | 1.0.4 | AprilTag detection |
| Flask | 3.1.1 | Web control interface |
| PySerial | 3.5 | Serial communication (Pi 5) |
| SMBus2 | 0.4.3 | I2C communication (Pi 4) |
| PyYAML | 6.0.2 | Configuration files |
| NumPy | 2.2.4 | Math operations |
| pygame | apt package | Gamepad input |
| joystick | apt package | `/dev/input/js*` gamepad tools |
| Pathfinder2026 | Latest | robot framework |

## Quick Reference: Hardware Interfaces

| Interface | Device | Config |
|-----------|--------|--------|
| UART (motor board) | `/dev/ttyAMA0` | `dtoverlay=disable-bt` |
| I2C (motor board) | `/dev/i2c-1` addr `0x7A` | `dtparam=i2c_arm=on` |
| I2C (sonar) | `/dev/i2c-1` addr `0x77` | Same bus |
| Camera | `/dev/video0` | USB camera |
| GPIO (buzzer) | Pin 31 | BOARD numbering |

---

## Troubleshooting

### UART not available / serial0 â ttyS0
- Bluetooth is still active â verify `dtoverlay=disable-bt` is in `/boot/firmware/config.txt` under `[all]`
- Verify `systemctl is-active bluetooth` returns `inactive`
- Must reboot after config change
- After fix: `ls -la /dev/serial0` should show `-> ttyAMA0`

### Motor board not responding / ttyAMA0 held at boot
Even after disabling Bluetooth, the serial console may still hold `ttyAMA0`. This is a separate issue â the kernel attaches a console to the port and systemd starts a login prompt on it.

Check:
```bash
fuser /dev/ttyAMA0                          # shows PID if port is held
systemctl is-active serial-getty@ttyAMA0   # should be inactive
grep 'serial0' /boot/firmware/cmdline.txt  # should return nothing
```

Fix:
```bash
sudo sed -i 's/console=serial0,[0-9]* //' /boot/firmware/cmdline.txt
sudo systemctl disable --now serial-getty@ttyAMA0.service
sudo reboot
```

### Permission denied on I2C
- User must be in `i2c` group: `groups robot`
- Fix: `sudo usermod -a -G i2c robot` then logout/login

### Locale warnings over SSH
- Harmless but can cause `perl` warnings in apt output
- Fix:
  ```bash
  sudo sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
  sudo locale-gen en_US.UTF-8
  sudo update-locale LANG=en_US.UTF-8
  ```
- Then re-login

### Pip warning about Flask path or types-flask-migrate
- `WARNING: The script flask is installed in '/home/robot/.local/bin' which is not on PATH` is OK for this workshop.
- `types-flask-migrate requires Flask-SQLAlchemy` is also OK unless the import checks fail.
- Verify with: `python3 -c "import flask; print('Flask: OK')"`
- Do not install extra Flask database packages just for this warning.

### WiFi config on Trixie
- Debian 13 uses NetworkManager, not wpa_supplicant
- Use `nmcli dev wifi connect "SSID" password "PASSWORD"` or the desktop GUI

---

*Created: March 26, 2026*
*Updated: July 24, 2026 - narrowed to robot OS image build steps; event connection and hardware tests moved to Connect and Test*
*Last tested: June 18, 2026*
*Tested on: Raspberry Pi 4 Model B, Raspberry Pi OS 64-bit, Debian 13 Trixie, kernel 6.18, Python 3.13.x*
