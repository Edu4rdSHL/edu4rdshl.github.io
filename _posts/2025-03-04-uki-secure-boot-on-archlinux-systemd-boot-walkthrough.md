---
layout: post
title: 'UKI + Secure Boot with sbctl on ArchLinux: systemd-boot walkthrough'
date: 2025-03-04 00:58 +0000
categories: ["archlinux", "systemd-boot", "secure-boot", "uki"]
tags: ["archlinux", "systemd-boot", "secure-boot", "uki", "sbctl"]
author: edu4rdshl
image:
  path: /secure-boot-linux.png
  alt: Image of a lock and a Linux logo. Image created with AI by the author.
excerpt: Let me guide you through the process of setting up UKI (Unified Kernel Image) with Secure Boot on ArchLinux using sbctl and systemd-boot.
---

# Introduction

This post is more of a reminder for me than anything else (so don't expect a very detailed, long post), but since that I've learned in the process, I decided to share it and try to help users who don't know anything about UKI and Secure Boot on Linux. This guide assumes that you use systemd-boot as your boot loader, that EFI is mounted on `/boot` and that `/boot` is FAT32. See the [ArchLinux wiki](https://wiki.archlinux.org/title/Systemd-boot) for more information.

**Tip**: If your `/boot` is FAT32, you can just uninstall any other bootloader, run `bootctl install` and you're good to continue with the guide.

### What is UKI?

UKI stands for [Unified Kernel Image](https://github.com/uapi-group/specifications/blob/main/specs/unified_kernel_image.md). As per the docs:

> A Unified Kernel Image (UKI) is a combination of an UEFI boot stub program, a Linux kernel image, an optional initrd, and further resources in a single UEFI PE file. This file can either be directly invoked by the UEFI firmware (which is useful in particular in some cloud/Confidential Computing environments) or through a boot loader (which is generally useful to allow multiple kernel versions with interactive or automatic selection of version to boot into).

TLDR: A UKI is a single file that contains all the necessary components to perform a boot. It contains the kernel, the initrd, modules, etc.

Benefits of UKI:

- **Secure Boot & Integrity**: A single signed binary simplifies Secure Boot verification.
- **Simplified Boot Process**: No need for a separate bootloader; UKI is directly launched by UEFI (optional).
- **Faster Boot Times**: Reduces disk I/O by loading the kernel and initrd together.
- **Improved TPM & Measured Boot**: Ensures all boot components are measured as a single unit for integrity verification.
- **Better Integration with systemd-boot**: Works seamlessly with systemd-boot for easy deployment and updates (apply to this post, you can use UKI with GRUB and other boot loaders too).
- **Easier Updates**: Eliminates kernel/initrd compatibility issues, making updates more reliable.
- **Reduces Attack Surface**: Removes complex bootloader scripting, lowering the risk of exploits.
- **Ideal for Immutable and embedded systems**: Perfect for these types of systems.

### What is systemd-boot?

It's a simple UEFI boot manager, like GRUB. It's part of the systemd project and is designed to be simple and efficient, plus, it's already installed on ArchLinux by default.

### What is Secure Boot?

Pretty sure you have heard about [Secure Boot](https://en.wikipedia.org/wiki/UEFI#Secure_Boot) (hello Windows users!). It's a feature of the UEFI firmware that helps to prevent unauthorized firmware, operating systems, or UEFI drivers from running at boot time. Basically, that's the only way to ensure that your boot process is secure and that no one has tampered with your system.

### Why UKI + Secure Boot?

To get Secure Boot working on Linux, you need to sign your kernel, initrd, boot loader, etc. UKI simplifies this process by combining all the necessary components into a single file, which makes it easier to sign and verify. Plus, systemd-boot has built-in support for UKI, making it a perfect match for Secure Boot.

# Requirements

For this post we will use:

- An ArchLinux system (or any Arch-based distro, but the commands may vary).
- mkinitcpio (for generating the initrd).
- A Secure Boot capable system (implies UEFI).
- A kernel that supports UKI (5.16+).
- A signed UKI (you can sign it yourself or use a pre-signed one).
- sbctl (for signing the UKI).
- systemd-ukify (for creating the UKI).

Those can be installed with:

```sh
sudo pacman -S --needed mkinitcpio sbctl systemd-ukify
```

# Setup

## Mkinitcpio setup

Please always refer to the [official documentation](https://wiki.archlinux.org/title/Unified_kernel_image#mkinitcpio) for the most up-to-date information.

1. Create `/etc/cmdline.d/` and add a file named `root.conf` with the root parameters that **you need**. Aditionally, **remove entries that point to microcode or initramfs**. In my case, I use a BTRFS root partition:

```sh
# The bgrt_disable parameter tells Linux to not display the OEM logo after loading the ACPI tables.
root=PARTUUID=5cedc40d-124d-4ecd-b0f5-fbe39545d0f7 rootflags=subvol=@ rw rootfstype=btrfs bgrt_disable quiet
```
These options are applied to every UKI image that you create, unless you specify a different configuration with the `--cmdline` option in the `default_options` variable in the kernel preset file.

2. Modify your kernel preset file (`/etc/mkinitcpio.d/*.preset`) to include the `uki` setup and remove others. Here's an example:

```sh
# mkinitcpio preset file for the 'linux' package

# ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux.img"
default_uki="/boot/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
```
### Generating the UKI

1. Run the following commands:

```sh
mkdir /boot/EFI/Linux
mkinitcpio -p linux
```
1. Check that the UKI was generated successfully:

```sh
$ tree /boot/EFI/Linux/
/boot/EFI/Linux/
└── arch-linux.efi
```
1. Set the newly generated UKI as the default boot option:

```sh
bootctl set-default arch-linux.efi
```

You are ready to boot into the UKI, remove `/boot/initramfs-*.img` files to avoid unnecessary files on your system.

## Secure Boot setup

**IMPORTANT**: Once you reach this point, reboot your computer and enter the UEFI settings. Look for the Secure Boot option and clear the keys/certs. **This is required to avoid issues with the sbctl setup**. Then, continue with the boot process as usual, with secure boot disabled - this will help you to test the UKI.

### sbctl setup

As usual, refer to the [official documentation](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Assisted_process_with_sbctl) for the most up-to-date information.

1. Create the signing keys:

```sh
sbctl create-keys
```
2. Enroll your keys with Microsoft's keys:

```sh
sbctl enroll-keys -m
```
3. Check the status of the sbctl setup:

```sh
sbctl status
```
sbctl should be installed now, but secure boot will not work until the boot files have been signed with the keys you just created and Secure Boot is enabled in the UEFI settings.

### Signing the UKI

To check which files need to be signed, run:

```sh
sbctl verify
```
sbctl comes with a pacman hook that automatically signs all new files whenever the Linux kernel, systemd or the boot loader is updated. To trigger it, just reinstall sbctl:

```sh
sudo pacman -S sbctl
```
Now, `sbctl verify` should show that the UKI is signed.

### Signing the boot loader

Setup a pacman hook in `/etc/pacman.d/hooks/95-systemd-boot.hook` to sign the boot loader:

```sh
[Trigger]
Type = Package
Operation = Install # Required for the first manual trigger that we are doing next, after that, you can remove this line. It isn't harmful anyway.
Operation = Upgrade
Target = systemd

[Action]
Description = Gracefully upgrading systemd-boot...
When = PostTransaction
Exec = /usr/bin/sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi && /usr/bin/systemctl restart systemd-boot-update.service
```
This hook will sign the boot loader whenever systemd is installed or upgraded. You can trigger it manually by reinstalling systemd:

```sh
sudo pacman -S systemd
```
Now, `sbctl verify` should show that the boot loader is also signed:

```sh
Verifying file database and EFI images in /boot...
✓ /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed is signed
✓ /boot/EFI/BOOT/BOOTX64.EFI is signed
✓ /boot/EFI/Linux/arch-linux.efi is signed
✓ /boot/EFI/systemd/systemd-bootx64.efi is signed
✗ /boot/vmlinuz-linux is not signed
```
Ignore `/boot/vmlinuz-linux is not signed`, we only need the `.efi` files signed.

At this point, you can enable Secure Boot in the UEFI settings and reboot. If everything is set up correctly, you should boot into the UKI with Secure Boot enabled, which can be verified by running:

```sh
$ bootctl status
System:
      Firmware: UEFI 2.70 (American Megatrends 5.17)
 Firmware Arch: x64
   Secure Boot: enabled (user)
  TPM2 Support: yes
  Measured UKI: yes
  Boot into FW: supported
```

## Especial considerations

This setup should continue to work even if you update the BIOS, as long as you don't clear the keys/certs, but you never know what OEMs can do. If you have any issues, you can always clear the keys/certs and re-enroll the keys.

# Conclusion

That's it! You have successfully set up UKI with Secure Boot on ArchLinux using systemd-boot. It may sound intimidating, but it's not that hard once you get the hang of it, and having a extra layer of security is always a good thing.
