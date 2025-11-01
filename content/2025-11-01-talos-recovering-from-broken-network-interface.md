---
title: "Talos: recovering from broken network interface"
date: 2025-11-01T23:00:00+02:00
---

While playing with my homelab Talos cluster, I accidentaly broke the network interface. As my Raspberry Pi has only one interface, that meant that my device was unreachable over the network. To make matter worse, the node I broke was the control plane, and I wanted to preserve as much data as possible.

To recover from this situation, I was left with no other option, but to boot the device from an SD card. When Talos is installed and first boots up, it creates a couple of partitions on the install disk (`nvme0n1` in my case):

```text
rpi@rpi1:~$ lsblk -o NAME,LABEL,SIZE,MOUNTPOINT
NAME        LABEL       SIZE MOUNTPOINT
mmcblk0                29.2G
├─mmcblk0p1 bootfs      512M /boot/firmware
└─mmcblk0p2 rootfs     28.6G /
nvme0n1               476.9G
├─nvme0n1p1 EFI         1.1G
├─nvme0n1p2               1M
├─nvme0n1p3 STATE       100M
├─nvme0n1p4 EPHEMERAL    50G
└─nvme0n1p5           425.8G
```

The one that I'm interesting it is `STATE`, which contains the Talos configuration (but not `etcd` which is stored in `EPHEMERAL`).

A simple fix is to "clean" the `STATE` partition and let Talos recreate it on the next boot. This can be done with the following command:

```nu
sudo mkfs.ext4 -L STATE /dev/nvme0n1p3
```

After that, I just needed to reboot the device (removing the SD card before) and apply valid configuration to the node with `--insecure` flag:

```nu
talosctl apply-config --nodes <node> --endpoints <node> --file <file> --insecure
```

Easy-peasy! If only all problems were this easy to solve :)
