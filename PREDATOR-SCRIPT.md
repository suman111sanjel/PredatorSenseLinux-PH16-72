# `predator` — unified control script

One script for fans, thermal profile, and keyboard RGB on the Acer Predator PH16-72
(Ubuntu 24.04). It wraps two separate subsystems that have nothing to do with each other:

| Area | Underlying mechanism |
|---|---|
| profile, fans, timeout, battery, lcd | `linuwu_sense` kernel module → ACPI-WMI sysfs |
| keyboard colour + patterns | `lamparray-kbd` → HID LampArray over `/dev/hidraw2` |

Installed at `~/.local/bin/predator` (symlink to `./predator` in this repo), so edits here
take effect immediately.

## Permissions

Verified on this machine — most things need **no sudo**:

| Command | sudo? | Why |
|---|---|---|
| `fan`, `timeout`, `battery`, `lcd`, `bootsound` | no | `predator_sense/*` is group-writable by `linuwu_sense`, and you are a member |
| `kb` (all) | no | udev `uaccess` ACL grants `user:suman:rw-` on `/dev/hidraw2` |
| `mode` | **yes** | `/sys/firmware/acpi/platform_profile` is `root:root 0644` |

## Commands

```bash
predator status                   # profile, fan rpm, temps, toggles, keyboard state
predator mode                     # show current profile + choices
predator mode perf                # quiet | balanced | perf | eco | balperf  (sudo)
predator fan                      # show current
predator fan auto                 # firmware fan curve
predator fan max                  # both fans 100%
predator fan 40,60                # cpu 40%, gpu 60%
predator fan silent               # both at 1% - watch temps
predator timeout off              # stop the 30s keyboard idle blank
predator battery on               # limit charge to 80%
predator lcd on                   # LCD overdrive, less ghosting
predator bootsound off            # boot animation + sound
```

### Keyboard — static patterns

These are saved and re-applied automatically after reboot and resume.

```bash
predator kb solid 4287f5
predator kb gradient blue magenta
predator kb gradient red yellow green cyan      # any number of stops
predator kb zones red green blue yellow         # the classic four-zone look
predator kb rainbow                             # full spectrum across the board
predator kb keys 202020 w=red a=red s=red d=red
predator kb bright 100                          # 0-100, sticky
predator kb off
predator kb auto                                # firmware's own effects back
predator kb list | predator kb info             # device / full 103-lamp map
predator kb restore                             # re-apply saved lighting
```

### Keyboard — animated patterns

Run until Ctrl+C at 30 fps (the device's declared `MinUpdateInterval` is 33 ms).
**Not persistent** — they stop when you close the terminal, and the saved static pattern
comes back on the next `restore`.

```bash
predator kb anim wave            # hue wave travelling left-to-right
predator kb anim wave 2.5        # faster
predator kb anim wave 1 0.6      # speed 1, desaturated
predator kb anim cycle           # whole board cycles the spectrum
predator kb anim breathe cyan 4  # fade in/out, 4s period
predator kb anim ripple magenta  # pulse radiating from centre
```

Animations honour the brightness in your saved state, so `predator kb bright 60` dims them
too. To run one in the background:

```bash
predator kb anim wave &          # then: kill %1
```

## Colours and key names

Colours: `rrggbb` or `rgb` hex, or `red green blue white cyan magenta yellow orange purple
pink teal warmwhite coolwhite off black`.

Keys: `a`–`z`, `0`–`9`, `f1`–`f24`, `esc space enter tab backspace capslock menu`,
`lctrl lshift lalt lmeta` / `rctrl rshift ralt rmeta`, `up down left right home end pageup
pagedown insert delete printscreen`, `grave minus equal lbracket rbracket backslash
semicolon quote comma period slash`, numpad `kp0`–`kp9 kpenter kpplus kpminus kpasterisk
kpslash kpdot numlock`.

Six lamps report no key binding — `16`–`19`, `34`, `90` (`90` is almost certainly Fn).
Address those by number: `predator kb keys 000000 16=red 90=blue`.

## Gotchas

- **`backlight_timeout` blanks your colours.** If it is on, the EC kills the backlight after
  30 s idle no matter what you set. `predator status` shows it; `predator timeout off` fixes it.
- **Brightness is sticky.** It persists in `~/.config/lamparray-kbd/state.json` and applies to
  every later command until changed.
- **`fan silent` sets 1%, not off.** Temperatures can climb fast under load. `predator status`
  shows live rpm and temps.
- **Manual fan speeds persist** until you run `predator fan auto` or reboot.
- **Animations are not a daemon.** Nothing restarts them after suspend.

## Profile names

Accepted shorthands map onto the kernel's `platform_profile_choices`:

| You type | Actual profile |
|---|---|
| `eco`, `low`, `lowpower` | `low-power` |
| `quiet`, `silent` | `quiet` |
| `bal`, `balanced` | `balanced` |
| `bp`, `balperf` | `balanced-performance` |
| `perf`, `turbo`, `max` | `performance` |

## See also

- [`lamparray-kbd/UBUNTU-24.04-GUIDE.md`](lamparray-kbd/UBUNTU-24.04-GUIDE.md) — full setup,
  troubleshooting, protocol background
- [`README.md`](README.md) — Linuwu-Sense itself (fans, profiles, battery, sysfs reference)
