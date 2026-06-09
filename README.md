# ThinkBook Media Keys Fix

Fix Lenovo laptop media-key events on Linux with DMI-scoped `systemd-hwdb` rules.

This project documents tested on Lenovo ThinkBook media-key quirks and installs
matching `hwdb` rules from `src/`.

> [!WARNING]
>
> Do not use a global match such as:

```ini
evdev:atkbd:dmi:*
```

## Test and Diagnose

Use this section to verify whether a Lenovo model needs a new `hwdb` rule.

### Record DMI Identity

```bash
cat /sys/class/dmi/id/sys_vendor
cat /sys/class/dmi/id/product_name
cat /sys/class/dmi/id/product_version
cat /sys/class/dmi/id/product_sku
cat /sys/class/dmi/id/board_name
cat /sys/class/dmi/id/bios_version
```

Use these values to create a narrow DMI match.

Example match for product name `21MW`:

```ini
evdev:atkbd:dmi:*svnLENOVO*:pn21MW:*
```

### List Input Devices

```bash
for e in /dev/input/event*; do
  printf '%s: ' "$e"
  cat "/sys/class/input/${e##*/}/device/name" 2>/dev/null
done
```

Common devices:

```text
AT Translated Set 2 keyboard
Ideapad extra buttons
```

### Watch Logical Key Events

```bash
sudo libinput debug-events | grep -i -E 'mute|volume|mic|brightness|touchpad'
```

A healthy key should emit one logical press/release pair per physical press.

Problem patterns:

```text
KEY_VOLUMEUP pressed
KEY_VOLUMEUP pressed
KEY_VOLUMEUP pressed
```

Missing or broken release.

```text
event3 KEY_MICMUTE pressed
event3 KEY_MICMUTE released
event6 KEY_MICMUTE pressed
event6 KEY_MICMUTE released
```

Duplicate source.

```text
event6 KEY_MICMUTE pressed
event6 KEY_MICMUTE released
event6 KEY_MICMUTE pressed
event6 KEY_MICMUTE released
```

Duplicate scancode from one source.

### Inspect Raw Scancodes

Check the AT keyboard path:

```bash
sudo timeout 20s evtest /dev/input/eventX
```

Check the Ideapad path:

```bash
sudo timeout 20s evtest /dev/input/eventY
```

Replace `eventX` and `eventY` with the correct event nodes from the device list.

Use `!` only for keys with broken release behavior:

```ini
KEYBOARD_KEY_ae=!volumedown
```

Use `reserved` for duplicate scancodes:

```ini
KEYBOARD_KEY_8c=reserved
```

Do not map unknown scancodes until confirming that no other input device already
emits the correct key.

---

## Tested Fixes

<details>
<summary><h3>ThinkBook 16 G7 ARP</h3></summary>

Tested and working with the `hwdb` rule in `src/`.

### System Identity

```text
sys_vendor: LENOVO
product_name: 21MW
product_version: ThinkBook 16 G7 ARP
product_sku: LENOVO_MT_21MW_BU_idea_FM_ThinkBook 16 G7 ARP
```

### Root Cause

This model reports affected media keys through two input devices:

```text
AT Translated Set 2 keyboard
Ideapad extra buttons
```

Volume Up, Volume Down are reported through the legacy AT
keyboard path:

```text
AT Translated Set 2 keyboard
```

Affected scancodes:

| Scancode | Key         |
| -------- | ----------- |
| `0xae`   | Volume Down |
| `0xb0`   | Volume Up   |

These keys can miss a usable release event or leave broken key state. The rule
uses the `!` modifier to emit an immediate synthetic release after each press.

Mic Mute is emitted by both input paths.

The AT keyboard path emits a duplicate Mic Mute scancode:

```text
AT Translated Set 2 keyboard
0x8c
```

The Ideapad path emits two Mic Mute scancodes from one physical key press:

```text
Ideapad extra buttons
0x08  -> KEY_MICMUTE
0x13e -> KEY_MICMUTE
```

The rule disables the duplicate AT-keyboard scancode and the duplicate Ideapad scancode.

Final Mic Mute source:

```text
Ideapad extra buttons
0x08 -> KEY_MICMUTE
```

### Install

Use either the package or manual installation.

<details>
<summary><h4>Arch Linux package</h4></summary>

Build and install from `archlinux/`:

```bash
cd archlinux
makepkg -si
```

The package installs the tested `hwdb` rule from `src/`.

</details>

<details>
<summary><h4>Manual install</h4></summary>

Install the `hwdb` rule:

```bash
sudo install -Dm644 src/61-lenovo-thinkbook-media.hwdb \
  /etc/udev/hwdb.d/61-lenovo-thinkbook-media.hwdb
```

Apply the update:

```bash
sudo systemd-hwdb update
sudo udevadm trigger -s input
```

If active input devices keep old properties, reboot:

```bash
systemctl reboot
```

</details>

### Verify

List input devices:

```bash
for e in /dev/input/event*; do
  printf '%s: ' "$e"
  cat "/sys/class/input/${e##*/}/device/name" 2>/dev/null
done
```

Expected relevant devices:

```text
AT Translated Set 2 keyboard
Ideapad extra buttons
```

Check the AT keyboard properties:

```bash
udevadm info --query=property --name=/dev/input/eventX | grep '^KEYBOARD_KEY_'
```

Expected properties:

```text
KEYBOARD_KEY_ae=!volumedown
KEYBOARD_KEY_b0=!volumeup
KEYBOARD_KEY_8c=reserved
```

Check the Ideapad extra-buttons properties:

```bash
udevadm info --query=property --name=/dev/input/eventY | grep '^KEYBOARD_KEY_'
```

Expected property:

```text
KEYBOARD_KEY_13e=reserved
```

Test final media-key output:

```bash
sudo libinput debug-events | grep -i -E 'mute|volume|mic'
```

Expected behavior:

| Physical key | Expected source                | Expected event                          |
| ------------ | ------------------------------ | --------------------------------------- |
| Volume Down  | `AT Translated Set 2 keyboard` | one `KEY_VOLUMEDOWN` press/release pair |
| Volume Up    | `AT Translated Set 2 keyboard` | one `KEY_VOLUMEUP` press/release pair   |
| Mic Mute     | `Ideapad extra buttons`        | one `KEY_MICMUTE` press/release pair    |

Mic Mute should not produce a second `KEY_MICMUTE` event within a few milliseconds.

</details>
