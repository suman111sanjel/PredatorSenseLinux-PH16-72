# Unofficial Linux Kernel Module for Acer Gaming RGB Keyboard Backlight and Turbo Mode (Acer Predator , Nitro)
The code base is still in its early stages, as I’ve just started working on developing this kernel module. It's a bit messy at the moment, but I’m hopeful that, with your help, we can collaborate to improve its structure and make it more organized over time.

Inspired by [acer-predator-turbo](https://github.com/JafarAkhondali/acer-predator-turbo-and-rgb-keyboard-linux-module), which has a similar goal, this project was born out of my own challenges. I faced issues detecting the Turbo key and ended up using [acer_wmi](https://github.com/torvalds/linux/blob/master/drivers/platform/x86/acer-wmi.c), but it lacked key features like RGB , custom fan support, battery limiter, and more. As a result, I decided to implement these missing features in my own project.

---

> ## 🖥️ Acer Predator PH16-72 on Ubuntu 24.04 — start here
>
> **➡️ [SETUP.md](SETUP.md) — complete, tested, fresh-install guide.**
>
> The generic instructions below target Arch and kernels 6.12–6.14. On a PH16-72 running
> Ubuntu 24.04 (kernel 6.8) they are not enough on their own, because:
>
> - upstream tracks kernel **6.14+**, so the build fails on 6.8 without the compatibility
>   patch in [`patches/0001-ph16-72-kernel-6.8-support.patch`](patches/0001-ph16-72-kernel-6.8-support.patch);
> - upstream has **no `Predator PH16-72` DMI entry**, so even a successful build leaves the
>   laptop detected as `UNKNOWN` and nothing works;
> - the PH16-72 keyboard is **per-key RGB over USB HID**, not a four-zone EC function, so
>   `four_zoned_kb` never appears and this module cannot drive the keyboard at all.
>
> [SETUP.md](SETUP.md) covers all three, plus the `predator` CLI and the optional DAMX GUI.
>
> | Guide | Covers |
> |---|---|
> | **[SETUP.md](SETUP.md)** | **fresh install, start to finish — read this first** |
> | [PREDATOR-SCRIPT.md](PREDATOR-SCRIPT.md) | the `predator` CLI: fans, thermal mode, keyboard colour and patterns |
> | [docs/KEYBOARD-GUIDE.md](docs/KEYBOARD-GUIDE.md) | per-key RGB deep-dive, LampArray protocol, troubleshooting |
> | [docs/DAMX-INSTALL.md](docs/DAMX-INSTALL.md) | the DAMX graphical front-end |
>
> Verified on Ubuntu 24.04.5 LTS, kernel 6.8.0-136-generic, Secure Boot off.

---

## 🚀 Installation
To begin, identify your current kernel version:
```bash
uname -r
```

Install the appropriate Linux headers based on your kernel version. This module has been tested with kernel version (6.12,6.13 ([previous code base](https://github.com/0x7375646F/Linuwu-Sense/tree/v6.13)),6.14) zen. 
For Arch Linux:
```bash
sudo pacman -S linux-headers
```
Next, clone the repository and build the module:
```bash
git clone https://github.com/0x7375646F/Linuwu-Sense.git
cd Linuwu-Sense
make install
```
The make command will remove the current acer_wmi module and load the patched version.

To Uninstall:
```bash
make uninstall
```

---

### 🐧 Ubuntu 24.04 LTS — detailed walkthrough

#### Step 0: Check your kernel version first

```bash
uname -r
```

This matters more on Ubuntu than on Arch. Ubuntu 24.04 ships the **6.8 GA kernel**, which is *older* than the 6.12–6.14 range this module targets. Building against 6.8 unmodified fails immediately with:

```
fatal error: linux/unaligned.h: No such file or directory
```

You have two ways forward:

| Option | Kernel | Notes |
| --- | --- | --- |
| **A — Upgrade the kernel** (recommended) | 6.14 via HWE | Matches what upstream tests against. Requires a reboot. |
| **B — Stay on 6.8 GA** | 6.8 | Only works on a tree carrying 6.8 compatibility guards. Without them the build fails — see [Troubleshooting](#troubleshooting). |

#### Step 1: Install build dependencies

```bash
sudo apt update
sudo apt install -y build-essential git linux-headers-$(uname -r)
```

`build-essential` provides `gcc` and `make`; the headers package must match your **running** kernel exactly, which is why `$(uname -r)` is used rather than a hardcoded version.

#### Step 2 (Option A only): Move to the HWE kernel

```bash
sudo apt install -y linux-generic-hwe-24.04
sudo reboot
```

After rebooting, confirm you are on the newer kernel and install its headers:

```bash
uname -r
sudo apt install -y linux-headers-$(uname -r)
```

#### Step 3: Check Secure Boot

```bash
mokutil --sb-state
```

- **`SecureBoot disabled`** — nothing to do, continue to Step 4.
- **`SecureBoot enabled`** — the kernel will refuse to load an unsigned module, failing with `Key was rejected by service`. Either generate and enroll a MOK key by following [`module_signing_readme`](module_signing_readme), or disable Secure Boot in your BIOS/UEFI settings.

The `Makefile` signs the module automatically if it finds `~/module-signing/MOK.priv` and `MOK.der`, and prints `MOK keys not found ... Skipping module signing` when it does not. That message is harmless if Secure Boot is off.

#### Step 4: Clone, build, and install

```bash
git clone https://github.com/0x7375646F/Linuwu-Sense.git
cd Linuwu-Sense
make install
```

`make install` needs `sudo` internally and changes your system in several persistent ways, so it is worth knowing what it does:

- Unloads the in-tree `acer_wmi` module and **permanently blacklists** it via `/etc/modprobe.d/blacklist-acer_wmi.conf` (the two modules claim the same WMI GUIDs and cannot coexist)
- Installs `linuwu_sense.ko` into `/lib/modules/$(uname -r)/kernel/drivers/platform/x86`
- Enables load-at-boot through `/etc/modules-load.d/linuwu_sense.conf`
- Installs and enables `linuwu_sense.service`, which restores your fan and thermal settings after a reboot
- Creates a `linuwu_sense` group, adds your user to it, and writes `/etc/tmpfiles.d/` rules so the sysfs controls are group-writable without `sudo`

All of it is reversible with `make uninstall`.

#### Step 5: Apply your new group membership

Group changes do not affect your current login session. Either log out and back in, or start a subshell:

```bash
newgrp linuwu_sense
```

Until you do this, writing to the sysfs files still requires `sudo`.

#### Step 6: Verify the install

```bash
lsmod | grep linuwu_sense
ls /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/
cat /sys/firmware/acpi/platform_profile_choices
systemctl status linuwu_sense.service
```

A healthy install shows a **`predator_sense`** directory (Predator models) or **`nitro_sense`** (Nitro models), plus `hwmon` for fan and temperature readings, and a `four_zoned_kb` directory on four-zone RGB keyboards. `platform_profile_choices` should list your supported thermal profiles, for example:

```
low-power quiet balanced balanced-performance performance
```

Check `sudo dmesg | grep linuwu_sense` for `Platform profile registered successfully` to confirm thermal control came up.

#### ⚠️ Rebuild after every kernel update

This module does **not** use DKMS, and Ubuntu ships kernel updates regularly. After any update that changes your kernel, the module will not load until you rebuild it against the new version:

```bash
sudo apt install -y linux-headers-$(uname -r)
cd /path/to/Linuwu-Sense
make install
```

Consider holding kernel upgrades if you would rather not repeat this, or add the rebuild to your post-upgrade routine.

#### Troubleshooting

**`fatal error: linux/unaligned.h: No such file or directory`**
Your kernel is older than 6.12. This header was renamed from `asm/unaligned.h` in 6.12. Follow Option A in Step 0, or use a tree with 6.8 compatibility guards.

**Errors mentioning `platform_profile_ops`, `devm_platform_profile_register`, `BACKLIGHT_POWER_ON`, `wmi_install_notify_handler`, or `.remove`**
Same root cause as above — these APIs all changed between 6.8 and 6.14. Upgrading the kernel resolves all of them at once.

**The module loads, but no `predator_sense` or `nitro_sense` directory appears**
The module loaded but your laptop is not in the driver's DMI quirk table, so no features were enabled. Check what your machine reports:

```bash
cat /sys/class/dmi/id/product_name
```

If that model is absent from `acer_quirks[]` in [`src/linuwu_sense.c`](src/linuwu_sense.c), it needs a new entry. Please open an issue or PR including your `product_name` and laptop model — this is the most common reason the module appears to install successfully yet does nothing.

**`modprobe: ERROR: could not insert 'linuwu_sense': Key was rejected by service`**
Secure Boot is enabled and the module is unsigned. See Step 3.

**`make: *** /lib/modules/6.8.0-XX-generic/build: No such file or directory`**
The headers for your running kernel are missing. Re-run Step 1. If you just upgraded your kernel but have not rebooted, either reboot first or the headers will not match `uname -r`.

**Thermal profile or Turbo key does nothing**
Confirm `acer_wmi` is actually blacklisted and not loaded, since it conflicts with this module:

```bash
lsmod | grep acer_wmi
```

That should return nothing. If it is loaded, run `sudo rmmod acer_wmi` and verify `/etc/modprobe.d/blacklist-acer_wmi.conf` exists.

---

> **⚠️ Warning!**
> ## Use at your own risk! This driver is independently developed through reverse engineering the official PredatorSense app, without any involvement from Acer. It interacts with low-level WMI methods, which may not be tested across all models.

## 🛠️ Usage
# Example Usage and Configuration

Thermal profiles can be easily switched with a single click! 😎 For battery mode, you can choose between Eco and Balanced, while when plugged into AC, you have the options for Quiet, Balanced, Performance, and Turbo. ⚡💻 Each profile will be different for battery and AC, and the thermal and fan settings will automatically adjust based on your current power source. Customize it to fit your preferences! 🌟

---

For **Predator** laptops, the following path is used: `/sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense`

For **Nitro** laptops, the following path is used: `/sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/nitro_sense`

predator_sense – This directory includes all the features, excluding the custom boot logo functionality.
four_zoned_kb – If your keyboard is four-zoned, this directory provides support for it. Unfortunately, there is no support for per-key RGB keyboards.
Here is how to interact with the Virtual Filesystems (VFS) mounted in this path:

### **0. Thermal Profiles (Nitro users especially who don't have switch key) 🚀**

Some acer nitro laptops don't come up with the thermal profile switch button in this case we manually need to set it:

To probe the current thermal profile:

`cat /sys/firmware/acpi/platform_profile`

To check the supported thermal profile:

`cat /sys/firmware/acpi/platform_profile_choices`

To switch the platform profile:

`echo balanced | sudo tee /sys/firmware/acpi/platform_profile`

Replace the balanced with the supported profile you have.

#### **1. Backlight Timeout ⏰**

This feature turns off the keyboard RGB after 30 seconds of idle mode.

- **0** – Disabled
- **1** – Enabled

To check the current status, use:

`cat /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/backlight_timeout`

To change the state, use:

`echo 1 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/backlight_timeout`

---

#### **2. Battery Calibration 🔋**

This function calibrates your battery to provide a more accurate percentage reading. It involves charging the battery to 100%, draining it to 0%, and recharging it back to 100%. **Do not unplug the laptop from AC power during calibration.**

- **1** – Start calibration
- **0** – Stop calibration

To check the current status:

`cat /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/battery_calibration`

To change the state:

`echo 1 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/battery_calibration`

---

#### **3. Battery Limiter ⚡**

Limits battery charging to 80%, preserving battery health for laptops primarily used while plugged into AC power.

- **1** – Enabled
- **0** – Disabled

To check the current status:

`cat /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/battery_limiter`

To change the state:

`echo 1 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/battery_limiter`

---

#### **4. Boot Animation Sound 🎶**

Enables or disables custom boot animation and sound.

- **1** – Enabled
- **0** – Disabled

To check the current status:

`cat /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/boot_animation_sound`

To change the state:

`echo 0 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/boot_animation_sound`

---

#### **5. Fan Speed  🌬️**

Controls the CPU and GPU fan speeds.

- **0** – Auto
- **1** – Minimum fan speed (not recommended)
- **100** – Maximum fan speed
- Other values like **50, 55, 70** can be set according to your preference.

Example (set CPU to 50 and GPU to 70):

`echo 50,70 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/fan_speed`

---

#### **6. LCD Override 🖥️**

Reduces LCD latency and minimizes ghosting.

- **1** – Enabled
- **0** – Disabled

To check the current status:

`cat /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/lcd_override`

To change the state:

`echo 1 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/lcd_override`

---

#### **7. USB Charging ⚡**

Allows the USB charging port to provide power even when the laptop is off.

- **0** – Disabled
- **10** – Provides power until battery reaches 10%
- **20** – Provides power until battery reaches 20%
- **30** – Provides power until battery reaches 30%

To check the current status:

`cat /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/usb_charging`

To change the state:

`echo 20 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/usb_charging`

---
## 💻 Keyboard Configuration 
### **Directory: `four_zoned_kb`**

The `four_zoned_kb` directory contains two Virtual File Systems (VFS) that control the RGB backlight behavior of the four-zone keyboard:

1. **`four_zone_mode`**
2. **`per_zone_mode`**

#### **1. Per-Zone Mode (`per_zone_mode`) 🎨**

This mode allows you to set a specific RGB color for each of the four keyboard zones individually. Each zone is represented by an RGB value in hexadecimal format (e.g., `4287f5` where `42` is Red, `87` is Green, and `f5` is Blue).

- **Parameters:**
    
    - The `per_zone_mode` file accepts four parameters, one for each zone, separated by commas.
    - The `per_zone_mode` also accepts brightness value.
    - Each parameter represents the RGB value for a specific zone in the format `RRGGBB`.
- **Example:**

To set all four zones to the same color (`4287f5`) and brightness to full:

`echo 4287f5,4287f5,4287f5,4287f5,100 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/four_zoned_kb/per_zone_mode`

To set each zone with unique colors:

`echo 4287f5,ff5733,33ff57,ff33a6,100 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/four_zoned_kb/per_zone_mode`

When reading (`cat`) the `per_zone_mode` file, the current color values for each zone are displayed in the format:

`4287f5,4287f5,4287f5,4287f5,100`

This indicates the current RGB color for each of the four zones.

### **Four-Zone Mode (`four_zone_mode`) ✨**

The `four_zone_mode` controls advanced RGB effects for your keyboard, requiring seven parameters:

- **Parameters:**
    
    - **Mode (0-7):** Lighting effect type (e.g., static, breathing, wave).
    - **Speed (0-9):** Speed of the effect (if applicable).
    - **Brightness (0-100):** Intensity of the lighting effect.
    - **Direction (1-2):** Direction of the effect (1 = right to left, 2 = left to right).
    - **Red (0-255), Green (0-255), Blue (0-255):** RGB color values.
- **Modes:**
    
    - **0:** Static Mode – Fixed color, no animation.
    - **1:** Breathing Mode – Color fades in and out.
    - **2:** Neon Mode – Neon glow effect, fixed color (black), direction ignored.
    - **3:** Wave Mode – Wave-like effect, color transitions across the keyboard.
    - **4:** Shifting Mode – Shifting light effect, full control over speed, direction, and color.
    - **5:** Zoom Mode – Zoom effect, direction ignored.
    - **6:** Meteor Mode – Meteor-like effect, direction ignored.
    - **7:** Twinkling Mode – Twinkling light effect, direction ignored.
- **Example Command:**
    
    Set to **Neon Mode** with speed 1, full brightness, and top-to-bottom direction:
    
    `echo 3,1,100,2,0,0,0 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/four_zoned_kb/four_zone_mode`
    
    **Explanation:**
    
    - `3`: Neon Mode
    - `1`: Speed (1)
    - `100`: Full brightness
    - `2`: Direction (top to bottom)
    - `0`: Red (black for Neon)
    - `0`: Green (black for Neon)
    - `0`: Blue (black for Neon)
 
The thermal and fan profiles will be saved and loaded on each reboot, ensuring that the settings remain persistent across restarts.
## GUI:
- [Div Acer Manager Max By PXDiv](https://github.com/PXDiv/Div-Acer-Manager-Max)
- [GUI LinuwuSense By KumarVivek](https://github.com/kumarvivek1752/Linuwu-Sense-GUI/tree/main)

## 🚧 Roadmap:
- [x] GUI for keyboard rgb controls to make it noob friendly.
- [x] Module Persistence After Reboot.
- [ ] Custom Boot Logo Feature Support.
- [ ] More device support currently only ( PHN16-71 ) is fully supported.

## License
GNU General Public License v3

### 💖 Donations
Donations are completely optional but show your love for open-source development and motivate me to add more features to this project!
USDT (BEP20 - BNB Smart Chain): 0xDA7aa42B9Fc3041F20f4Ec828A70E9bDD54A6822
