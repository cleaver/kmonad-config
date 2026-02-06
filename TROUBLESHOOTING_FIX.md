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

# Fixing Permissions and Login Issues

## The Problem

Three main issues can occur when setting up `kmonad` for a built-in keyboard on a modern Linux system (e.g., with Wayland):

1.  **Permission Denied on `/dev/uinput`**: `kmonad` fails with a `permission denied` error because it cannot access the `uinput` device to create a virtual keyboard.

2.  **"Could not perform IOCTL grab" Error**: `kmonad` starts but immediately fails because the graphical display server (Xorg/Wayland) already has exclusive control (a "grab") of the keyboard.

3.  **Keyboard not working at Login Screen**: After fixing the "grab" issue by telling the display server to ignore the keyboard, the keyboard doesn't work at the login screen, making it impossible to log in without an external keyboard.

## The Solution

A series of steps are needed to correctly configure permissions and services.

### 1. Fix `/dev/uinput` Permissions

The user running `kmonad` needs write access to `/dev/uinput`. This can be done with a `udev` rule that is compatible with modern `udev` versions that are particular about group permissions.

Create `/etc/udev/rules.d/99-kmonad.rules`:
```udev
# Set permissions directly as modern udev can ignore GROUP assignments for non-system groups
KERNEL=="uinput", RUN+="/bin/chmod 0660 /dev/uinput", RUN+="/bin/chgrp uinput /dev/uinput"
```
Then, ensure the user who will run `kmonad` is in the `uinput` group:
```sh
sudo usermod -aG uinput your_username
```

### 2. Isolate the Keyboard from the Display Server (Fix "Grab" issue)

This is done by telling the display server to ignore the device, freeing it up for `kmonad`.

#### For Wayland (Gnome, etc.)
Create a `udev` rule to set the `LIBINPUT_IGNORE_DEVICE` environment variable.

File: `/etc/udev/rules.d/99-kmonad-ignore.rules`
```udev
# This rule targets a specific built-in keyboard via its physical address
ACTION=="add|change", KERNEL=="event*", ATTRS{phys}=="isa0060/serio0/input0", ENV{LIBINPUT_IGNORE_DEVICE}="1"
```
*(The `phys` attribute should be confirmed for your specific device using `udevadm info`).*

### 3. Run `kmonad` as a System Service (Fix Login issue)

To ensure the keyboard works at the login screen, `kmonad` must run as a system service that starts at boot, rather than a user service that starts after login.

1.  **Create a system service file** at `/etc/systemd/system/kmonad.service`:
    ```
    [Unit]
    Description=KMonad keyboard remapping (system service)

    [Service]
    # Run as your user, not root
    User=cleaver
    ExecStart=/usr/bin/kmonad /home/cleaver/.config/kmonad/config.kbd
    Restart=always
    RestartSec=3

    [Install]
    WantedBy=graphical.target
    ```

2.  **Disable the old user service** and **enable the new system service**:
    ```sh
    systemctl --user disable kmonad.service
    sudo systemctl daemon-reload
    sudo systemctl enable kmonad.service
    ```

This sequence ensures `kmonad` is running with the correct permissions before a user logs in, providing keyboard input at the login screen and within the user session.