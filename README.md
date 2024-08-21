# Summary
Notes to install Ubuntu 24.04LTS on my 2017 MacBook Pro 15 inch (MacBookPro14,3).

Everything except the Bluetooth, TouchID (Fingerprint), Suspend and Hibernation are working.

## Pre-requirement
**The MacBook's EFI partition must be kept.** 
Please see the following comment for more information.
https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7?permalink_comment_id=4937505#gistcomment-4937505

If you have deleted the EFI partition you can reboot and use Option + Command + R to do an Internet Recovery to MacOS Ventura.  This will restore the EFI partition and then you can re-install Ubuntu.

## Out of the Box

### What worked
- The basic keyboard (not the Touchbar with Esc and Function keys though); 
- The touchpad;
- USB-C for important infrastructure such as LAN connection, external monitor, and external storage.

### What didn't work
- Wifi: It was partly functional, but was not fully usable. Only 2.4Ghz access points shown with poor level and can be connected;
- Touchbar;
- Sound;
- Camera.

### What still doesn't work
- Bluetooth - this should work with this patch https://github.com/leifliddy/macbook12-bluetooth-driver however that is broken with the 6.8.0 kernel, see: https://github.com/leifliddy/macbook12-bluetooth-driver/issues/24
- Suspend (Can not resume many hardware);
  2016 version seems to work with following method: https://ubuntuforums.org/showthread.php?t=2492426
  But, 2017 version seems doesn't work.
- Hibernation (When I execute `systemctl hibernate`, it seems shutting down);
- TouchID (It seems difficult, but there might be a possibility. See: https://github.com/Dunedan/mbp-2016-linux/issues/71#issuecomment-528545490);

The following notes mostly document what worked to get Wifi, Touchbar, Camera, and Sound working.

## Resolution

### WiFi
Copy https://github.com/marvinrobot78/MacBookPro-14-3-Ubuntu/blob/main/brcmfmac43602-pcie.txt to /lib/firmware/brcm/ and reboot.

### Touchbar

#### Step-by-Step Touchbar
Become the superuser
```bash
sudo su
```

Ubuntu uses initramfs so add our new modules to the list to be loaded
```bash
cat <<EOF | tee -a /etc/initramfs-tools/modules
# drivers for keyboard+touchpad
applespi
apple-ib-tb
intel_lpss_pci
spi_pxa2xx_platform
EOF
```

Build and install drivers from the source code.
```bash
apt install dkms git

cd {your preferred source download folder}
git clone https://github.com/almas/macbook12-spi-driver
cd macbook12-spi-driver
git checkout touchbar-driver-hid-driver
ln -s `pwd` /usr/src/applespi-0.1
dkms install applespi/0.1
```

Test the drivers by loading them and their dependencies
```bash
modprobe intel_lpss_pci spi_pxa2xx_platform applespi apple-ib-tb
```
An empty output indicates success.  Reboot.

To get the touchbar to light up:
```bash
echo '1-3' | sudo tee /sys/bus/usb/drivers/usb/unbind
echo '1-3' | sudo tee /sys/bus/usb/drivers/usb/bind
```

Create a "macbook-quirks.service" and have it start up with the computer:
```bash
sudo su
cat <<EOF | tee /etc/systemd/system/macbook-quirks.service
[Unit]
Description=Re-enable MacBook 14,3 TouchBar
Before=display-manager.service

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 2
ExecStart=/bin/sh -c "echo '1-3' > /sys/bus/usb/drivers/usb/unbind"
ExecStart=/bin/sh -c "echo '1-3' > /sys/bus/usb/drivers/usb/bind"
RemainAfterExit=yes
TimeoutSec=0

[Install]
WantedBy=multi-user.target
EOF
```
Now enable with `systemctl enable macbook-quirks.service` and reboot to check.

#### Touchbar Tweaking (Optional)
If you want the Function keys to appear by default (instead of the brightness and sound keys), then you need to pass some parameters to the apple_ib_tb module. 
'modinfo apple_ib_tb' lists the parameters and what they do.
```bash
sudo su
cat <<EOF | tee /etc/modprobe.d/apple_ib_tb.conf
options apple_ib_tb fnmode=2           # Default to Function Keys, Fn key toggles to "special"
options apple_ib_tb idle_timeout=60    # Turn off the Touchbar after 60 seconds.
EOF
```

After reloading the module with the following command, the Function keys will be shown by default.
```bash
sudo modprobe -r apple_ib_tb
sudo modprobe apple_ib_tb
```
### Adjust Trackpad Sensitivity

Make file

```bash
sudo nano /usr/share/libinput/local-overrides.quirks
```

Add content:

```bash
[MacBook(Pro) SPI Touchpads]
MatchName=*Apple SPI Touchpad*
ModelAppleTouchpad=1
AttrTouchSizeRange=200:150
AttrPalmSizeThreshold=1100

[MacBook(Pro) SPI Keyboards]
MatchName=*Apple SPI Keyboard*
AttrKeyboardIntegration=internal

[MacBookPro Touchbar]
MatchBus=usb
MatchVendor=0x05AC
MatchProduct=0x8600
AttrKeyboardIntegration=internal
```

Register the file

```bash
sudo systemd-hwdb update
```

### Sound
```bash
sudo su

apt install wget make gcc linux-headers-generic
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
./install.cirrus.driver.sh

# It will give you instructions to install package linux-source-XXXXX. Install it.
apt install linux-source-XXXXX

# Then re-run install script.
./install.cirrus.driver.sh

reboot
```


### Camera
```bash
echo "options uvcvideo quirks=0x100" > /etc/modprobe.d/uvcvideo.conf
```


## References
- https://github.com/leifliddy/macbook12-bluetooth-driver
- https://gist.github.com/almas/5f75adb61bccf604b6572f763ce63e3e
- https://gist.github.com/rob-hills/9134b7352ee7471c4d4f4fbd6454c4b9
- https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7
- https://github.com/Dunedan/mbp-2016-linux
