# GIGABYTE AERO X16 RGB Linux

Control **GIGABYTE AERO X16 1VH** keyboard RGB lighting on Ubuntu Linux using the dedicated **Gimate** key. No GCC required.

## Introduction

Hi everyone!

I decided to share this project because I thought it might help someone save a lot of **time, frustration, and hassle**.

Unfortunately, **GIGABYTE Control Center (GCC)** is not available on Linux, but I really wanted to be able to control the RGB lighting on my keyboard. In my case, every time I turned on my laptop, the keyboard backlight would start cycling through colors, and it was incredibly annoying that I had no way to change it.

I even installed Windows just to try to solve the problem there. Unfortunately, GCC didn't help me either. I eventually had to download third-party software from GitHub just to get proper control over the keyboard lighting.

For some reason, **RGB Fusion** on Windows doesn't even show me the available color options for the keyboard, but that's a separate issue.

After spending quite a bit of time figuring everything out, I decided to document my solution and share it here.

This is my working method for controlling the keyboard RGB lighting on **Ubuntu** on the **GIGABYTE AERO X16 1VH**.

The setup allows you to control the keyboard lighting directly from Linux using the dedicated **Gimate** key, without needing Windows or GIGABYTE software.

> **Tested and working on GIGABYTE AERO X16 1VH with Ubuntu Linux.**
> Special thanks to ChatGPT for helping me figure this out and for helping me put this documentation together

# GIGABYTE AERO X16 1VH RGB Keyboard Control on Linux

A Linux-based solution for controlling the keyboard RGB backlight on the **GIGABYTE AERO X16 1VH** and assigning the dedicated **Gimate** key to RGB controls.

The setup was developed and tested on:

* Laptop: **GIGABYTE AERO X16 1VH**
* OS: **Ubuntu Linux**
* USB Vendor ID: `0414` - GIGABYTE
* USB Product ID: `8104`
* USB ID: `0414:8104`
* HID device: `GIGABYTE USB-HID Keyboard`
* Keyboard event device used during testing: `/dev/input/event15`
* Gimate scan code: `0x70067`
* Linux key code: `KEY_KPEQUAL` / `117`

> **Important:** `/dev/input/event15` is not guaranteed to be the same on every installation. Always find the correct event device with `evtest` instead of assuming it is `event15`.

---

# Features

The dedicated Gimate key works as an RGB controller:

| Key combination  | Action                |
| ---------------- | --------------------- |
| `Gimate`         | Next RGB color        |
| `Shift + Gimate` | Next RGB mode         |
| `Ctrl + Gimate`  | RGB off               |
| Normal `=`       | Still works normally  |
| Gimate itself    | Does **not** type `=` |

### RGB modes

* Static
* Breathing
* Rainbow

### RGB colors

* Red
* Green
* Blue
* Purple
* Yellow
* Cyan
* White

The color and mode indexes are independent.

Color cycle:

`Gimate -> Red -> Green -> Blue -> Purple -> Yellow -> Cyan -> White -> Red -> ...`

Mode cycle:

`Shift + Gimate -> Static -> Breathing -> Rainbow -> Static -> ...`

RGB off:

`Ctrl + Gimate -> RGB OFF`

All of the above behavior is **tested and working on my GIGABYTE AERO X16 1VH**.

---

# Why This Is Necessary

On Ubuntu, the Gimate key is not exposed as a dedicated RGB button.

The keyboard reports it as:

```text
MSC_SCAN = 0x70067
EV_KEY   = KEY_KPEQUAL
```

Linux therefore treats the physical Gimate key as:

```text
=
```

On the tested AERO X16, pressing Gimate originally produced:

```text
type=4 code=4 value=458855
type=1 code=117 value=1
type=1 code=117 value=0
```

`458855` in decimal is:

```text
0x70067
```

So the important identifier is not just `KEY_KPEQUAL`.

The normal `=` key must not be confused with Gimate.

The reliable identifier is:

```text
MSC_SCAN = 0x70067
```

This allows the daemon to distinguish the physical Gimate key from the normal `=` key.

---

# 1. Find the GIGABYTE USB Device

Run:

```bash
lsusb
```

Look for:

```text
0414:8104
```

The device is:

```text
GIGABYTE
```

The USB device exposes several HID interfaces, so multiple `/dev/input/event*` devices may exist.

---

# 2. Install Required Packages

Install the Python HID API:

```bash
sudo apt install python3-hidapi
```

Install `evdev`:

```bash
sudo apt install python3-evdev
```

Install `evtest`:

```bash
sudo apt install evtest
```

Verify `hidapi`:

```bash
python3 -c "import hidapi; print('hidapi OK')"
```

Expected:

```text
hidapi OK
```

Verify `evdev`:

```bash
python3 -c "import evdev; print(evdev)"
```

Expected output should look similar to:

```text
<module 'evdev' from '/usr/lib/python3/dist-packages/evdev/__init__.py'>
```

---

# 3. Find the Keyboard Event Device

Run:

```bash
sudo evtest
```

Find:

```text
GIGABYTE USB-HID Keyboard
```

On the tested system, the relevant device was:

```text
/dev/input/event15
```

Test it:

```bash
sudo evtest /dev/input/event15
```

Press the Gimate key.

The important events should look like:

```text
Event: type 4 (EV_MSC), code 4 (MSC_SCAN), value 70067
Event: type 1 (EV_KEY), code 117 (KEY_KPEQUAL), value 1
Event: type 1 (EV_KEY), code 117 (KEY_KPEQUAL), value 0
```

This establishes that:

```text
Gimate scan code = 0x70067
Gimate Linux keycode = 117 / KEY_KPEQUAL
```

---

# 4. Give the RGB HID Device User Access

Create:

```bash
sudo nano /etc/udev/rules.d/99-gigabyte-aero-rgb.rules
```

Use:

```udev
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="0414", ATTRS{idProduct}=="8104", GROUP="plugdev", MODE="0660"
```

Reload the rules:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Make sure your user belongs to `plugdev`:

```bash
groups
```

If necessary:

```bash
sudo usermod -aG plugdev "$USER"
```

Log out and back in after changing group membership.

---

# 5. Give the Keyboard Input Device User Access

The Gimate daemon needs access to the keyboard event device.

Create:

```bash
sudo nano /etc/udev/rules.d/99-gigabyte-keyboard-input.rules
```

Use:

```udev
SUBSYSTEM=="input", KERNEL=="event*", ENV{ID_VENDOR_ID}=="0414", ENV{ID_MODEL_ID}=="8104", ENV{ID_INPUT_KEYBOARD}=="1", GROUP="plugdev", MODE="0660"
```

Reload:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=input
```

Check:

```bash
ls -l /dev/input/event15
```

Expected:

```text
crw-rw---- 1 root plugdev ... /dev/input/event15
```

The important part is:

```text
root plugdev
```

---

# 6. RGB Control Script

The RGB protocol is handled by:

```text
~/.local/bin/aero_rgb.py
```

Make it executable:

```bash
chmod +x ~/.local/bin/aero_rgb.py
```

The script supports three modes:

```text
static
breathing
rainbow
```

Static color:

```bash
python3 ~/.local/bin/aero_rgb.py static ff0000
```

Breathing color:

```bash
python3 ~/.local/bin/aero_rgb.py breathing ff0000
```

Rainbow:

```bash
python3 ~/.local/bin/aero_rgb.py rainbow
```

For `static` and `breathing`, colors are specified as hexadecimal RGB values.

---

# 7. Gimate Daemon

The final daemon is:

```text
~/.local/bin/aero-gimate
```

It performs two main jobs:

1. controls RGB;
2. intercepts the physical Gimate key.

The important part is that it uses `evdev` and `uinput`.

The physical keyboard is grabbed:

```python
device.grab()
```

A virtual keyboard is created:

```python
ui = UInput.from_device(
    device,
    name="AERO X16 Gimate Remapper",
)
```

All normal keyboard events are forwarded through the virtual keyboard.

The Gimate event is detected using:

```python
event.code == ecodes.KEY_KPEQUAL
```

combined with:

```python
last_scan == 0x70067
```

This distinction is important because otherwise the normal `=` key would also be intercepted.

---

# 8. uinput Setup

The remapper uses Linux `uinput`.

Check:

```bash
ls -l /dev/uinput
```

If necessary, create:

```bash
sudo nano /etc/udev/rules.d/99-uinput.rules
```

Use:

```udev
KERNEL=="uinput", GROUP="plugdev", MODE="0660"
```

Reload:

```bash
sudo udevadm control --reload-rules
```

On the tested system, `uinput` is built into the kernel, so:

```bash
lsmod | grep uinput
```

may return nothing.

That does not necessarily mean `uinput` is unavailable.

The important check is:

```bash
ls -l /dev/uinput
```

Expected:

```text
crw-rw---- 1 root plugdev ... /dev/uinput
```

---

# 9. Important Remapper Detail

The first attempt was made without grabbing the physical keyboard.

That did not work correctly because the physical keyboard continued sending the original event directly to Linux.

Even if another program detected Gimate, the original keyboard could still generate:

```text
=
```

The solution is:

```python
device.grab()
```

This prevents the original device events from being delivered directly to the desktop.

The daemon then creates a virtual keyboard using `uinput` and forwards all normal events.

Only the Gimate event is removed.

Therefore:

```text
Gimate
```

is intercepted, while:

```text
=
```

continues to work normally.

---

# 10. Final Gimate Behavior

### Gimate

Changes the RGB color.

`Gimate -> Red -> Green -> Blue -> Purple -> Yellow -> Cyan -> White -> Red -> ...`

### Shift + Gimate

Changes the RGB mode.

`Shift + Gimate -> Static -> Breathing -> Rainbow -> Static -> ...`

### Ctrl + Gimate

Turns RGB off.

Implementation:

```text
static 000000
```

The above mappings are **tested and working on my GIGABYTE AERO X16 1VH**.

---

# 11. RGB Process Handling

The RGB script can run continuously for animated modes such as:

```text
breathing
rainbow
```

If multiple copies of `aero_rgb.py` run simultaneously, they can fight over the keyboard HID device.

This can cause:

```text
rapid flashing
unexpected color changes
rainbow/breathing continuing after switching modes
```

The daemon therefore keeps a reference to the current RGB process.

Before starting a new mode, it terminates the previous process and waits for it to exit.

Only then is the new RGB mode started.

This prevents multiple RGB processes from controlling the keyboard at the same time.

---

# 12. systemd User Service

To start the controller automatically after login:

Create:

```ba
```
