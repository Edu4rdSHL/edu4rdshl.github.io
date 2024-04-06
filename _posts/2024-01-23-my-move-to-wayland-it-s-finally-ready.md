---
layout: post
title: 'My move to wayland: it''s finally ready'
date: 2024-01-23 17:57 -0500
categories: ['linux', 'wayland', 'xorg', 'gnome', 'archlinux']
tags: ['linux', 'wayland', 'xorg', 'gnome', 'archlinux']
author: edu4rdshl
image:
  path: /wayland-gnome.png
  alt: Wayland and Gnome. Image by itsfoss.com
excerpt: I was a Wayland detractor, had very bad experiences with it in the past, but I decided to give it a try again, and it's time to say goodbye to Xorg.
---

## My past experiences with Wayland

I tried Wayland for the first time in 2016, I was using ArchLinux, tried to create a Wayland setup using Sway on my Intel laptop. The remnants of that setup, can still be found in my [GitHub repo](https://github.com/Edu4rdSHL/linuxscripts/tree/8acc4a6bff033db7291d701169beb8d5c278f3eb/user-config).

I had a lot of problems with it, a LOT of applications didn't work, not natively, not through XWayland, and the few applications that worked, had a lot of bugs, like closing unexpectedly, stop responding to mouse events, sporadic freezes, etc. I use my PC for work, therefore I can't afford to have a buggy system, so I decided to go back to Xorg.
 
## Why I decided to give it a try again?

My setup was working perfectly with Xorg, using Xfce as my desktop environment, but the news about Fedora and other distributions going Wayland by default, Gnome 45 and its many Wayland improvements, Nvidia drivers supporting GBM, and the fact that I was curious about the advances of Wayland, made me want to give it a try again.

## My current setup

I'm not using my laptop anymore, currently I use a desktop PC with an AMD Ryzen 9 5950X, 2 NVMe of 1TB each one, 64GB of RAM, and an Nvidia RTX 3090 Ti (one of the main reasons of the curiosity to try Wayland). The software installed is ArchLinux with Gnome (45.3), BTRFS, the Nvidia drivers from the official repositories (545.29.06 at this date), and, Xwayland (23.2.4).

## My current experience with Wayland

I switched from Xfce (Xorg) to Gnome (Wayland) about 2 months ago, and I have to say that I'm very happy with it, I haven't had any problems with it after [tweaking some configs](#some-settings-needs-to-be-updated), I'm using it with the Nvidia drivers, and I haven't had any problems with it either, Xwayland works very well too, I like to play games on my PC using Steam and Lutris and everything has worked perfectly there as well.

I only have noticed some bugs on the Steam client, which creates some artifacts in the application while scrolling, but it's not a big deal.

## The daily software I use

I use my PC for work, so I use a lot of software, which includes a lot of the infamous Electron apps: VSCode, Slack, Discord, Spotify, Element Desktop, Obsidian, Signal Desktop, Termius, Insomnia, Tidal HiFi, and many others, as well as Google Chrome, WeeChat, Dino (XMPP client), and a lot of other software. Again, I haven't had any problems with the mentioned software after [tweaking some configs](#some-settings-needs-to-be-updated).

As usual, there are some alternatives that should "just work" with Wayland, but the amount of configurations required to get the previous software working is very small, plus, the fact that I'm already used to that software for years, makes me want to keep using them and I'm very happy with the results.

## Some settings needs to be updated

Xorg and Wayland are totally different, so, it's expected that some settings need to be updated, here are some of the settings I had to update to get my setup working properly:

### Screen Sharing

Many users have noticed that when running a Xwayland application inside a Wayland session, the screen sharing feature doesn't work as expected. It's because Wayland security model doesn't allow X clients to access the content of Wayland applications, so, the screen sharing feature will only show the applications running under X. To solve it, all you need to do is install [xwaylandvideobridge](https://invent.kde.org/system/xwaylandvideobridge) and it will do the job.

### Apps not respecting the system's dark mode

Some apps don't respect or just ignore the system's dark mode (i.e xfce4-terminal, Element Desktop, and others), to fix it you need to:

```bash
# Install the gnome-themes-extra package
pacman -S gnome-themes-extra
```

```bash
# Set the following setting using gsettings
gsettings set org.gnome.desktop.interface gtk-theme "Adwaita-dark"
```

### Electron apps

All you need is a `.conf` file in your `$XDG_CONFIG_HOME` folder (usually `$HOME/.config`), or whatever the application read it (if capable) with the following content:

```bash
--enable-features=UseOzonePlatform,WaylandWindowDecorations
--ozone-platform=wayland
```

There are applications that doesn't read the `.conf` file, so you need to update the `.desktop` file, to avoid changes being overwritten, you can copy the `.desktop` file to your `$XDG_DATA_HOME/applications` (usually `$HOME/.local/share/applications`) folder, and update the `Exec` line with the following content:

```bash
Exec=/usr/bin/executable --enable-features=UseOzonePlatform,WaylandWindowDecorations --ozone-platform=wayland
```

### Google Chrome

[Google Chrome Beta](https://aur.archlinux.org/packages/google-chrome-beta) reads the configuration [from](https://aur.archlinux.org/cgit/aur.git/tree/google-chrome-beta.sh?h=google-chrome-beta) `$HOME/.config/chrome-beta-flags.conf`, so you can create a file with the following content:

```bash
--ozone-platform-hint=auto
--enable-features=NativeNotifications,UseOzonePlatform
```

and it will configure the application to use Wayland.

### Steam

At this point, Steam doesn't support Wayland and you will have rendering issues when scrolling the webviews inside the app, unless you turn off hardware acceleration for the webviews rendering and then it will work fine. To do that, go to `Steam > Settings > Interface > Turn off "Enable GPU accelerated rendering in webviews"`.

### Visual Studio Code

I use the [visual-studio-code-insiders-bin](https://aur.archlinux.org/packages/visual-studio-code-insiders-bin) package, and it reads the configuration from `$HOME/.config/code-flags.conf`, so you can add the same content as for the Electron apps. I decided to add this app here separately because as far as I know, the `code` app from the official repositories doesn't work with Wayland.

### Nvidia drivers

Please read the [ArchWiki](https://wiki.archlinux.org/title/wayland#NVIDIA_driver) for more information about this. Basically, enable DRM KMS and you're good to go. For Xwayland apps, you want to enable `kms-modifiers` on Gnome, that's as easy as running the following command:

```bash
gsettings set org.gnome.mutter experimental-features [\"kms-modifiers\"]
```

### GTK

GTK 3 and 4 have Wayland enabled by default, but you can force it for another versions by setting the following environment variable (probably in your `profile.d`):

```bash
export GDK_BACKEND=wayland
```

Check [this](https://wiki.archlinux.org/title/GTK#Wayland_backend) if you're having issues with themes.

### QT

QT 5 and 6 have Wayland enabled by default only if the `qt5-wayland` and `qt6-wayland` packages are installed, for another versions, set the following environment variable (probably in your `profile.d`):

```bash
export QT_QPA_PLATFORM=wayland
```

## Additional tweaks

### Extensions

I use the following extensions:

- Workspace Indicator (built-in).
- [Clipboard Indicator](https://extensions.gnome.org/extension/779/clipboard-indicator/): A clipboard manager which history and multimedia support.
- [Just Perfection](https://extensions.gnome.org/extension/3843/just-perfection/): A collection of tweaks for the Gnome Shell. I personally recommend enabling the "Window Demands Attention Focus" under the "Behavior" section.

### Additional packages

- [nautilus-open-any-terminal](https://aur.archlinux.org/packages/nautilus-open-any-terminal/): Context-menu entry for opening other terminal in Nautilus. Note: I use Xfce terminal because Console/Gnome Terminal does have a focus issue when you open a link from the terminal and go back to it. You can use it to have the "Open in terminal" option in Nautilus for any terminal you want.
- [vesktop](https://aur.archlinux.org/packages/vesktop/): A standalone Electron app that loads Discord & Vencord. Supports streaming audio via Discord's screen share feature, something that the official Discord app [doesn't support yet](https://support.discord.com/hc/en-us/community/posts/360050971374-Linux-Screen-Share-Sound-Support?page=2).

### Some tweaks/tips

- Disable the "Activities" hot corner if you don't like it.
- Set a fixed number of workspaces. That way, you can have pre-defined apps for each workspace number.
- Change the Alt+Tab behavior to "Switch Windows" instead of the default "Switch Applications". It will make the Alt+Tab behavior similar to any other desktop environment.
- If an app supports Wayland, please use it, it will improve your experience by a lot. Running apps through Xwayland is not bad, but it's not the same as running them natively and you'll notice it.
- You can use [xorg-xeyes](https://www.archlinux.org/packages/extra/x86_64/xorg-xeyes/) to check if an app is running through Xwayland or not, just run `xeyes` in your terminal and move the mouse on the app's surface. If you see the eyes moving, it's running through Xwayland, if you don't, it's running natively.

## Conclusion

I'm very happy with my current setup, I haven't had any problems with it, and I'm very happy with the performance of Wayland, I'm not going back to Xorg, and I'm looking forward to the future of Wayland.