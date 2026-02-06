# Kmonad config

 - home row mods
 - swap `esc` and `caps`
 - hold `esc` for nav layer
 - hold `spc` for layer1

## Run a Kmonad Service

To have `kmonad` run automatically, you need to set it up as a service. For the built-in keyboard, this must be a **system service** so that it can take control of the keyboard at the login screen.

### 1. Isolate the Keyboard from the Display Server

First, you must prevent your graphical environment (Xorg or Wayland) from grabbing the keyboard. This allows `kmonad` to get the exclusive access it needs.

#### For Wayland (Gnome, etc.)
Create a `udev` rule to tell `libinput` (the input library used by Wayland) to ignore the keyboard.

1.  Find a unique attribute for your keyboard, like its physical address:
    ```sh
    # Replace 'eventX' with your keyboard's actual event device
    udevadm info -a -n /dev/input/eventX | grep 'phys'
    ```

2.  Create the file `/etc/udev/rules.d/99-kmonad-ignore.rules` with a rule matching your device.
    ```udev
    # This rule is for the built-in laptop keyboard.
    # Replace the ATTRS{phys} value if your keyboard is different.
    ACTION=="add|change", KERNEL=="event*", ATTRS{phys}=="isa0060/serio0/input0", ENV{LIBINPUT_IGNORE_DEVICE}="1"
    ```

#### For Xorg
Create an `xorg.conf.d` file to tell Xorg to ignore the keyboard.

1.  Create the file `/etc/X11/xorg.conf.d/99-kmonad.conf`:
    ```
    Section "InputClass"
        Identifier "kmonad keyboard"
        # Match your keyboard's device path
        MatchDevicePath "/dev/input/by-path/platform-i8042-serio-0-event-kbd"
        Option "Ignore" "on"
    EndSection
    ```

After creating the appropriate file, reload the `udev` rules and reboot.
```sh
sudo udevadm control --reload-rules && sudo udevadm trigger
# A reboot is required for the display server to release the device.
```

### 2. Define a service for the built-in keyboard

Because the display server now ignores the keyboard, `kmonad` *must* be running at the login screen to provide input.

1.  Create the system service file: `/etc/systemd/system/kmonad.service`
    ```
    [Unit]
    Description=KMonad keyboard remapping (system service)

    [Service]
    # Replace 'cleaver' with your actual username
    User=cleaver
    ExecStart=/usr/bin/kmonad /home/cleaver/.config/kmonad/config.kbd
    Restart=always
    RestartSec=3

    [Install]
    WantedBy=graphical.target
    ```

2.  Enable and start the service:
    ```sh
    sudo systemctl daemon-reload
    sudo systemctl enable kmonad.service
    sudo systemctl start kmonad.service
    ```


### Trigger on Keyboard Connect:

This will set up an external keyboard with kmonad and trigger the service when it is plugged in. It uses `udev` to trigger the service for the newly attached keyboard.

1. Find the `idVendor` and `idProduct` for the keyboard:

```
lsusb
```

2. Define, enable, and test a new service for external keyboard in `.config/kmonad/kmonad-ext.service`
   Note that there are no restart lines, since that is controlled by `udev`.
```
[Unit]
Description=KMonad keyboard remapping

[Service]
Type=simple
ExecStart=/usr/bin/kmonad %h/.config/kmonad/aapl-config.kbd

[Install]
WantedBy=default.target
```

3. Create a kmonad config specifically for the keyboard. The input device file will be a symlink created when the keyboard is attached:

```
input (device-file "/dev/input/aapl-keyboard")
```

4. Create a udev rule to trigger the creation of the symlink. In the file `/etc/udev/rules.d/99-aapl-keyboard.rules`:

```
ACTION=="add", SUBSYSTEM=="input", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", SYMLINK+="input/aapl-keyboard"
```

    (`idVendor` and `idProduct` were found in step 1.)

3. Create a file: `/etc/udev/rules.d/99-kmonad-keyboard.rules`

```
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", TAG+="systemd", ENV{SYSTEMD_USER_WANTS}="kmonad-ext.service"
ACTION=="remove", SUBSYSTEM=="usb", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", RUN+="/bin/systemctl --user stop kmonad-ext.service"
```

4. Refresh rules:

```
sudo udevadm control --reload-rules
```


## Testing

To check the key codes from keyboard:

```
sudo evtest
```
