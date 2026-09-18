# Per-Key RGB Keyboard Control on Ubuntu 24.04 — Acer Predator PH16-72

A complete setup and usage guide for `lamparray-kbd` on Ubuntu 24.04 LTS.

**Verified on this machine:**

| | |
|---|---|
| Laptop | Acer Predator Helios 16 **PH16-72** |
| OS | Ubuntu **24.04.5 LTS** |
| Kernel | `6.8.0-136-generic` |
| systemd | 255 |
| Session | Wayland, active |
| Keyboard | `05af:666a` "ACER Keyboard" (Jing-Mold / Sunrex), 103 per-key lamps |
| Node | `/dev/hidraw2` |

> Upstream developed and tested this on Fedora 44 / kernel 7.2. This guide covers the
> Ubuntu 24.04 path specifically, including a few distro quirks upstream does not mention.

---

## 1. Why not Linuwu-Sense?

`linuwu_sense` controls this laptop over **ACPI-WMI** — fans, thermal profiles, battery
limiter, `backlight_timeout`. Its `four_zoned_kb` directory drives a **four-zone** keyboard
through WMI calls.

The PH16-72 has a **per-key** keyboard, and it is a **separate USB device**, not a WMI
function. No WMI call reaches it. Two independent consequences:

- `four_zoned_kb` never appears in sysfs on this model. The DMI entry for `Predator PH16-72`
  maps to `quirk_acer_predator_v4`, which does not set `.four_zone_kb`, so the driver skips
  creating that attribute group at probe time. This is correct behaviour, not a bug.
- Even if the flag were forced on, the WMI writes would address zone registers this keyboard
  does not implement.

**The two tools are complementary, not competing. Keep both:**

| Want to control | Use |
|---|---|
| Fan speed, thermal/platform profile, battery limiter, `backlight_timeout` | `linuwu_sense` (sysfs) |
| Keyboard colours, per-key, brightness, effects | `lamparray-kbd` |

---

## 2. How it works

The keyboard implements **HID LampArray** — the standard USB-IF "Lighting and Illumination"
protocol (usage page `0x59`, HUT 1.5 §25), the same one Windows *Dynamic Lighting* uses.
That is why no reverse-engineering was needed: it is an open standard, and the keyboard
advertises it on HID interface 0.

`lamparray-kbd` is a single ~320-line Python 3 file that sends HID feature reports to
`/dev/hidraw2` via `HIDIOCSFEATURE`/`HIDIOCGFEATURE` ioctls.

**No kernel module. No compilation. No DKMS. No Secure Boot signing.** This matters on
Ubuntu — unlike `linuwu_sense`, there is nothing to rebuild when you take a kernel update.

---

## 3. Prerequisites

Ubuntu 24.04 already ships everything:

- Python 3.8+ — Ubuntu 24.04 has 3.12.3 at `/usr/bin/python3`
- systemd + udev — version 255
- An **active graphical login session** (see §8.2 for why)
- `sudo` access, needed exactly once

Nothing to `apt install`.

---

## 4. Pre-flight check — confirm your keyboard qualifies

Do this **before** installing. It is read-only and needs no root.

```bash
cd /home/suman/SharedPath/Software/Linuwu-Sense/lamparray-kbd
./lamparray-kbd list
```

Expected on the PH16-72:

```
05af:666a  /dev/hidraw2   ACER Keyboard  (reports 0x02->0x81, 0x20->0x82, 0x22->0x83, 0x50->0x84, 0x60->0x85, 0x70->0x86, 8 slots/multi-update)
```

All six LampArray reports must be listed — attributes, attribute request/response,
multi-update, range-update, and control. If you instead see `no HID LampArray devices found`,
your keyboard does not speak this protocol and this tool cannot help it.

---

## 5. Install

```bash
cd /home/suman/SharedPath/Software/Linuwu-Sense/lamparray-kbd
./install.sh
```

This does four things, all reversible:

1. Copies the script to `~/.local/bin/lamparray-kbd`
2. Installs `~/.config/systemd/user/lamparray-kbd-restore.service`
3. Enables that service for your user
4. Calls `sudo lamparray-kbd install-udev` to write the udev rule

**Run `install.sh` as your normal user — never with `sudo`.** It asks for sudo itself at
step 4 only. Running the whole thing as root installs the service into root's systemd, and
auto-restore will silently never fire.

### If the sudo step fails

In a non-interactive shell the sudo prompt cannot be answered and step 4 fails while steps
1–3 succeed. Finish it manually:

```bash
sudo ~/.local/bin/lamparray-kbd install-udev
```

### What the udev rule contains

Inspect it first if you like — `./lamparray-kbd udev-rule` prints it without writing anything:

```
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="05af", ATTRS{idProduct}=="666a", TAG+="uaccess", TAG+="systemd", ENV{SYSTEMD_USER_WANTS}="lamparray-kbd-restore.service"
```

Scoped to this one keyboard by vendor and product ID. `uaccess` grants access only to the
user at the active seat, via a POSIX ACL — it does **not** make the device world-writable.
`SYSTEMD_USER_WANTS` re-triggers the restore service when the device re-enumerates.

Written to `/etc/udev/rules.d/70-lamparray-kbd.rules`. It is the only file placed outside
your home directory.

---

## 6. Verify

```bash
lamparray-kbd list          # should now work without sudo
getfacl /dev/hidraw2        # should show: user:suman:rw-
lamparray-kbd solid 4287f5  # keyboard turns blue immediately
```

If `lamparray-kbd: command not found`, open a new terminal — `~/.local/bin` is on Ubuntu
24.04's default PATH, but only for shells started after install. Or call it by full path:
`~/.local/bin/lamparray-kbd`.

---

## 7. Usage

### Commands

```bash
lamparray-kbd list                          # detected LampArray devices
lamparray-kbd info                          # all 103 lamps: key name, mm position, levels
lamparray-kbd solid 4287f5                  # one colour everywhere
lamparray-kbd gradient blue magenta         # left-to-right, uses real lamp x positions
lamparray-kbd gradient red yellow green cyan  # any number of stops
lamparray-kbd zones red green blue yellow   # N equal left-to-right zones
lamparray-kbd keys 202020 w=red a=red s=red d=red   # per-key over a base colour
lamparray-kbd off                           # all lamps off
lamparray-kbd auto                          # hand control back to firmware effects
lamparray-kbd restore                       # re-apply saved state (what systemd calls)
```

### Flags

```bash
lamparray-kbd -b 60 solid cyan       # brightness 0-100, remembered for later commands
lamparray-kbd -b 100 solid cyan      # ...so set it back explicitly when you want full
lamparray-kbd -d 05af:666a solid red # pick a device if several are present
```

Brightness is **sticky**. It is stored in your state file and silently applies to every
later command until you change it. A later "why is my keyboard dim" is usually this.

### Colours

Hex `rrggbb` or `rgb`, with or without a leading `#`, or a name:

```
red green blue white cyan magenta yellow orange purple pink teal
warmwhite coolwhite off black
```

### Key names for `keys`

Taken from the HID keycode each lamp reports, so they come from the hardware itself:

- Letters `a`–`z`, digits `0`–`9`, function keys `f1`–`f24`
- `esc space enter tab backspace capslock menu`
- `lctrl lshift lalt lmeta` / `rctrl rshift ralt rmeta`
- `up down left right home end pageup pagedown insert delete printscreen`
- `grave minus equal lbracket rbracket backslash semicolon quote comma period slash`
- Numpad: `kp0`–`kp9 kpenter kpplus kpminus kpasterisk kpslash kpdot numlock`
- Raw lamp numbers from `info` — needed for the few unnamed lamps

**Six lamps on this keyboard report no key binding** and appear by number instead of name:
`lamp16`–`lamp19` (top row, right of Delete, x = 282–331 mm), `lamp34` (numpad column,
second row) and `lamp90` (bottom row between Left Ctrl and the Meta/Super key — almost
certainly the Fn key, which never sends a normal HID keycode). Address them by number:

```bash
lamparray-kbd keys 000000 16=red 17=red 18=red 19=red
```

`lamparray-kbd info` is the authoritative map for your unit.

### Recipes

```bash
# WASD gaming highlight, everything else dim grey
lamparray-kbd keys 202020 w=red a=red s=red d=red

# WASD + arrows + space, dark base
lamparray-kbd keys 101010 w=cyan a=cyan s=cyan d=cyan up=cyan down=cyan left=cyan right=cyan space=white

# Function row picked out
lamparray-kbd keys 303030 f1=orange f2=orange f3=orange f4=orange

# The four-zone look, done per-key
lamparray-kbd zones red green blue yellow

# Subtle warm desk light
lamparray-kbd -b 35 solid warmwhite

# Escape key as a modal indicator
lamparray-kbd keys 4287f5 esc=red
```

---

## 8. Ubuntu 24.04 specifics

### 8.1 Conda's Python shadows the system Python

If you use miniconda/anaconda, `python3` resolves to conda's interpreter:

```
$ which python3
/home/suman/miniconda3/bin/python3     # 3.13.11, not the system 3.12.3
```

The script's shebang is `#!/usr/bin/env python3`, so it runs under **whichever Python is
first on PATH**. This is harmless — the tool imports only the standard library
(`argparse fcntl glob json os re struct subprocess sys time`) and works on any Python 3.8+.

But if you ever hit a Python error, rule the interpreter out first:

```bash
/usr/bin/python3 ~/.local/bin/lamparray-kbd list
```

### 8.2 `uaccess` requires an active local session

The udev rule grants access through systemd-logind's `uaccess` mechanism, which applies the
ACL to the user at the **active local seat**. Practical consequences:

- Works normally on Ubuntu Desktop, Wayland or X11
- **Does not work over plain SSH** while no local session is active — you will get
  `no permission on /dev/hidraw2`. Use `sudo` for the occasional remote invocation.
- Briefly unavailable during fast-user-switching and on the lock screen at suspend entry

The tool already retries for five seconds on open, because udev applies the ACL slightly
after the device node appears.

### 8.3 Kernel updates cost you nothing

`linuwu_sense` is an out-of-tree kernel module and must be rebuilt for every new kernel.
`lamparray-kbd` is userspace talking to `hid-generic`. Ubuntu kernel upgrades do not affect
it, and Secure Boot is irrelevant to it.

### 8.4 `backlight_timeout` interacts with this

`linuwu_sense` exposes a firmware idle-off for the keyboard backlight:

```bash
cat /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/backlight_timeout
```

If it reads `1`, the EC blanks the keyboard after 30 seconds idle regardless of what colour
you set. If your lighting "randomly turns off", check this before suspecting `lamparray-kbd`:

```bash
echo 0 | sudo tee /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/predator_sense/backlight_timeout
```

### 8.5 The lid logo is a different device

The Predator logo on the lid is `0d62:ba51` (Darfon, "Cover Logo") on `/dev/hidraw6`. It uses
a **proprietary** protocol, not LampArray, and this tool does not control it.

---

## 9. Persistence

The firmware does **not** keep host-set colours. Power-cycling the keyboard — which happens
on every suspend/resume — reverts it. The restore service exists to paper over this.

- Your last setting per device lives in `~/.config/lamparray-kbd/state.json`
- `lamparray-kbd-restore.service` runs `lamparray-kbd restore` at login
- The udev rule re-triggers that service whenever the keyboard re-enumerates

Check it:

```bash
systemctl --user status lamparray-kbd-restore.service
journalctl --user -u lamparray-kbd-restore -n 50
cat ~/.config/lamparray-kbd/state.json
```

To make a setting permanent, just set it — it is saved automatically. To stop auto-restore
without uninstalling:

```bash
systemctl --user disable --now lamparray-kbd-restore.service
```

---

## 10. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `no HID LampArray devices found` (as user) | udev rule missing or not applied. Run `sudo ~/.local/bin/lamparray-kbd install-udev`, then reboot. Confirm with `getfacl /dev/hidraw2`. |
| `no HID LampArray devices found` (as root too) | Keyboard genuinely does not expose LampArray. Verify with `lsusb \| grep 05af`. |
| `no permission on /dev/hidrawN` | No active local session — see §8.2. Over SSH, use `sudo`. |
| `command not found` | Open a new terminal, or use `~/.local/bin/lamparray-kbd`. |
| Colour set, then reverts after a few seconds | `backlight_timeout` is on — see §8.4. |
| Colour lost after suspend | Restore service not running. Check `systemctl --user status lamparray-kbd-restore.service`. Most common cause: `install.sh` was run with `sudo`. |
| `list` works but colours do nothing | Firmware may want a vendor mode-switch first. Try `lamparray-kbd auto`, then your command. If still dead, open an upstream issue with `info` output. |
| Keyboard dimmer than expected | Sticky brightness — run `lamparray-kbd -b 100 solid white`. |
| Keys named `lampNN` | Those lamps report no HID binding. Address by number — see §7. |
| Want the Fn-key effects back | `lamparray-kbd auto` returns control to firmware. |

---

## 11. Updating and removing

```bash
# Update
cd /home/suman/SharedPath/Software/Linuwu-Sense/lamparray-kbd
git pull
./install.sh          # safe to re-run

# Remove (keeps ~/.config/lamparray-kbd)
./uninstall.sh

# Remove config too
rm -rf ~/.config/lamparray-kbd
```

`uninstall.sh` disables and deletes the service, removes the binary, deletes the udev rule,
and reloads udev.

---

## 12. Known limitations

- **No animated effects.** LampArray is a static per-lamp protocol; breathing and wave need a
  daemon streaming frames at up to 30 fps (`MinUpdateInterval` is 33 ms). Use
  `lamparray-kbd auto` for the firmware's own animations, or script your own loop.
- **Lid logo not covered** — separate proprietary device, §8.5.
- **Six lamps unnamed** — addressable by number, §7.
- **No GUI.** OpenRGB gained a generic HID LampArray controller on master in August 2026;
  once that reaches a tagged release it should detect this keyboard and provide one.

---

## 13. Reference

- Upstream: <https://github.com/usamathpc/lamparray-kbd> (MIT)
- Protocol notes: [`docs/PROTOCOL.md`](docs/PROTOCOL.md)
- This exact keyboard: [`docs/devices/acer-predator-helios-16-ph16-72.md`](docs/devices/acer-predator-helios-16-ph16-72.md)
  — full report descriptor and all 103 lamp positions
- Microsoft Dynamic Lighting device guidelines:
  <https://learn.microsoft.com/en-us/windows-hardware/design/component-guidelines/dynamic-lighting-devices>
