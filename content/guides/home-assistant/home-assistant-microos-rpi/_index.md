---
title: Setting up Home Assistant with openSUSE MicroOS on a Raspberry Pi
resources:
  - name: port-authority
    src: "port_authority.jpg"
    title: Port authority scan revealing the Raspberry Pi
---

[openSUSE MicroOS](https://microos.opensuse.org/) is a small operating system
designed to run container workloads. It features a read only root partition,
transactional updates which allow for easy rollbacks thereby making it highly
resilient and perfect for running small IoT devices.

Home Assistant can be run very easily from within a container preferably on a
system requiring as little maintenance as possible. This makes MicroOS the perfect
operating system for deploying Home Assistant.

In this guide, we will show how to install MicroOS on your Raspberry Pi,
configure it with ignition and deploy Home Assistant using podman and systemd.


## Prerequisites

You will need the following hardware:
1. Raspberry Pi 4 + Power Supply
2. Micro SD-Card
3. USB drive
4. PC with a SD-Card reader and a Micro SD Adapter
5. Ethernet cable

This guide will assume that your PC is running Linux. You can perform most of
these steps on Windows or MacOS, but some of the commands will require
adjustment.


## Install MicroOS

1. Download the latest MicroOS image (aarch64 variant, Base System + Container
   Runtime Environment) from
   [download.opensuse.org](https://download.opensuse.org/ports/aarch64/tumbleweed/appliances/openSUSE-MicroOS.aarch64-ContainerHost-RaspberryPi.raw.xz)

2. Insert the SD Card into your PC and find out which device name corresponds to
   it, e.g. via `udiskctl status` or `lsblk`:

```ShellSession
 ❯ udisksctl status
MODEL                     REVISION  SERIAL               DEVICE
--------------------------------------------------------------------------
ADATA SX8200PNP           42G1TBKA  2L08294NSJU9         nvme0n1
ST2000LM007-1R8174        SDM2      WDZG289H             sda
SE32G                               0x0f6986e5           mmcblk0

 ❯ # The micro SD Card would be mmcblk0
```

3. Flash the downloaded image to the correct device. Double check the correct
   path or you might accidentally overwrite important data!
```ShellSession
 ❯ xzcat /path/to/download/folder/openSUSE-MicroOS.aarch64-ContainerHost-RaspberryPi.raw.xz | \
       dd bs=4M of=/dev/mmcblk0 iflag=fullblock oflag=direct status=progress
# wait a bit
 ❯ sync
```

4. Grab the spare USB drive and either create a new partition on it with the
   label `ignition` or relabel the existing partition as `ignition` (e.g. via
   `gparted`).
   Create a directory called `ignition` in the root directory of the USB drive
   and put the file `config.ign` with the following contents into it:
```json
{
  "ignition": { "version": "3.0.0" },
  "passwd": {
    "users": [
      {
        "name": "root",
        "sshAuthorizedKeys": [
          "ssh-rsa insertYourPublicKeyHere"
        ]
      }
    ]
  }
}
```
Replace the line `ssh-rsa insertYourPublicKeyHere` with your preferred ssh
public key that you can find in `~/.ssh/id_*.pub`.

5. Unmount the SD-Card and the USB drive, insert both into your Raspberry Pi,
   connect it to your local network using an Ethernet cable and power it on.

6. Get the IP address of your Raspberry Pi. You can use your router's web
   interface for this (if you have access to it). Alternatively, you can scan
   all PCs in your local network using nmap:
```ShellSession
 ❯ nmap -p22 -v 192.168.1.* # adjust to your IP range
 # lots of output...

Nmap scan report for 192.168.1.107
Host is up (0.00018s latency).

PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: B8:27:EB:64:2C:20 (Raspberry Pi Trading)

```
Or use any other port scanner,
e.g. [PortAuthority](https://github.com/aaronjwood/PortAuthority) for Android:


{{< img name="port-authority" size="small" >}}


7. Log in to your Pi and update all packages:
```ShellSession
 ❯ ssh root@$RPI_IP_HERE
localhost:~ # transactional-update
Checking for newer version.
transactional-update 4.0.0~rc4 started
Options:
Separate /var detected.
2022-08-18 13:35:48 tukit 4.0.0~rc4 started
2022-08-18 13:35:48 Options: -c1 open
2022-08-18 13:35:48 Using snapshot 1 as base for new snapshot 2.
2022-08-18 13:35:48 No previous snapshot to sync with - skipping
ID: 2
2022-08-18 13:35:49 Transaction completed.
Calling zypper up
zypper: nothing to update
Removing snapshot #2...
2022-08-18 13:35:52 tukit 4.0.0~rc4 started
2022-08-18 13:35:52 Options: abort 2
2022-08-18 13:35:52 Discarding snapshot 2.
2022-08-18 13:35:52 Transaction completed.
transactional-update finished
localhost:~ # reboot # only if packages were actually updated
```


## Setup Home Assistant

We will now setup Home Assistant to be run via podman in a systemd unit and
optionally enable automatic updates of the container image.

Log in to the Raspberry Pi and proceed as follows:

1. Create a directory for Home Assistant's configuration files and database and
   switch to it:
```ShellSession
 ❯ ssh root@$RPI_IP_HERE
localhost:~ # mkdir -p /path/to/conf/dir
localhost:~ # podman run -d -v /path/to/conf/dir:/config:Z \
      -v /etc/localtime:/etc/localtime:ro \
      --privileged --network=host \
      --name=homeassistant \
      --label "io.containers.autoupdate=registry" \
      ghcr.io/home-assistant/home-assistant:stable
```

2. Create a systemd unit using podman and enable it at startup. Now Home
   Assistant will be automatically started whenever the Raspberry Pi is
   rebooted.
```
localhost:~ # podman generate systemd --new homeassistant > \
                  /etc/systemd/system/homeassistant.service
localhost:~ # podman stop homeassistant
homeassistant
localhost:~ # systemctl daemon-reload
localhost:~ # systemctl enable --now homeassistant
Created symlink /etc/systemd/system/default.target.wants/homeassistant.service → /etc/systemd/system/homeassistant.service.
```

3. Depending on your preferences, you can enable automatic updates of the Home
   Assistant container image using [podman's
   auto-update](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html)
   command. podman will pull a new version of all images with the label
   `io.containers.autoupdate` set to `registry` every time `podman auto-update`
   is run. You can either run `podman auto-update` yourself whenever you
   consider safe to do or let systemd perform the update periodically by
   enabling the `podman-auto-update.timer`:
```ShellSession
localhost:~ # systemctl enable --now podman-auto-update.timer
Created symlink /etc/systemd/system/timers.target.wants/podman-auto-update.timer → /usr/lib/systemd/system/podman-auto-update.timer.
```

Home Assistant is now running on your Raspberry Pi and you can go through the
[onboarding process]([https://www.home-assistant.io/getting-started/onboarding/]).


## Going forward

We have only scratched the surface of what can be achieved with MicroOS as a
host operating system. There are multiple areas where this setup can be improved
further:
- Leverage (health-checker)[https://github.com/openSUSE/health-checker] to
  verify whether Home Assistant is able to launch with a new snapshot.
- Use [ignition](https://coreos.github.io/ignition/) to its full potential and
  setup Home Assistant completely automated via ignition.
- Setup proper monitoring

There is a lot to tinker with, so go and check out the [MicroOS Portal in the
openSUSE Wiki](https://en.opensuse.org/Portal:MicroOS) to find out more.
