---
layout: default
title: "Autel EVO Nano+: getting a root shell"
---

# Autel EVO Nano+: getting a root shell

Autel dropped support for the EVO Nano series. No more firmware, no more downloads, nothing. If you own one, you are on your own now. That is the bad news.

The good news is the drone, the controller, and the camera are three little Linux and Android boxes wired together over a normal IP network, and you can get a root shell on all of them in a matter of minutes. No soldering, no JTAG, no special gear. Just the drone, a microSD card, and a computer with WiFi.

This is part one. The goal here is small and specific: get on the network and open a shell on each of the three nodes. Later posts go further.

## The layout

Three systems, one network, fixed addresses:

- **Drone** (the aircraft) at `192.168.1.1`. The flight computer, running Android.
- **Controller** (the remote) at `192.168.1.5`. Same board as the drone, also Android.
- **Camera** (on the gimbal) at `192.168.1.11`. Runs Linux and handles imaging and video.

Those addresses do not change. They are set at boot, every time.

## Step 1: get on the WiFi

The camera runs a WiFi access point. It is meant for pulling photos and clips off the aircraft, but once you are on it you are on the internal network with everything else.

Turn it on at boot:

Put an empty file named `enable_wifi` on the root of the drone's microSD card, then put the card back in the aircraft and power on. On linux you can use:

```sh
touch /path/to/sdcard/enable_wifi
```

The aircraft sees the file and brings the AP up by itself.

The network shows up as `Autel-XXXXXX`, where the last six characters come from the camera's MAC. Mine is not `Autel-587042`. The password is baked into the firmware and it is the same on every unit:

```
12345678
```

Join it from a PC like any other network and you will land in the `192.168.1.0/24` range.

## Step 2: open a shell on each node

**Camera.** SSH as root:

```sh
ssh root@192.168.1.11
```

It runs an old SSH server, so a current client may refuse the handshake. If that happens, let the legacy algorithms through:

```sh
ssh -o HostKeyAlgorithms=+ssh-rsa \
    -o KexAlgorithms=+diffie-hellman-group14-sha1 \
    -o PubkeyAcceptedAlgorithms=+ssh-rsa \
    root@192.168.1.11
```

**Drone.** Telnet drops you straight into a root shell. No login, no password:

```sh
telnet 192.168.1.1
```

**Controller.** Same thing, as long as the remote is powered on and linked to the drone:

```sh
telnet 192.168.1.5
```

Done. Root on all three.

## A few things to know

The controller only shows up when it is on and bound to the drone. Its link to the aircraft is the radio, not WiFi, but that link is bridged onto the same `192.168.1.0/24` network, so once the remote is talking to the drone it answers at `.5` like any other host.

Everything here is root with full read and write. It is easy to brick something. Pull a backup before you change anything on these boxes.

And the obvious one: this is for your own aircraft. Leave other people's gear alone, and leave anything that keeps flights legal where it is.
