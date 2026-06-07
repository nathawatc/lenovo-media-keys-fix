# ThinkBook Media Keys Fix

Fix broken Lenovo ThinkBook media keys on Linux.

Affected systems may have two input issues:

1. **Runaway volume keys** — Volume Up/Down send repeat events but no release event.
2. **Mute key bounce** — The mute key emit multiple events from one press.

## Install

Choose one installation method.

<details>
    <summary>
        <h3>Arch Linux package</h3>
    </summary>

Build and install from the `archlinux/`:

```bash
cd archlinux
makepkg -si
```

The package installs the hwdb rule for the runaway volume-key fix.

</details>

<details>
    <summary>
        <h3>Manual hwdb install</h3>
    </summary>

Install the hwdb rule manually:

```bash
sudo install -Dm644 src/61-lenovo-media.hwdb \
  /etc/udev/hwdb.d/61-lenovo-media.hwdb
```

Apply the hwdb update:

```bash
sudo systemd-hwdb update
sudo udevadm trigger
```

</details>

## How It Works

### Runaway Volume-Key Fix

Volume Up/Down are fixed with a `systemd-hwdb` rule.

The affected keys do not send release events, so the hwdb rule maps them with
the `!` modifier. This makes the input stack emit an immediate synthetic release
event after each key press.

```ini
evdev:atkbd:dmi:*
 KEYBOARD_KEY_ae=!volumedown
 KEYBOARD_KEY_b0=!volumeup
```

> [!IMPORTANT]
>
> Keep exactly one leading space before each `KEYBOARD_KEY` line.
> The `!` modifier forces an instant release event.

### Mic Mute-Key Debounce

Mic mute is handled separately in user space.

The mute key can emit multiple press/release events from one physical press.
`systemd-hwdb` cannot debounce keyboard events by time, so this needs a
session-level binding in the window manager or desktop environment.

For Hyprland 0.55 Lua config, bind `XF86AudioMicMute` to:

```lua
hl.bind(
  "XF86AudioMicMute",
  hl.dsp.exec_cmd("flock -n /tmp/micmute-$UID.lock -c 'wpctl set-mute @DEFAULT_AUDIO_SOURCE@ toggle && sleep 0.2; rm -f /tmp/micmute-$UID.lock'"),
  { locked = true, repeating = true }
)
```

The first event toggles mic mute. Repeated bounce events within 200 ms are
ignored while the lock is held.
