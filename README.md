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
systemctl --user enable kmonad-ext.service
systemctl --user start kmonad-ext.service
```

### Trigger on Keyboard Connect:

1. Find the `idVendor` and `idProduct` for the keyboard:

```
lsusb
```

2. Define, enable, and test a new service for external keyboard.

3. Create a file: `/etc/udev/rules.d/99-kmonad-keyboard.rules`

```
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", TAG+="systemd", ENV{SYSTEMD_WANTS}="kmonad-ext.service"
ACTION=="remove", SUBSYSTEM=="usb", ATTRS{idVendor}=="05ac", ATTRS{idProduct}=="0220", TAG+="systemd", ENV{SYSTEMD_WANTS}="kmonad-ext.service"
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
