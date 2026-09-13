# Xiaomi Pad Pen Fix for macOS

A lightweight macOS workaround for the **high pen-click pressure threshold** when using a Xiaomi Pad 9 Promax as a pen/tablet input device over USB.

It fixes two issues observed with Xiaomi Pad 9 Promax pen input on macOS:

- ✍️ **Light pen touches don't register as clicks / strokes**
- 🖱️ **Cursor jumps to a screen corner when the pen leaves the sensing range**

The workaround preserves the **native pen position, pressure, and tilt data** and only fixes the problematic click-state handling.

## The Problem

When a Xiaomi Pad 9 Promax is connected to a Mac and used as a pen input device, macOS can already receive the pen's HID data correctly, including:

- X / Y position
- Tip Pressure
- Tip Switch
- In Range
- X / Y Tilt

However, there appears to be a compatibility issue in how macOS converts the digitizer input into pointer events.

### 1. Excessive pressure required to start a stroke

The pen may already report:

```text
Tip = 1
Pressure > 0
```

while macOS still does **not** generate a mouse-down event.

As a result, lightly touching the screen may:

- fail to click;
- fail to start a stroke;
- cause the beginning of a stroke to be missing.

Increasing the pressure eventually triggers the native mouse-down event.

In practice, this makes the pen feel as if it has a very large activation force.

### 2. Cursor jumps when the pen leaves the sensing range

When the pen leaves the tablet's sensing range, the device may briefly report a special coordinate such as:

```text
X = 0
Y = 32767
Pressure = 0
```

If this coordinate is propagated to the macOS pointer system, the cursor suddenly moves toward a screen corner.

This is particularly annoying when macOS **Hot Corners** are enabled.

---

## What This Fix Does

The key idea is simple:

> **Use the HID Tip Switch to determine whether the pen is touching the screen, instead of relying on macOS's unusually high native click threshold.**

The script directly monitors the Xiaomi Pad through macOS `IOHIDManager`.

When the pen reports:

```text
Tip = 1
```

the script ensures that the corresponding mouse-down state exists.

The stroke then remains active until the pen actually reports:

```text
Tip = 0
```

This avoids using pressure itself as the click threshold.

Conceptually:

```text
                Xiaomi Pad HID
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      X / Y         Pressure         Tip
        │              │              │
        │              │              ▼
        │              │       Click-state fix
        │              │              │
        └──────────────┼──────────────┘
                       │
                       ▼
                  macOS Input
```

The native pressure data is **not remapped or modified**.

This is important because the pressure values themselves appear to be correct. The problem is the transition from pen contact to mouse-down.

---

## HID Data

The Xiaomi Pad exposes the pen as a standard HID Digitizer device.

The Tip Pressure element observed on the tested device is:

```text
Usage Page:   0x0D    # Digitizer
Usage:        0x30    # Tip Pressure
Logical Min:  0
Logical Max:  8191
Report ID:    3
Report Size:  16 bit
```

Actual HID reports show that low pressure values are transmitted correctly:

```text
Tip=1  Pressure=138
Tip=1  Pressure=517
Tip=1  Pressure=1040
Tip=1  Pressure=3000
Tip=1  Pressure=6000
...
```

This confirms that the issue is **not caused by the tablet failing to detect light pressure**.

The tablet already knows that the pen has touched the screen and sends both the Tip Switch and pressure information to the Mac.

---

## Why Not Remap Pressure?

An earlier approach was to artificially boost low pressure values, for example:

```text
0 ... 500  →  higher synthetic pressure
```

so that they would cross the macOS native click threshold.

This works in principle, but has several disadvantages:

- modifies the real pressure curve;
- reduces low-pressure drawing precision;
- may interfere with applications that use native pressure;
- can introduce discontinuities when switching between synthetic and native events.

A particularly noticeable failure mode was:

```text
High pressure
    ↓
Native MouseDown
    ↓
Pressure decreases
    ↓
Native MouseUp
    ↓
Synthetic MouseDown
```

That tiny `MouseUp` produces a visible break in the stroke.

The current implementation therefore treats one physical pen contact as one continuous stroke:

```text
Tip Down
   │
   ├── low pressure
   ├── medium pressure
   ├── high pressure
   ├── low pressure again
   │
Tip Up
```

No pressure-dependent state transition occurs in the middle.

---

## Out-of-Range Cursor Fix

The script also detects invalid/out-of-range HID coordinates generated when the pen leaves the sensing area.

Instead of allowing a value such as:

```text
X = 0
Y = 32767
```

to move the macOS cursor, the invalid coordinate is ignored.

The cursor therefore remains at the **last valid pen position** after the pen leaves the tablet.

---

## Requirements

- macOS
- Swift
- Xiaomi Pad with USB pen/tablet input support
- Accessibility/Input Monitoring permissions may be required by macOS

The current implementation has been tested with:

```text
Product:   Xiaomi Pad 9 Pro Max
VendorID:  6353
ProductID: 11525
```

Other Xiaomi Pad models may work if they expose a compatible HID Digitizer interface, but they have not been tested yet.

---

## Usage

Clone the repository:

```bash
git clone <repository-url>
cd <repository-name>
```

Run the script:

```bash
swift xiaomi_pen.swift
```

You should see the Xiaomi Pad HID device being detected.

Keep the script running while using the tablet:

```text
Ctrl-C to stop.
```

The script searches for the HID device when it starts, so changing the physical USB port should not matter as long as macOS detects the tablet normally.

If the tablet is disconnected while the script is already running, restart the script after reconnecting it.

---

## Expected Behavior

Without the fix:

```text
Light touch
    ↓
Tip = 1
    ↓
macOS does not click
    ↓
Increase pressure
    ↓
MouseDown
```

With the fix:

```text
Light touch
    ↓
Tip = 1
    ↓
MouseDown immediately
    ↓
Native pressure continues normally
    ↓
Tip = 0
    ↓
MouseUp
```

The result should be:

- ✅ much lighter activation force;
- ✅ no missing beginning of strokes;
- ✅ continuous strokes when pressure changes;
- ✅ native 0–8191 pressure information preserved;
- ✅ native pen coordinates preserved;
- ✅ native tilt information preserved;
- ✅ cursor remains at the last valid position when the pen leaves the sensing range.

---

## Why Does This Happen?

The exact root cause is not yet confirmed.

Based on the HID data, the most likely explanation is a compatibility issue somewhere in the following path:

```text
Xiaomi Pen
    ↓
Xiaomi Pad
    ↓
USB HID Digitizer
    ↓
macOS IOHID
    ↓
Digitizer → Pointer/Mouse event translation
```

The raw HID data looks reasonable: macOS receives low pressure values and the Tip Switch correctly.

The unexpected behavior appears later, when the digitizer contact is translated into a macOS mouse/pointer click.

It is therefore possible that this could eventually be fixed by either:

- a Xiaomi Pad firmware / HyperOS update; or
- improved macOS handling of this HID device.

This project should be considered a **workaround rather than a device driver**.

---

## Limitations

- Currently intended for the tested Xiaomi Pad HID behavior.
- Other Xiaomi tablets may use different Vendor IDs, Product IDs, report layouts, or HID usages.
- The script must currently be started manually.
- Device reconnection while the script is running may require restarting the script.
- This is an experimental workaround and has not been extensively tested across all macOS versions or drawing applications.

---

## Debugging

If your Xiaomi Pad is detected but the fix does not work, inspect its HID Digitizer elements and compare:

```text
Usage Page
Usage
Logical Min / Max
Report ID
Report Size
Report Count
```

In particular, look for:

```text
Usage Page 0x0D
Usage 0x30    Tip Pressure
Usage 0x42    Tip Switch
```

If your device exposes different HID fields, please open an issue and include the HID element information.

---

## Contributing

Reports from other Xiaomi Pad models are welcome.

When reporting compatibility, it is useful to include:

```text
Xiaomi Pad model:
macOS version:
Vendor ID:
Product ID:
Tip Pressure logical range:
Tip Switch usage:
Works / does not work:
```

This can help determine whether the workaround can be generalized to other Xiaomi tablets.

---

## Disclaimer

This project is an unofficial workaround and is not affiliated with or endorsed by Xiaomi or Apple.

Use it at your own risk.
