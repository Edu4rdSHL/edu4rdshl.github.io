---
layout: post
title: 'Improving your VirtualBox VMs I/O on Linux'
date: 2026-02-16 22:48 -0500
categories: ["virtualization", "linux", "virtualbox"]
tags: ["performance", "optimization", "vms"]
author: edu4rdshl
image:
  path: /2026-02-16/virtualbox-storage-settings.png
  alt: VirtualBox storage settings
excerpt: A quick TLDR to greatly improve the performance of your VirtualBox VMs on Linux by changing the storage controller and disk type.
---

**TLDR:** If you have your VMs in SSD or NVMe storage (being the latter an SSD anyways but just for clarity), change the I/O controller to "NVMe" and enable the "Solid State Drive" option for the VM disk (the VDI one, usually) in the storage settings of your VM. This will improve the performance of your VM by a lot. Please read the [notes](#notes) at the end of the post as there are some issues with Windows VMs when using the NVMe controller.

Usually my posts have a well structured introduction, problem, body, and conclusion. This one is going to be a bit different. I just want to share a quick tip that improves the performance VirtualBox VMs on Linux by a lot and that I discovered recently (QEMU was my previous go-to for virtualization but Windows VMs haven't been playing nice there).

## Speeding things up

If you have spinned up VMs in VirtualBox, they usually feel sluggish, especially Windows VMs, no matter if you have the latest SSD or NVMe. The root cause of this is that the default storage controller for VirtualBox VMs (AHCI) uses defaults that are aimed to work everywhere. These defaults are not optimized for modern storage devices, and that's why I/O performance is limited even on fast storage.

To solve it, follow the steps below:

1. Open your VM settings.
2. Go to the "Storage" section.
3. Select the "Controller: AHCI" (or whatever controller you have) and change the "Type" to "NVMe".
4. Select the disk (the VDI file, usually) and enable the "Solid State Drive" option.
5. Optionally enable the "Use Host I/O Cache" option for the disk. Read the notes below before enabling it.
6. Save the settings and start your VM.

With this change I passed from ~300 MB/s read/write speeds to ~4.2 GB/s on my NVMe drive, which is a huge improvement and makes the VM feel much faster and more responsive.

## Notes

1. Windows VMs may require additional drivers to work properly with the NVMe controller and don't work by default. However, setting only the "Solid State Drive" option for the disk already provides most of the performance boost, so you can try that first and see if it works for your Windows VM.
2. You can optionally enable the "Use Host I/O Cache" option for the disk if you have enough RAM. It will improve raw performance on the machine, but be aware that it may cause data loss if the host crashes or loses power. More information can be found in the [VirtualBox documentation](https://www.virtualbox.org/manual/ch05.html#iocaching).

And that's it. Now your machines will feel much faster, especially when it comes to disk I/O, which improves the overall performance of the VM significantly.
