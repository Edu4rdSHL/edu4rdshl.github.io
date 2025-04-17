---
layout: post
title: 'Fixing Intel AX200/AX210 random driver crashes on Linux'
date: 2025-04-16 22:59 -0500
categories: ["linux", "intel", "wifi"]
tags: ["linux", "intel", "wifi", "ax200", "ax210"]
author: edu4rdshl
image:
  path: /intel-ax210-wifi-card.png
  alt: An image of an Intel AX210 WiFi card chip. Taken from the internet.
excerpt: The Intel AX200 and AX210 WiFi cards are AWESOME cards for Linux, but they can be annoying when they crash randomly. Let's fix that!
---

# Introduction

After fighting with bluetooth dongles, which have patetic support on Linux, I decided to buy an Intel AX210 WiFi card because I needed WiFi6 for doing some job-related tasks like measuring WiFi 6 performance of a firmware that I was developing.

I was amazed by the performance and the fact that it worked out of the box on Linux. However, I started to notice random crashes of the driver, causing hard freezes of the whole network stack, affecting also wired connections. The error log that I was getting, started with:

{% raw %}
```txt
Apr 15 00:06:47 Behemoth kernel: ------------[ cut here ]------------
Apr 15 00:06:47 Behemoth kernel: Timeout waiting for hardware access (CSR_GP_CNTRL 0xffffffff)
Apr 15 00:06:47 Behemoth kernel: WARNING: CPU: 5 PID: 383554 at drivers/net/wireless/intel/iwlwifi/pcie/trans.c:2460 __iwl_trans_pcie_grab_nic_access+0x13c/0x140 [iwlwifi]
Apr 15 00:06:47 Behemoth kernel: Modules linked in: hid_xpadneo(OE) ff_memless xfrm_interface xfrm6_tunnel tunnel4 tunnel6 xfrm_user xfrm_algo l2tp_ppp l2tp_netlink l2tp_core ip6_udp_tunnel udp_tunnel pppox uinp>
Apr 15 00:06:47 Behemoth kernel:  polyval_generic btintel iwlwifi ghash_clmulni_intel snd_hda_codec snd_ump eeepc_wmi btbcm sha512_ssse3 nvidia_uvm(OE) nvidia_modeset(OE) asus_wmi btmtk snd_rawmidi snd_hda_core >
Apr 15 00:06:47 Behemoth kernel: CPU: 5 UID: 0 PID: 383554 Comm: iwd Tainted: G         C OE      6.14.2-arch1-1 #1 51440b8a0cc8bb91764dac94f6c2b53455e5a907
Apr 15 00:06:47 Behemoth kernel: Tainted: [C]=CRAP, [O]=OOT_MODULE, [E]=UNSIGNED_MODULE
Apr 15 00:06:47 Behemoth kernel: Hardware name: ASUS System Product Name/PRIME B550-PLUS, BIOS 3621 01/13/2025
Apr 15 00:06:47 Behemoth kernel: RIP: 0010:__iwl_trans_pcie_grab_nic_access+0x13c/0x140 [iwlwifi]
Apr 15 00:06:47 Behemoth kernel: Code: 2e f2 31 c0 eb 88 be 02 00 00 00 48 89 df e8 4b fd ff ff eb e5 89 c6 48 c7 c7 d8 9d 22 c2 c6 05 3e 50 02 00 01 e8 74 4f 38 f1 <0f> 0b eb a4 90 90 90 90 90 90 90 90 90 90 90>
Apr 15 00:06:47 Behemoth kernel: RSP: 0018:ffffa850d9467150 EFLAGS: 00010286
Apr 15 00:06:47 Behemoth kernel: RAX: 0000000000000000 RBX: ffff904b26118028 RCX: 0000000000000027
Apr 15 00:06:47 Behemoth kernel: RDX: ffff9059ee4a1948 RSI: 0000000000000001 RDI: ffff9059ee4a1940
Apr 15 00:06:47 Behemoth kernel: RBP: 00000000ffffffff R08: 0000000000000000 R09: ffffa850d9466fd0
Apr 15 00:06:47 Behemoth kernel: R10: ffff9059ee1fffa8 R11: 0000000000000003 R12: ffff904b26119c1c
Apr 15 00:06:47 Behemoth kernel: R13: 0000000000000001 R14: 0000000000000011 R15: ffff904b2d992038
Apr 15 00:06:47 Behemoth kernel: FS:  00007137d6b77b80(0000) GS:ffff9059ee480000(0000) knlGS:0000000000000000
Apr 15 00:06:47 Behemoth kernel: CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
Apr 15 00:06:47 Behemoth kernel: CR2: 000021fc045e9000 CR3: 0000000505dae000 CR4: 0000000000f50ef0
Apr 15 00:06:47 Behemoth kernel: PKRU: 55555554
Apr 15 00:06:47 Behemoth kernel: Call Trace:
Apr 15 00:06:47 Behemoth kernel:  <TASK>
Apr 15 00:06:47 Behemoth kernel:  iwl_trans_pcie_grab_nic_access+0x1a/0x40 [iwlwifi e98919710b14e871ba2583c35186fbfcf23c0c5d]
Apr 15 00:06:47 Behemoth kernel:  iwl_write_prph_delay+0x1f/0x60 [iwlwifi e98919710b14e871ba2583c35186fbfcf23c0c5d]
Apr 15 00:06:47 Behemoth kernel:  iwl_pcie_conf_msix_hw+0x1a1/0x200 [iwlwifi e98919710b14e871ba2583c35186fbfcf23c0c5d]
Apr 15 00:06:47 Behemoth kernel:  iwl_trans_pcie_start_hw+0x1aa/0x2b0 [iwlwifi e98919710b14e871ba2583c35186fbfcf23c0c5d]
Apr 15 00:06:47 Behemoth kernel:  iwl_mvm_up+0x38/0xb10 [iwlmvm c68c30791f8a0b7ac01cb7ffe0d26e3cf2222dbf]
Apr 15 00:06:47 Behemoth kernel:  ? mas_topiary_replace+0xa8d/0xd90
Apr 15 00:06:47 Behemoth kernel:  __iwl_mvm_mac_start+0x78/0x2a0 [iwlmvm c68c30791f8a0b7ac01cb7ffe0d26e3cf2222dbf]
Apr 15 00:06:47 Behemoth kernel:  iwl_mvm_mac_start+0x4d/0xf0 [iwlmvm c68c30791f8a0b7ac01cb7ffe0d26e3cf2222dbf]
Apr 15 00:06:47 Behemoth kernel:  drv_start+0x3f/0x100 [mac80211 682e229732a6bfe53cb0bbcde81ec2801d27c374]
Apr 15 00:06:47 Behemoth kernel:  ieee80211_do_open+0x286/0x7e0 [mac80211 682e229732a6bfe53cb0bbcde81ec2801d27c374]
Apr 15 00:06:47 Behemoth kernel:  ieee80211_open+0x8c/0xa0 [mac80211 682e229732a6bfe53cb0bbcde81ec2801d27c374]
Apr 15 00:06:47 Behemoth kernel:  __dev_open+0xfc/0x1d0
Apr 15 00:06:47 Behemoth kernel:  __dev_change_flags+0x1e4/0x230
Apr 15 00:06:47 Behemoth kernel:  dev_change_flags+0x26/0x70
Apr 15 00:06:47 Behemoth kernel:  do_setlink.isra.0+0x2f7/0x1150
Apr 15 00:06:47 Behemoth kernel:  ? nla_put+0x2c/0x40
Apr 15 00:06:47 Behemoth kernel:  ? inet6_fill_ifla6_attrs+0x4d1/0x540
Apr 15 00:06:47 Behemoth kernel:  ? ep_poll_callback+0x2a0/0x2f0
Apr 15 00:06:47 Behemoth kernel:  ? __nla_validate_parse+0x5f/0xca0
Apr 15 00:06:47 Behemoth kernel:  ? __wake_up_sync_key+0x3b/0x60
...
```
{% endraw %}

The issue was easily reproducible by simply using the card for a while (during the tests), then stopping `iwd` for a few hours and then starting it again. The card would crash 100% of the time. If I leave it connected for a long time, it didn't crash, so I concluded that it was something related to power management.

# Troubleshooting

After checking the logs, I decided to do a quick search on the internet and quickly found a lot of people with the same issue. The most common recommendation was to disable power management on the card, which is a common fix for many WiFi issues on Linux. However, it didn't worked reliably for a lot of users, and didn't worked at all for me. The things that I tried were:

- Disabling power saving on the card by setting `power_save=0` on the iwlwifi module.
- Disabling ASPM on the whole PCIE by adding `pcie_port_pm=off pcie_aspm.policy=performance` to the kernel parameters.
- Properly configuring the wireless regulatory domain.
- Disabling `11n` modes on the card by setting `11n_disable=1` on the iwlwifi module.
- Disabling the card's power control using a udev rule.

But nothing worked. After messing with `/sys/bus/pci/devices/0000:05:00.0/` (the device path of the card), I found that the card's `power_state` was going to `D3cold` even after disabling every PCI power management option. `D3cold` is the lowest power state, which means that the card is completely powered off after some time of inactivity. This means that when a device goes into `D3cold`, it needs to be reinitialized to start working again, basically, supplying voltage to the device. This exists to save power and decrease temperature (which is negligible anyways). The problem is that the driver doesn't handle this state properly, causing the driver to crash when trying to access the device after it has been powered off.

When the card was working properly, the `power_state` was `D0`, which is the fully powered state.

# Solution

So, I rolled back all the kernel and module parameters and decided to add this simple udev rule to restrict the card from entering `D3cold`:

```bash
# /etc/udev/rules.d/99-disable-d3cold-ax210.rules
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x8086", ATTR{device}=="0x2725", ATTR{d3cold_allowed}="0"
```

and rebooted. It has been working flawlessly since then, even trying to reproduce the issue, and the only noticeable change is that the card's `power_state` is now `D0` all the time, which is what we want.

You can verify if the udev rule is working by checking the `d3cold_allowed` attribute:

```bash
cat /sys/bus/pci/devices/0000:05:00.0/d3cold_allowed
```

Make sure to change the PCIE device path to your card's path. If the output is `0`, then the udev rule is working.

# Conclusion

This was a very annoying issue that took me a while to figure out, so instead of disabling all the PCI power management for the whole system and affecting temperatures/power saving in general, we can make it work only affected the specific device.

Happy browsing!