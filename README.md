# Kmonad config

 - home row mods
 - swap `esc` and `caps`
 - hold `esc` for nav layer
 - hold `spc` for layer1

## Run a Kmonad Service

### Define a service for built-in keyboard

1. Create file: `.config/systemd/user/kmonad.service`

```
[Unit]
Description=KMonad keyboard remapping

[Service]
Type=simple
ExecStart=/usr/bin/kmonad %h/.config/kmonad/config.kbd
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

2. Enable and start the service:

```
systemctl --user enable kmonad.service
systemctl --user start kmonad.service
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
