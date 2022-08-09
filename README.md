# ArchLinuxARM aarch64 RetroArch

This is my current setup for RetroArch using ArchLinuxARM aarch64 on Raspberry Pi
4B. You need to understand some basic Linux setup to get this working.

## Packages

The following is the output of:
`comm -3 <(pacman -Sp --print-format %n base | sort -u) <(pacman -Qqe | sort)`
That is, the list of explicitly installed packages not in the `base` group.

	# Bluetooth support
	bluez                   # For Bluetooth controller support
	bluez-utils             # For bluetoothctl, see below

	# Raspberry Pi specific
	raspberrypi-bootloader  # RPi bootloader
	raspberrypi-firmware    # RPi firmware
	firmware-raspberrypi    # Firmware RPi (?)
	linux-rpi               # We need the official RPi kernel fork, mainline won't cut it

	# RetroArch
	retroarch               # The stuff we want
	libxinerama             # Requirement for RetroArch (unsure why it isn't auto)
	libxrandr               # Requirement for RetroArch (unsure why it isn't auto)

	# Support
	networkmanager          # See below
	wpa_supplicant          # WiFi connection
	openssh                 # SSH for maintenance
	sudo                    # Sudo is useful, but you might just login as root
	vim                     # The Best Editor Ever

	# Pretty boot screen
	plymouth-git            # Compile and install plymouth-git from AUR

You can get away without NetworkManager if you configure wpa_supplicant directly.
From my experience, NetworkManager is better if the radio signal is weak where
the device will be sitting. I also wrote a small service which attempts to force
wifi to keep connected by restarting NetworkManager periodically. For some reason
I was not able to keep wifi connected using wpa_supplicant alone, so I sticked
with NetworkManager.

## Scripts

I wrote a few support scripts in order to help with the overall user experience.

### `mgmt`

A small maintenance script spawned by plugging in some specific device in one of
the pi's USB ports. This is configured by an udev rule (`mgmt.rules`). You can
choose any USB device you want (in my case, an USB stick).

To pick the correct udev values for your device, do this:

1. Run `udevadm monitor --udev --environment`;
2. Plug in your device, and take note of whatever udev variables you want
   (`PRODUCT` is the best candidate, since it's the `vid:pid` USB ID of your
   device);
3. Replace these values in `mgmt.rules`.

Put `mgmt` somewhere suitable (`/usr/local/sbin` is probably fine), and in
`/etc/mgmt.conf` configure which services to start and stop when the USB device
is plugged in and removed (`KEY_ON_START` and `KEY_OFF_STOP`).

### `btconn`

This is a service for automatic pairing of specific Bluetooth devices
(controllers, for example). Put `btconn.service` in your `/etc/systemd/system`
directory and `btconn` in `/usr/local/sbin`, and put your devices' MACs in
`btconn.conf`.

`btconn` works by starting `bluetoothctl` and parsing its output. (I have not
found a better way to do this in a shell script.) It manually removes all
devices' link keys (BlueZ does not have an API for this) and does the pairing
process again. This is useful, for example, if you share controllers with other
consoles as in my case.

If you set `btconn` as one of the services in `mgmt`, every time your configured
USB device is plugged in, your pi will re-pair with the specified Bluetooth
devices. As soon as you remove the USB device, the pi will stop the pairing mode.

Note: you cannot leave pairing or scanning mode always on, as this greatly
affects Bluetooth performance.

## Plymouth

[plymouth-git](https://aur.archlinux.org/packages/plymouth-git) compiles cleanly
on aarch64 with `makepkg -A`. You will need to install all necessary devtools for
it, though, so you might just not care enough to bother.

If you do, you might want to take a look at some [RetroArch Plymouth
themes](https://github.com/HerbFargus/plymouth-themes).

You need to add `splash` to your command-line (`/boot/cmdline.txt`) and also
remove `plymouth.enabled=0`. Also take a look at [more ways to keep your boot
quiet](https://wiki.archlinux.org/title/Silent_boot) to further enjoy it.

## RetroArch

To start RetroArch, I suggest just creating a service for that:

	# /etc/systemd/system/retroarch.system
	# Remember to create a `retroarch` user & group.

	[Unit]
	Description=RetroArch
	ConditionPathExists=/dev/fb0

	[Service]
	User=retroarch
	Group=retroarch
	Environment=HOME=/home/retroarch
	WorkingDirectory=/home/retroarch
	ExecStart=/usr/bin/retroarch

	[Install]
	WantedBy=multi-user.target

## `config.txt`

The essential setup for pi's firmware is:

	dtparam=audio=on        # Enable audio
	dtparam=krnbt=on        # Enable bluetooth
	dtoverlay=vc4-fkms-v3d  # Enable FKMS

Newer kernels have better support for non-fake KMS, but I could not get the audio
to work. Stick with the basics and you should have no problem.

## Other notes

I have many other non-essential customizations in my setup but this is enough to
make the magic work.

I suggest putting all emulation material (ROMs, system files, thumbnails, saves)
in a separate partition and configure RetroArch to just look there. This way you
can format your system if needed without losing stuff.

The pi is pretty limited so I recommend sticking to 1080p as a default for
RetroArch (you can configure it in the GUI). I might actually write a small
optimization guide later for good defaults + overclocking the pi to get even the
N64 running as smoothly as possible.
