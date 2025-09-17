# Summary of KMonad and Sleep Issue Fix

This document summarizes the changes made to resolve a system sleep issue caused by the `kmonad-ext.service`.

## The Problem

The system was entering a rapid sleep-wake cycle, preventing it from suspending properly. The root cause was the `kmonad-ext.service` being stuck in a restart loop.

This happened because:
1.  The `kmonad` configuration (`aapl-config.kbd`) pointed to a non-stable device path (`/dev/input/by-path/...`). When the keyboard was disconnected or the system resumed from sleep, this path could disappear, causing `kmonad` to crash.
2.  The `systemd` service (`kmonad-ext.service`) was configured with `Restart=always`, which caused `systemd` to immediately try and restart the crashing `kmonad` process.
3.  The `udev` rule (`99-kmonad-keyboard.rules`) was misconfigured for the "remove" action, attempting to start the service instead of stopping it when the keyboard was disconnected.

This combination created a constant loop of service failures that blocked the system's sleep process.

## The Solution

A multi-step solution was implemented to make the service more robust and correctly manage its lifecycle.

### 1. Created a Stable Device Symlink

A new `udev` rule was created at `/etc/udev/rules.d/99-aapl-keyboard.rules` to create a stable symlink for the keyboard:

```udev
ACTION=="add", SUBSYSTEM=="input", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", SYMLINK+="input/aapl-keyboard"
```
This ensures there is always a consistent path (`/dev/input/aapl-keyboard`) for `kmonad` to use, regardless of the underlying system device path.

### 2. Updated KMonad Configuration

The `aapl-config.kbd` file was updated to use the new stable symlink:

```diff
-  input (device-file "/dev/input/by-path/pci-0000:07:00.3-usb-0:2.2:1.0-event-kbd")
+  input (device-file "/dev/input/aapl-keyboard")
```

### 3. Modified the Systemd Service

The `kmonad-ext.service` file was simplified to remove the problematic restart logic. The `Restart=always` and `RestartSec=3` lines were removed, handing over control of the service's lifecycle to `udev`.

### 4. Corrected the Udev Trigger Rule

The original `udev` rule at `/etc/udev/rules.d/99-kmonad-keyboard.rules` was updated to correctly start the service on device connection and explicitly stop it on removal:

```udev
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", TAG+="systemd", ENV{SYSTEMD_USER_WANTS}="kmonad-ext.service"
ACTION=="remove", SUBSYSTEM=="usb", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", RUN+="/bin/systemctl --user stop kmonad-ext.service"
```

These changes ensure that the `kmonad` service for the external keyboard starts and stops cleanly with the device, resolving the crash loop and allowing the system to sleep normally.
