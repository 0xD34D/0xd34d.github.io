---
layout: default
title: "Autel EVO Nano+: SD Card Shenanigans"
---

# Autel EVO Nano+: SD Card Shenanigans

The EVO Nano's camera watches the microSD card for files with special names. Drop an empty file with the right name on the root of the card, put the card back in the aircraft, and the camera does something extra on boot: brings up WiFi, saves RAW photos, dumps its logs, switches the USB port, and so on. These are the factory and debug hooks the firmware shipped with, and they are handy once you know they exist.

The card goes in the aircraft, behind the gimbal, and it is the camera that reads these files, not the drone or the controller. This list came straight out of the camera firmware, so it is the real set and not guesswork.

## How it works

Two rules:

- The file goes at the root of the card. It can be empty. The name is the whole trick.
- Most names also accept a `.txt` version, so `enable_wifi` and `enable_wifi.txt` both work.

The camera checks for most of these the moment the card is inserted, so the usual flow is: create the file, put the card in, power on. A few are checked while the camera is running. A couple delete themselves after they run, which is a nice way to tell they worked.

## The useful ones

**`enable_wifi`**

Brings up the camera's WiFi access point without the app. This is the one from my previous [root shell writeup](drone-pwn). Join the `Autel-xxxxxx` network and you are on the drone's internal network.

**`usb2net`**

Switches the aircraft's USB-C port into a USB network (RNDIS) device. Plug it into a PC and you get a wired network link to the camera, no WiFi needed. One quirk: the camera does not hand out an IP on this link, because the USB switch bounces the bridge and the camera's DHCP server stops serving it. So set a static address on the PC side, for example `192.168.1.50` with netmask `255.255.255.0`, and you can reach the camera at `192.168.1.11` and the drone at `192.168.1.1`.

**`vision_flash`**

Puts the USB port into the factory calibration gadget mode. Same idea as `usb2net`, a different personality on the port.

**`copy_all_log`**

Copies the camera's entire `/mnt/log` directory onto the card and then deletes the trigger file. This is the easy way to pull every log off the camera. `copy_log` does the same thing.

**`photo_with_raw.txt`**

Makes the camera save a RAW (DNG) file next to every JPEG. This is the good one for stills. The flag turns on when the card is inserted and turns back off when you pull the card, so keep the file on the card you actually fly with. This one wants the `.txt` form.

**`save_raw`**

Dumps raw sensor frames to the card. This is the one to know if you want unprocessed sensor data, though the files are large. Its sibling `send_raw` sounds similar but does the opposite: it reads raw frame files off the card and feeds them back into the image pipeline (a factory ISP replay test), so it is not a way to pull data out to a PC.

**`IQCalibSetting.ini`**

Not quite what the name suggests. The contents do not matter. Its mere presence flips the camera into an image quality calibration mode: it skips the normal resolution remap and captures at the sensor's native resolution. An empty file is enough. Useful if you want full, unscaled frames.

**`Enable_Camera_Debug`**, **`Lens_Debug`**, and **`PDAF_Debug`**

Turn on extra debug logging for the camera, the lens, and the phase detect autofocus. Pair them with `copy_all_log` to pull the output.

**`Disable_Force_LowPower`**

Stops the camera from being forced into a low power state. It is checked both at boot and on insert.

**`.firmwareUpgrade`**

Kicks off a firmware update from the card. This one is a rabbit hole of its own, since it needs a manifest and the component images alongside it, so it is a topic for another day.

## *Bonus:* You do not even need to pull the card

The camera runs an FTP server on port 21 that serves the SD card, all the time, right next to the telnet and SSH servers. So once you are on the drone's WiFi (see the [root shell writeup](drone-pwn)), you can drop these trigger files onto the card over the network without ever taking it out:

```sh
ftp 192.168.1.11      # anonymous, lands in the SD card root
```

Upload the file you want, then reboot the aircraft so the insert check runs again. Same card, no screwdriver.

## Closing

This is all for your own aircraft. These are camera debug and factory hooks, so some of them produce big files or change how the camera captures, and one of them wipes calibration. Back up `/mnt/config` before you experiment, and remember that almost everything here is undone by removing the file and rebooting.
