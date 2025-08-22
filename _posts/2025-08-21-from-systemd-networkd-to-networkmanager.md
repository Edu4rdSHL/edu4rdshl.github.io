---
layout: post
title: 'From systemd-networkd to NetworkManager'
date: 2025-08-21 18:29 -0500
categories: ["networking", "systemd", "networkd", "networkmanager"]
tags: ["networking", "systemd", "networkd", "networkmanager"]
author: edu4rdshl
image:
  path: /networkmanager.jpg
  alt: A image of the NetworkManager logo
excerpt: After 7 years, I recently switched from systemd-networkd to NetworkManager for managing my network connections. VPNs were the main reason for the switch.
---

## Introduction

I have been a very pro-systemd user for many years, that can be easily verified on [some of my publications](https://infosec.exchange/@edu4rdshl/111701075297507646). My setup was based on systemd components for almost everything: networking, DNS, logging, out-of-memory management, booting my system, and more. I love systemd, there is no doubt about that, but things got complicated when having many VPNs.

## The problem

My VPN setup for L2TP/IPsec was: xl2tpd + strongSwan, which included manually editing files for it to be setup. Then I had a service like it to setup the routes:

{% raw %}
```bash
# /etc/systemd/system/corp-vpn.service
[Unit]
Description=Start Corp VPN
After=strongswan.service xl2tpd.service
Wants=strongswan.service xl2tpd.service
Requires=strongswan.service xl2tpd.service
PropagatesStopTo=strongswan.service xl2tpd.service

[Service]
Type=oneshot
# This was the script to setup the VPN routes
ExecStart=/usr/local/bin/corp-vpn
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
{% endraw %}

And the `corp-vpn` script itself looked like this:

```bash
#!/usr/bin/env bash
ROUTES=("route1/24" "route2/24" "route3/24" "route4/16")
GATEWAY_IP="x.x.x.x"
ipsec up myvpn
sleep 3 && echo 'c myvpn' > /var/run/xl2tpd/l2tp-control
# Give xl2tpd some time to establish the connection
sleep 5
for route in ${ROUTES[@]}; do
  ip route add "$route" via "$GATEWAY_IP" dev ppp0
done
```

Then I had to simply start the vpn service and everything was set up automatically. It worked fine for many time, where I had 2/3 VPNs to manage, but recently I needed to connect to more VPNs, a total of 7. Things got complicated quickly, and modifying every service and script became a hassle. As I couldn't easily change the route metric per VPN, because the VPN devices are randomly assigned at the moment of the connection, I decided to look for alternatives, which obviously led me to NetworkManager.

A special mention, is that my wireless setup was `iwd` + `systemd-networkd`, which required me to manually run `iwctl`, scan, see available networks, and connect to them. I do some wireless testing for work which requires me to frequently change networks and configurations, so this was not ideal as well.

## NetworkManager

It's almost the de-facto standard for managing network connections on Linux desktops. Almost every desktop environment or window manager integrates with it, directly like Gnome does, or indirectly through various applets and tools. I use [Gnome + Wayland as my DE](https://www.edu4rdshl.dev/posts/my-move-to-wayland-it-s-finally-ready/), so the integration with NetworkManager is seamless. Here are the things that I love from NetworkManager:

- **Ease of use**: The CLI/TUI and graphical interfaces make it easy to manage connections without diving into configuration files. This is because the rich D-Bus API allows for dynamic updates and changes.
- **VPN management**: NetworkManager has built-in support for various VPN technologies, and community plugins extend this support even further. It makes configuring and managing VPN connections much simpler.
- **Dynamic routing**: It can automatically adjust routes based on the type of active connections, simplifying the management of complex networking setups.
- **Sane defaults**: NetworkManager comes with sensible default configurations that work for most users, reducing the need for manual adjustments. For example, it does default to a route metric of 50 for VPNs and 100 for regular connections.
- **Wireless management**: Wireless management is greatly simplified with NetworkManager. It can handle scanning, connecting, and managing Wi-Fi networks without the need for manual intervention. You can choose between the `iwd` and `wpa_supplicant` backends, depending on your preference.
- **Integration with systemd**: NetworkManager integrates very well with systemd, allowing you to use systemd-resolved for DNS resolution. This aligns with my existing setup.

It took me a few minutes to migrate all my VPN connections to NetworkManager. No more manual routing (due to the dynamic route-metric handling), one-click activation, and everything just works. Do I want to add a new VPN? Just a few clicks in the GUI, and I'm done.

### Tips and tricks

- **Use the CLI**: NetworkManager's command-line interface (`nmcli`) is powerful and allows for scripting and automation. Familiarize yourself with it to make the most of your VPN connections. Understanding the options available in `nmcli` can help you fine-tune your setup.
- **Explore plugins**: NetworkManager supports various plugins for different VPN technologies. Explore these to enhance your VPN experience.
- **Use the dispatcher system**: NetworkManager's dispatcher system allows you to run scripts in response to network events. This can be useful for automating tasks when a connection is established or disconnected.
- **For systemd-resolved**: Make sure to not set global DNS servers in `/etc/systemd/resolved.conf{,.d}`, as this can interfere with NetworkManager's DNS management. The same applies for options such as `DNSOverTLS=`/`DNSSEC=`, if your current VPN DNS doesn't support them, you will end up with connectivity issues.
- **Set ipv4.route-metric/ipv4.dns-priority**: You can fine-tune the routing and DNS settings for each connection by adjusting the `ipv4.route-metric` and `ipv4.dns-priority` properties.
- **Check the documentation**: NetworkManager has extensive documentation available. If you encounter issues or need to configure advanced settings, refer to the official documentation for guidance. I recommend checking `man nm-settings` and the [NetworkManager Docs](https://networkmanager.dev/docs/).

## Conclusion

Migrating from systemd-networkd to NetworkManager has significantly improved my network management experience on my desktop (I will still use systemd-networkd for my server). The dynamic routing capabilities, ease of use, and seamless integration with my desktop environment have made it a worthwhile transition. Thank you, systemd-networkd for your service, but I think I will be sticking with NetworkManager for the foreseeable future.

Happy networking!