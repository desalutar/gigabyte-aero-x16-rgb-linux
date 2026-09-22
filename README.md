# GIGABYTE AERO X16 RGB Linux

Control **GIGABYTE AERO X16 1VH** keyboard RGB lighting on Ubuntu Linux using the dedicated **Gimate** key. No GIGABYTE Control Center required.

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
>
> **Special thanks to ChatGPT for helping me figure this out and for helping me put this documentation together.**

# GIGABYTE AERO X16 1VH RGB Keyboard Control on Linux

A Linux-based solution for controlling the keyboard RGB backlight on the **GIGABYTE AERO X16 1VH** and assigning the dedicated **Gimate** key to RGB controls.

The setup was developed and tested on:

* Laptop: **GIGABYTE AERO X16 1VH**
* OS: **Ubuntu Linux**
* USB Vendor ID: `0414` - GIGABYTE
* USB Product ID: `8104`
* USB ID: `0414:8104`
* HID device: `GIGABYTE USB-HID Keyboard`
* HID Usage Page: `0x59` - LampArray
* Keyboard event device used during testing: `/dev/input/event15`
* Raw HID device used during testing: `/dev/hidraw10`
* Gimate scan code: `0x70067`
* Linux key code: `KEY_KPEQUAL` / `117`
* Input interface: `4`

> **Important:** `/dev/input/event15` and `/dev/hidraw10` are installation-specific and may be different on another system. The stable identifiers are the USB VID/PID and the Gimate scan code.

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

## RGB modes

* Static
* Breathing
* Rainbow

## RGB colors

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

Install the required Python packages and `evtest`:

```bash
sudo apt update
sudo apt install python3-evdev python3-hidapi evtest
```

Verify `evdev`:

```bash
python3 -c "import evdev; print('evdev OK')"
```

Expected:

```text
evdev OK
```

Verify `hidapi`:

```bash
python3 -c "import hidapi; print('hidapi OK')"
```

Expected:

```text
hidapi OK
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

> **Important:** Do not assume that `/dev/input/event15` will be the same on another installation. Use `evtest` to find the correct device.

---

# 4. Give the RGB HID Device User Access

The RGB script needs access to the GIGABYTE LampArray HID device.

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

Reload the rules:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=input
```

Check the device:

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

> If your keyboard uses a different event number, replace `event15` with the device found using `evtest`.

---

# 6. Install the RGB Control Script

The repository contains:

```text
scripts/aero_rgb.py
```

Create the local binary directory:

```bash
mkdir -p ~/.local/bin
```

Copy the script:

```bash
cp scripts/aero_rgb.py ~/.local/bin/aero_rgb.py
```

Make it executable:

```bash
chmod +x ~/.local/bin/aero_rgb.py
```

The RGB script supports three modes:

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

Press `Ctrl+C` to stop an animated mode.

---

# 7. Install the Gimate Daemon

The repository contains:

```text
scripts/aero-gimate
```

Copy it to the local binary directory:

```bash
cp scripts/aero-gimate ~/.local/bin/aero-gimate
```

Make it executable:

```bash
chmod +x ~/.local/bin/aero-gimate
```

The daemon automatically finds `aero_rgb.py` in the same directory.

The final installation should therefore contain:

```text
~/.local/bin/
├── aero-gimate
└── aero_rgb.py
```

The daemon performs two main jobs:

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
last_scan == GIMATE_SCAN
```

where:

```python
GIMATE_SCAN = 0x70067
```

This distinction is important because otherwise the normal `=` key would also be intercepted.

---

# 8. uinput Setup

The remapper uses Linux `uinput`.

Check:

```bash
ls -l /dev/uinput
```

Create the udev rule:

```bash
sudo nano /etc/udev/rules.d/99-uinput.rules
```

Use:

```udev
KERNEL=="uinput", GROUP="plugdev", MODE="0660"
```

Reload the rules:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Check again:

```bash
ls -l /dev/uinput
```

Expected:

```text
crw-rw---- 1 root plugdev ... /dev/uinput
```

On the tested system, `uinput` is built into the kernel, so:

```bash
lsmod | grep uinput
```

may return nothing.

That does not necessarily mean that `uinput` is unavailable.

The important check is:

```bash
ls -l /dev/uinput
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

# 10. Test the Gimate Daemon Manually

Before enabling systemd, test the daemon manually.

Run:

```bash
~/.local/bin/aero-gimate
```

You should see something similar to:

```text
Opening: /dev/input/event15
Name: GIGABYTE USB-HID Keyboard
Keyboard grabbed.
Gimate RGB daemon started
```

Now test:

```text
Gimate -> next color
Shift + Gimate -> next mode
Ctrl + Gimate -> RGB OFF
```

Also verify that the normal `=` key still types:

```text
=
```

Press `Ctrl+C` to stop the daemon.

---

# 11. Final Gimate Behavior

## Gimate

Changes the RGB color.

`Gimate -> Red -> Green -> Blue -> Purple -> Yellow -> Cyan -> White -> Red -> ...`

## Shift + Gimate

Changes the RGB mode.

`Shift + Gimate -> Static -> Breathing -> Rainbow -> Static -> ...`

## Ctrl + Gimate

Turns RGB off.

Implementation:

```text
static 000000
```

The above mappings are **tested and working on my GIGABYTE AERO X16 1VH**.

---

# 12. RGB Process Handling

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

# 13. systemd User Service

To start the controller automatically after login, copy the service file from the repository:

```bash
mkdir -p ~/.config/systemd/user
cp systemd/aero-gimate.service ~/.config/systemd/user/aero-gimate.service
```

The service file is:

```ini
[Unit]
Description=GIGABYTE AERO Gimate RGB Controller
After=graphical-session.target

[Service]
Type=simple
ExecStart=%h/.local/bin/aero-gimate
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
```

Reload systemd:

```bash
systemctl --user daemon-reload
```

Enable the service:

```bash
systemctl --user enable aero-gimate.service
```

Start it:

```bash
systemctl --user start aero-gimate.service
```

Check the status:

```bash
systemctl --user status aero-gimate.service --no-pager
```

You should see:

```text
Active: active (running)
```

---

# 14. Useful systemd Commands

Start:

```bash
systemctl --user start aero-gimate.service
```

Stop:

```bash
systemctl --user stop aero-gimate.service
```

Restart:

```bash
systemctl --user restart aero-gimate.service
```

Check status:

```bash
systemctl --user status aero-gimate.service
```

View live logs:

```bash
journalctl --user -u aero-gimate.service -f
```

Disable automatic startup:

```bash
systemctl --user disable aero-gimate.service
```

---

# 15. Troubleshooting

## RGB device not found

If you see:

```text
LampArray GIGABYTE 0414:8104 не найден.
```

Check:

```bash
lsusb
```

Make sure the device:

```text
0414:8104
```

is present.

Then check:

```bash
ls -l /dev/hidraw*
```

and verify the udev rule is installed.

Reload:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## Permission denied on `/dev/hidraw*`

Check:

```bash
ls -l /dev/hidraw*
```

The relevant device should belong to:

```text
root plugdev
```

Make sure your user is in `plugdev`:

```bash
groups
```

If necessary:

```bash
sudo usermod -aG plugdev "$USER"
```

Then log out and back in.

---

## Permission denied on `/dev/input/event*`

Check:

```bash
ls -l /dev/input/event15
```

The relevant device should belong to:

```text
root plugdev
```

If the event number is different, find the correct device using:

```bash
sudo evtest
```

---

## `/dev/uinput` permission denied

Check:

```bash
ls -l /dev/uinput
```

Expected:

```text
crw-rw---- 1 root plugdev ... /dev/uinput
```

Reload the udev rules:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## Gimate still types `=`

Make sure the daemon is running:

```bash
systemctl --user status aero-gimate.service
```

Check the event:

```bash
sudo evtest /dev/input/event15
```

The Gimate key must report:

```text
MSC_SCAN = 0x70067
KEY_KPEQUAL
```

The daemon must also be using the correct event device.

---

## Normal `=` key stops working

Make sure you are using the final `aero-gimate` implementation.

The daemon must:

1. grab the physical key
