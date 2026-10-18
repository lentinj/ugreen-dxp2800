# Debian on the UGreen DXP2800

Some notes on supporting the hardware on mainline Linux / Debian.

## BIOS

Enter with Ctrl-F12. Other keys seem to be disabled.

Disable watchdog before installing Debian, 

## Re-enabling watchdog

By default Linux loads the ``iTCO_wdt`` module, but the BIOS uses the watchdog in the IT8613.

Disable the former and load the appropriate driver:

```
cat <<EOF > /etc/modules-load.d/ugreen-watchdog.conf
it87_wdt
EOF

cat <<EOF > /etc/modprobe.d/ugreen-watchdog.conf
blacklist iTCO_wdt
EOF
```

You also need the watchdog package:

```
apt install watchdog
```

## Hardware monitoring

The IT8613 chip manages hardware monitoring, but there isn't a direct driver.
Arguably trying to have one is a bad plan, as you run the risk of talking to the hardware at the same time as the BIOS.

If you're feeling brave, you can at least enable some reads by forcing the driver to load:

```
cat <<EOF > /etc/modules-load.d/ugreen-hwmon.conf
it87
EOF

cat <<EOF > /etc/modprobe.d/ugreen-hwmon.conf
# The IT8613 is roughly like an 8622, so long as you're just reading temps/fan speed
options it87 force_id=0x8622
EOF
```

## LEDs

```
cat <<EOF > /etc/modules-load.d/ugreen-i2c.conf
# Make I2C busses available to userspace
i2c-dev
EOF
```

The LEDs are, in order power / netdev / disk1 / disk2:

```
./ugreen_leds_cli power -on -color 255 255 0
./ugreen_leds_cli netdev -on -color 0 255 0
./ugreen_leds_cli disk1 -on -color 0 0 255
./ugreen_leds_cli disk2 -on -color 255 0 255
```

```
wget https://github.com/miskcoo/ugreen_leds_controller/releases/download/v0.3/led-ugreen-dkms_0.3_amd64.deb
apt install dkms linux-headers-amd64
dpkg -i led-ugreen-dkms_0.3_amd64.deb
apt-get --fix-broken install
```

## Serial port

The serial port is on J1, an unpopulated 0.1" surface-mount header. The pins are, from the little arrow:

    |>  TX /  RX / GND

The serial port shows up at ``/dev/ttyS0``.

### Enabling a getty using systemd

```
systemctl enable getty@ttyS0
systemctl start getty@ttyS0
```

### Enabling serial console in GRUB

Add the following to ``/etc/default/grub``:

```
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0,115200"
GRUB_SERIAL_COMMAND="serial --unit=0 --speed=115200"
GRUB_TERMINAL_INPUT="console serial"
GRUB_TERMINAL_OUTPUT="gfxterm serial"
```

## I2C expansion port (RTC?)

On J5 (seems) to be an I2C expansion. The pins are, from the little arrow:

    |>  VCC / GND / CLK / DAT / RST

I manage to burn a trace out by shorting VCC, I didn't want a real time clock anyway.
