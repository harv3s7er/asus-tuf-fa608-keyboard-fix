# ASUS TUF Gaming A16 (FA608UH) Keyboard Backlight Fix

This repository provides a permanent, native solution for fixing the keyboard backlight synchronization issue on the **ASUS TUF Gaming A16 (FA608UH)** laptop running Linux (Ubuntu 24.04+). 

## The Problem (Asusctl Issue #682)
On a cold boot or system startup, the built-in **ITE51368 (`0B05:19B6`)** I2C-HID keyboard controller drops into a deep hardware sleep state. Because the Linux kernel cannot wake it up natively, the keyboard backlight remains dead and tools like `asusctl` fail unless the user reboots directly from a Windows environment.

## The Solution
This fix utilizes a two-part sequence to regain complete control of the hardware:
1. **Wakes the hardware** out of Windows Dynamic Lighting mode via a raw volatile initialization feature packet (`70,1`) sent through `hidapitester`.
2. **Overrides the default rainbow loop** using native multi-parameter Linux kernel `sysfs` string attributes (`1 0 R G B Speed`).

---

## Installation & Setup

### 1. Prerequisites & Dependencies
Install the required system compilation tools and dependencies:
```bash
sudo apt update
sudo apt install -y build-essential git libusb-1.0-0-dev libudev-dev
```

### 2. Install hidapitester
Clone and compile the lightweight hardware utility:
```bash
git clone https://github.com
cd hidapitester
make
sudo cp hidapitester /usr/local/bin/
cd ..
```

### 3. Create the Master Controller Script
Create a centralized management script to wake the controller and apply multi-parameter profiles:
```bash
sudo nano /usr/local/bin/kb-control.sh
```
Paste the following code inside the file:
```bash
#!/bin/bash

# 1. Master Hardware Wake: Free the ITE device from Windows Dynamic mode
/usr/local/bin/hidapitester --vidpid 0B05/19B6 --open --send-feature 70,1 >/dev/null 2>&1
sleep 0.1

# Locate the sysfs kernel path
MODE_FILE="/sys/class/leds/asus::kbd_backlight/kbd_rgb_mode"
if [ ! -f "$MODE_FILE" ]; then
    MODE_FILE="/sys/devices/platform/asus-nb-wmi/leds/asus::kbd_backlight/kbd_rgb_mode"
fi

case "\$1" in
    white)
        echo "Setting keyboard to Solid White..."
        echo "1 0 255 255 255 0" | sudo tee "\$MODE_FILE" >/dev/null
        ;;
    green)
        echo "Setting keyboard to Solid Green..."
        echo "1 0 0 255 0 0" | sudo tee "\$MODE_FILE" >/dev/null
        ;;
    blue)
        echo "Setting keyboard to Solid Blue..."
        echo "1 0 0 0 255 0" | sudo tee "\$MODE_FILE" >/dev/null
        ;;
    red)
        echo "Setting keyboard to Solid Red..."
        echo "1 0 255 0 0 0" | sudo tee "\$MODE_FILE" >/dev/null
        ;;
    dynamic|rainbow)
        echo "Setting keyboard to Dynamic Rainbow Cycle..."
        echo "1 2 0 0 0 2" | sudo tee "\$MODE_FILE" >/dev/null
        ;;
    off)
        echo "Turning off keyboard backlight..."
        echo "1 0 0 0 0 0" | sudo tee "\$MODE_FILE" >/dev/null
        ;;
    *)
        echo "Usage: kb [white|green|blue|red|dynamic|off]"
        ;;
esac
```
Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`), then mark it executable:
```bash
sudo chmod +x /usr/local/bin/kb-control.sh
```

### 4. Configure Boot-Time Automation
Enable a system service to initialize the hardware immediately on system startup:
```bash
sudo nano /etc/systemd/system/wake-keyboard.service
```
Paste this configuration layout inside:
```ini
[Unit]
Description=Wake up ASUS TUF Keyboard Backlight (Issue 682)
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/kb-control.sh dynamic
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable the system layer service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable wake-keyboard.service
```

### 5. Configure Global Multi-User Shortcuts
To allow your user profile and specific users to trigger shortcuts password-free:

1. Allow execution without `sudo` prompt barriers:
   ```bash
   sudo visudo
   ```
2. Append your system administrative users to the very bottom:
   ```text
   alex ALL=(ALL) NOPASSWD: /usr/local/bin/kb-control.sh
   ```
3. Establish a global alias link across the shell configuration map:
   ```bash
   sudo nano /etc/bash.bashrc
   ```
4. Append the short-string handle line at the bottom:
   ```bash
   alias kb="sudo /usr/local/bin/kb-control.sh"
   ```

---

## Usage Controls
Open any active terminal interface session and directly invoke your profile handles:
* `kb white` - Switches to a static white panel layout.
* `kb green` - Locks down a steady solid green backdrop.
* `kb blue` - Applies pure solid blue backlighting.
* `kb red` - Drops into deep solid red mode.
* `kb dynamic` - Resumes the native multi-color factory rainbow wave.
* `kb off` - Instantly kills the LED light strings.

## Dual-Boot Behavior
This deployment targets volatile system hardware buffers exclusively. When you dual-boot or pivot directly back into Windows, the operational parameters drop back entirely to native factory profiles. Windows frameworks like Armoury Crate or G-Helper will retain default behaviors with zero configuration conflicts.
