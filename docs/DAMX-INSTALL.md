# DAMX install — staged and ready

Everything is prepared. One command left, and it needs a password so it has to be yours.

```bash
cd /home/suman/SharedPath/Software/Linuwu-Sense/DAMX-Install
sudo ./setup.sh
```

**Choose option `2` — "Install DAMX Suite (without drivers)".**

Your `linuwu_sense` module is already built, loaded and working. Option 2 installs the
daemon, GUI and LampArray helper without touching it. Option 1 is also safe (this tree
carries *your* driver source, not the release's), but it rebuilds a module that already
works, so there is no reason to.

When it asks about the Nitro/PredatorSense key, answer as you like — it times out to "no"
after 10 seconds.

## What is in this tree, and why

| Component | Source | Why |
|---|---|---|
| `DAMX-GUI/` | official release 1.0.2-h1 | SHA256 verified against upstream's published checksum; self-contained, needs no .NET |
| `DAMX-Daemon/` | **patched source**, script mode | carries the LampArray backend so your keyboard works |
| `Linuwu-Sense/` | **your local fork** | see the warning below |
| `lamparray-kbd/` | your clone | the installer picks it up locally, no network needed |
| `setup.sh` | **patched** `local-setup.sh` | adds `install_lamparray()` and sibling-module copying |

### Why not the release's driver

The bundled Linuwu-Sense (drivers version 25.701) has **no `Predator PH16-72` DMI entry** —
it jumps from PH16-71 straight to PH18-71. Installing it would leave your laptop detected as
`UNKNOWN`, which puts the daemon into its restart loop and removes `predator_sense`
altogether: no fan control, no thermal profile, no `backlight_timeout`.

This tree ships your working backported source instead, so neither menu option can regress you.

### Why the daemon is a script, not a binary

Upstream freezes the daemon with PyInstaller. PyInstaller is not installed here, and the
daemon is **pure standard library** (verified), so it runs directly on the system Python 3.12.
Benefits: no build step, no 17 MB binary, and the patch stays readable and editable at
`/opt/damx/daemon/`. The systemd unit runs it via `#!/usr/bin/env python3`, which under
systemd's PATH resolves to `/usr/bin/python3` — not your conda Python.

## Verify after installing

```bash
systemctl status damx-daemon.service
journalctl -u damx-daemon -n 40 | grep -i lamparray
```

Expect a line like:

```
LampArray per-key RGB: 05af:666a (103 lamps) on /dev/hidraw2
```

Then launch the GUI (`DAMX`, or from your app launcher) and open **Keyboard Lighting**. Both
the zone-colour and lighting-effects panels should be present and should drive the keyboard.

## If something goes wrong

```bash
sudo ./setup.sh        # option 3 = uninstall, removes service, /opt/damx, udev rule
```

Your driver, your `predator` CLI and `~/.local/bin/lamparray-kbd` are untouched by the
uninstall, so you always have a working fallback.

## Known unverified

The GUI was never compiled or clicked through here — no .NET SDK on this machine. The daemon
side is tested (feature detection, socket dispatch, all 8 modes, validation), and the GUI is
the official unmodified binary, but the end-to-end click-through is the one thing installing
will actually prove.
