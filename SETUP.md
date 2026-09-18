# Acer Predator PH16-72 on Ubuntu 24.04 — complete setup

Everything needed to go from a fresh Ubuntu 24.04 install to full hardware control on a
Predator Helios 16 PH16-72: fan speed, thermal profiles, battery limiter, and **per-key RGB**.

Follow §1 → §6 in order. Allow about 20 minutes, including one reboot.

> ### ⚠️ Read this first if you are setting up a *new* machine
>
> This guide tells you to clone `github.com/0x7375646F/Linuwu-Sense` and use files from it —
> `patches/`, `predator`, `PREDATOR-SCRIPT.md`, and this file. **Those are currently untracked
> and unpushed**, so a fresh clone will not contain them, and §2b will fail.
>
> Commit and push them from the working machine *before* you need them:
>
> ```bash
> cd ~/Linuwu-Sense
> git add SETUP.md PREDATOR-SCRIPT.md predator patches/
> git commit -m "PH16-72 Ubuntu 24.04 setup: guide, CLI, driver patch"
> git push origin main            # also pushes commit 8884e8f
> ```
>
> See [Keep this reproducible](#keep-this-reproducible) for the full picture, including a
> `.gitignore` you want before pushing.

---

## 0. What you are installing, and why it takes four pieces

The PH16-72 splits its hardware control across two unrelated interfaces, which is why no
single off-the-shelf package covers it.

| Piece | Provides | Talks to |
|---|---|---|
| **Linuwu-Sense** (patched) | fans, thermal profile, battery limiter, backlight timeout, LCD overdrive | ACPI-WMI → `/sys/module/linuwu_sense/...` |
| **lamparray-kbd** | per-key RGB keyboard | USB HID → `/dev/hidraw*` |
| **`predator`** (this repo) | one CLI over both | wraps the two above |
| **DAMX** (patched) | GUI with presets, auto profile switching, tray | daemon over a Unix socket |

The keyboard is **not** a four-zone EC function on this model. It is a separate USB device
(`05af:666a`, 103 per-key lamps) that implements the standard **HID LampArray** protocol.
`linuwu_sense` cannot drive it at all, and never creates a `four_zoned_kb` sysfs group.
Anything telling you to `echo` colours into `four_zoned_kb/per_zone_mode` does not apply here.

> **Stock upstream Linuwu-Sense does not support this laptop.** It targets kernel 6.14+ and has
> no PH16-72 DMI entry. Both gaps are fixed by the patch in §2.

---

## 1. Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) git curl
```

Confirm you are on the right machine and note your kernel:

```bash
cat /sys/class/dmi/id/product_name     # expect: Predator PH16-72
uname -r                               # e.g. 6.8.0-136-generic
```

### Secure Boot

```bash
mokutil --sb-state 2>/dev/null || echo "mokutil not installed - likely disabled"
```

- **Disabled** → nothing to do. This is the simple path.
- **Enabled** → the module must be signed or it will not load. Either disable Secure Boot in
  the BIOS, or generate a MOK and place it at `~/module-signing/MOK.priv` and
  `~/module-signing/MOK.der`; the Makefile signs automatically when both exist.

---

## 2. Linuwu-Sense driver (fans, profiles, battery)

### 2a. Get the source

```bash
cd ~
git clone https://github.com/0x7375646F/Linuwu-Sense.git
cd Linuwu-Sense
```

### 2b. Apply the PH16-72 + kernel 6.8 patch

**This step is mandatory and easy to forget.** Without it the build fails on kernel < 6.14,
and even if it builds, your laptop is detected as `UNKNOWN` and nothing works.

If `src/linuwu_sense.c` already contains `Predator PH16-72`, skip to §2c:

```bash
grep -c "Predator PH16-72" src/linuwu_sense.c    # 2 = already patched, 0 = apply it
```

To apply:

```bash
patch -p1 < patches/0001-ph16-72-kernel-6.8-support.patch
```

If `patches/` is not in your clone, the repo was never pushed with it — see the warning at the
top. Copy the patch file across from a working machine, or from your backup, and apply it the
same way. It is 215 lines and touches only `src/linuwu_sense.c`.

The patch does two things:

1. **Kernel compatibility guards.** Upstream tracks 6.14+. `LINUX_VERSION_CODE` guards pick the
   right APIs on 6.8 — `asm/unaligned.h` instead of `linux/unaligned.h`,
   `BACKLIGHT_POWER_ON` → `FB_BLANK_UNBLANK`, and the older `platform_profile` registration.
2. **The PH16-72 DMI entry**, mapping it to `quirk_acer_predator_v4`.

### 2c. Build and install

```bash
make
sudo make install
```

`make install` blacklists the stock `acer_wmi`, installs the module, enables a systemd unit to
load it at boot, creates a **`linuwu_sense` group**, adds you to it, and writes
`/etc/tmpfiles.d/` rules giving that group write access to the control files. That group is why
fan and profile changes later need no `sudo`.

### 2d. Log out and back in

Required — group membership only takes effect in a new session.

```bash
id -nG | tr ' ' '\n' | grep linuwu_sense     # must print linuwu_sense
```

### 2e. Verify

```bash
ls /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/
```

Expect `predator_sense`, `hwmon`, `power`. **`four_zoned_kb` will not be there — that is
correct on this model**, see §0.

```bash
cat /sys/firmware/acpi/platform_profile_choices
# low-power quiet balanced balanced-performance performance
```

---

## 3. Per-key RGB keyboard

### 3a. Confirm the keyboard speaks LampArray

```bash
cd ~
git clone https://github.com/usamathpc/lamparray-kbd.git
cd lamparray-kbd
./lamparray-kbd list
```

Expected:

```
05af:666a  /dev/hidraw2   ACER Keyboard  (reports 0x02->0x81, ..., 8 slots/multi-update)
```

If it prints `no HID LampArray devices found`, stop — the rest of this section will not work.

### 3b. Install

```bash
./install.sh
```

Run as **your normal user, never with `sudo`** — it asks for sudo itself at the one step that
needs it. Running the whole thing as root installs the restore service into root's systemd and
your colours silently never come back after resume.

If the sudo step fails (no terminal), finish it manually:

```bash
sudo ~/.local/bin/lamparray-kbd install-udev
```

### 3c. Verify

```bash
getfacl /dev/hidraw2 | grep "^user:"      # expect user:<you>:rw-
lamparray-kbd solid 4287f5                # keyboard turns blue
```

---

## 4. The `predator` CLI

One command for everything. Copy it from this repo and put it on your PATH:

```bash
cd ~/Linuwu-Sense
ln -sf "$PWD/predator" ~/.local/bin/predator
predator status
```

```bash
predator status                     # profile, fan rpm, temps, toggles, keyboard state
predator mode perf                  # quiet | balanced | perf | eco | balperf   (needs sudo)
predator fan auto | max | 40,60
predator kb solid 4287f5
predator kb gradient blue magenta
predator kb zones red green blue yellow
predator kb keys 202020 w=red a=red s=red d=red
predator kb anim wave               # wave | cycle | breathe | ripple, Ctrl+C to stop
predator timeout off                # stop the 30s keyboard idle blank
```

Full reference: [`PREDATOR-SCRIPT.md`](PREDATOR-SCRIPT.md).

Only `predator mode` needs `sudo` — `platform_profile` is root-owned. Everything else works
through the `linuwu_sense` group and the keyboard's udev ACL.

---

## 5. DAMX GUI (optional)

A graphical front-end with presets, automatic profile switching on AC/battery, and a tray icon.
It needs a patched daemon, because stock DAMX hides its keyboard panels on this hardware.

```bash
cd ~/Linuwu-Sense
git clone https://github.com/PXDiv/Div-Acer-Manager-Max.git
```

Apply the LampArray integration — see [`Div-Acer-Manager-Max/LAMPARRAY-INTEGRATION.md`](Div-Acer-Manager-Max/LAMPARRAY-INTEGRATION.md)
for what it changes and why. Then stage and install as described in
[`DAMX-Install/INSTALL.md`](DAMX-Install/INSTALL.md):

```bash
cd ~/Linuwu-Sense/DAMX-Install
sudo ./setup.sh          # choose option 2, "without drivers"
```

> **Always choose option 2.** Option 1 rebuilds the driver, and the DAMX release bundles a
> Linuwu-Sense with **no PH16-72 entry** — installing it detects your laptop as `UNKNOWN`,
> puts the daemon in a restart loop, and removes `predator_sense` entirely. The staged tree
> in `DAMX-Install/` carries *your* driver source precisely so neither option can regress you.

Verify, then launch:

```bash
journalctl -u damx-daemon -n 40 | grep -i lamparray
#   LampArray per-key RGB: 05af:666a (103 lamps) on /dev/hidraw2
DAMX
```

Open **Keyboard Lighting** — the zone-colour and effects panels should both be present.

---

## 6. Post-install checks

```bash
predator status
```

Sanity list:

- `profile` shows one of the five choices
- `fan rpm` shows two non-zero numbers under load
- `temps` shows three plausible values
- keyboard section shows `05af:666a` and your current mode

Reboot once and re-run it. The driver loads via `/etc/modules-load.d/linuwu_sense.conf` and the
keyboard colour is restored by `lamparray-kbd-restore.service`.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `make` fails with `unaligned.h` or `BACKLIGHT_POWER_ON` errors | The §2b patch was not applied. |
| `predator_sense` directory missing | Laptop detected as `UNKNOWN` — the PH16-72 DMI entry is absent. Re-apply §2b, rebuild, reboot. |
| Fan/profile writes say "permission denied" | Not in the `linuwu_sense` group yet, or you have not logged out and back in since §2c. |
| `four_zoned_kb` missing | **Expected.** Per-key keyboard, see §0. |
| `no HID LampArray devices found` as your user | udev rule missing. `sudo ~/.local/bin/lamparray-kbd install-udev`, then reboot. |
| Keyboard colour reverts after ~30 s | `backlight_timeout` is on. `predator timeout off`. |
| Keyboard colour lost after suspend | Restore service not running, usually because `install.sh` was run with `sudo`. Check `systemctl --user status lamparray-kbd-restore.service`. |
| Keyboard dimmer than expected | Brightness is sticky. `predator kb bright 100`. |
| Module will not load, Secure Boot on | Sign it or disable Secure Boot — see §1. |
| Everything broke after a DAMX install | You chose option 1 and installed the bundled driver. Rebuild yours: `cd ~/Linuwu-Sense && make && sudo make install`, then reboot. |

### After a kernel upgrade

`linuwu_sense` is an out-of-tree module and **must be rebuilt for every new kernel**:

```bash
sudo apt install -y linux-headers-$(uname -r)
cd ~/Linuwu-Sense && make && sudo make install
```

`lamparray-kbd` is userspace and needs nothing. If the new kernel is 6.14 or later, the §2b
version guards simply compile the upstream branches — the patch stays valid.

---

## Keep this reproducible

⚠️ **The PH16-72 commit on the fork is currently local-only.** `8884e8f` ("PH16-72 fan and mode
implemented") has not been pushed to `origin`, so a fresh `git clone` today gives you upstream
code **without** PH16-72 support. Either push it:

```bash
cd ~/Linuwu-Sense && git push origin main
```

…or rely on [`patches/0001-ph16-72-kernel-6.8-support.patch`](patches/0001-ph16-72-kernel-6.8-support.patch),
which is a standalone copy of that change, verified to apply cleanly to upstream `73a25ec`.
Keep a copy off this machine.

That commit also accidentally includes build artifacts (`src/*.ko`, `*.o`, `*.cmd`). Worth a
`.gitignore` before pushing:

```
src/*.o
src/*.ko
src/*.mod*
src/.*.cmd
Module.symvers
modules.order
```

---

## Reference

| Path | What |
|---|---|
| `patches/0001-ph16-72-kernel-6.8-support.patch` | the driver patch, standalone |
| `predator` | the unified CLI |
| `PREDATOR-SCRIPT.md` | CLI reference |
| `lamparray-kbd/UBUNTU-24.04-GUIDE.md` | keyboard deep-dive, protocol, troubleshooting |
| `Div-Acer-Manager-Max/LAMPARRAY-INTEGRATION.md` | GUI integration design |
| `DAMX-Install/INSTALL.md` | staged GUI install |

Verified on: Ubuntu 24.04.5 LTS, kernel 6.8.0-136-generic, Secure Boot off, Predator PH16-72,
keyboard `05af:666a` (103 lamps).
