# Summary
Notes to install Ubuntu 24.04LTS on my 2017 MacBook Pro 15 inch (MacBookPro14,3).

Now everything except the Bluetooth, TouchID (Fingerprint), Suspend and Hibernation seems to work for me.

## Sources
- [Rob Hills's original Gist] - https://gist.github.com/rob-hills/9134b7352ee7471c4d4f4fbd6454c4b9 - very helpful

## Out of the Box

### What worked
- The basic keyboard (not the Touchbar with Esc and Function keys though); 
- The touchpad;
- USB-C for important infrastructure such as LAN connection, external monitor, and external storage.

### What didn't work
- Wifi: It was partly functional, but was not fully usable. Only 2.4Ghz access points shown with poor level and can be connected;
- Bluetooth;
- Touchbar;
- Sound;
- Camera.

### What still doesn't work
- Suspend (Can not resume many hardware);
  2016 version seems to work with following method: https://ubuntuforums.org/showthread.php?t=2492426
  But, 2017 version seems doesn't work.
- Hibernation (When I execute `systemctl hibernate`, it seems shutting down);
- TouchID (It seems diffucult. There might be a possibility. Please see this: https://github.com/Dunedan/mbp-2016-linux/issues/71#issuecomment-528545490);

The following notes mostly document what worked to get Wifi, Touchbar, Camera, and Sound working.

## Pre-requirement
**The MacBook's EFI partition must be kept.** Please see the following comment for more information.
https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7?permalink_comment_id=4937505#gistcomment-4937505
If you have deleted the EFI partition you can reboot and use Option + Command + R to do an Internet Recovery to MacOS Ventura.  This will restore the EFI partition and then you can re-install Ubuntu.

## Resolution

### WiFi
Just creating the configuration file `/usr/lib/firmware/brcm/brcmfmac43602-pcie.txt` was enough for my hardware.

[*Andy Holst's* example configuration file final iteration here](https://bugzilla.kernel.org/show_bug.cgi?id=193121#c74) was ideal and only required a little tweaking for my system.

#### WiFi Driver Configuration tweaks
I made the following changes to Andy Holst's `/usr/lib/firmware/brcm/brcmfmac43602-pcie.txt`
```
macaddr=00:90:4c:0d:f5:30
ccode=ALL
```

- *macaddr* - set to the mac address of my WiFi card (I got it with the command `ip addr`)
- *ccode* - I just simply set it to ALL, (ALL (Channel 1-14), US, EU etc.)

### Touchbar

#### Step-by-Step Touchbar
Become the superuser
```
sudo su
```

Ubuntu uses initramfs so add our new modules to the list to be loaded
```
cat <<EOF | tee -a /etc/initramfs-tools/modules
# drivers for keyboard+touchpad
applespi
apple-ib-tb
intel_lpss_pci
spi_pxa2xx_platform
EOF
```

Build and install drivers from the source code.
```
apt install dkms

cd {your preferred source download folder}
git clone https://github.com/almas/macbook12-spi-driver
cd macbook12-spi-driver
git checkout touchbar-driver-hid-driver
ln -s `pwd` /usr/src/applespi-0.1
dkms install applespi/0.1
```

Test the drivers by loading them and their dependencies
```
modprobe intel_lpss_pci spi_pxa2xx_platform applespi apple-ib-tb
```
An empty output indicates success

Reboot

At this point of the process, my touchbar was not working.  More Googling led me to someone else logging this as [an issue on RoadRunner2's Driver repository[(https://github.com/roadrunner2/macbook12-spi-driver/issues/42] and in the discussion I found a workaround.  The following commands:
```
echo '1-3' | sudo tee /sys/bus/usb/drivers/usb/unbind
echo '1-3' | sudo tee /sys/bus/usb/drivers/usb/bind
```
caused my touchbar to light up!!

To save having to run these commands each time I start the computer, I needed to create a "macbook-quirks.service" and have it start up with the computer:
```
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

My MBP now boots up with the Touchbar working!

From the discussion in the link above, it appears this may be due to a bug in Linux and the usbmuxd system so hopefully will be able to remove the workaround in the future, so keep an eye on the [relevant issue discussion](https://github.com/roadrunner2/macbook12-spi-driver/issues/42).

#### Touchbar Tweaking (Optional)
Personally, I don't like this. But, if you want the Function keys to appear by default (instead of the brightness and sound keys), then you need to pass some parameters to the apple_ib_tb module. 
'modinfo apple_ib_tb' lists the parameters and what they do.
```
sudo su
cat <<EOF | tee /etc/modprobe.d/apple_ib_tb.conf
options apple_ib_tb fnmode=2           # Default to Function Keys, Fn key toggles to "special"
options apple_ib_tb idle_timeout=60    # Turn off the Touchbar after 60 seconds.
EOF
```

After reloading the module with the following command, the Function keys will be shown by default.
```
sudo modprobe -r apple_ib_tb
sudo modprobe apple_ib_tb
```

### Sound
I found the resolution from the Rade0nFighter's answer from the following QA:
https://askubuntu.com/questions/1475091/sound-macbook-pro-ubuntu-22-04

```
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
Now the sound should work.


### Camera
Execute the following command and reboot.
```
echo "options uvcvideo quirks=0x100" > /etc/modprobe.d/uvcvideo.conf
```
Now the camera should work too.


## References
- https://gist.github.com/rob-hills/9134b7352ee7471c4d4f4fbd6454c4b9
- https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7
- https://github.com/Dunedan/mbp-2016-linux
