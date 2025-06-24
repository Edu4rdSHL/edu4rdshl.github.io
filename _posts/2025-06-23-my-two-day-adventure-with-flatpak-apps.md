---
layout: post
title: 'My two-day adventure with Flatpak apps'
date: 2025-06-23 18:13 -0500
categories: ["linux", "Flatpak", "gnome"]
tags: ["flatpak", "linux", "gnome", "apps"]
author: edu4rdshl
image:
  path: /flatpak.png
  alt: Flatpak logo
excerpt: I tried to replace my native apps with Flatpak apps on my main computer. Here's how it went.
---

## Introduction

The past week I was thinking about making my main computer more portable, secure, with no untracked config files thrown around in `$HOME`, and without having to learn some ugly functional programming language to achieve that. I thought about using [Flatpak](https://flatpak.org/) apps, which basically provided everything I wanted, so I decided to give it a try. I was motivated to see if I could replace all my native apps with Flatpak apps on my main system; I took a snapshot of my system, and started the experiment.

**Note**: My current system setup is ArchLinux + Gnome + Wayland, probably one of the best setups for Flatpak apps.

## The migration process

One of the cool things about Flatpak is that it has a large repository of apps available. I was able to find Flatpak versions of all the apps I use daily, including:

- Chrome
- Discord
- Spotify
- Telegram
- Obsidian
- Slack
- Signal
- Gajim
- Visual Studio Code
- GIMP
- OBS Studio
- and many more.

So, technically, the migration process for each app was as simple as:

- Uninstall the native app.
- Install the Flatpak version of the app.
- Copy the configuration and data files from the native app (usually in `$HOME/.config/$app`) to the Flatpak app directory (usually in `$HOME/.var/app/$app/config/$app`).
- Configure environment variables like `ELECTRON_OZONE_PLATFORM_HINT=auto` for Electron apps or `QT_QPA_PLATFORM=wayland` for Qt apps to ensure they work natively with Wayland. **Tip:** variables can be set globally using the flatpak command line or using Flatseal to avoid having to set them for each app.

And it worked! Well, mostly...

### The issues I faced

I have to start by saying that the migration process was really smooth; kudos to the Flatpak team for that. However, I did face some issues:

- **Broken mimetype associations**: every time I modified a `.desktop` file to add my custom flags, which means copying it to `$HOME/.local/share/applications`, the mimetype associations were lost, Chrome would not open any links, couldn't be set as default browser, and so on. To solve it, I had to manually run `update-desktop-database ~/.local/share/applications/` after modifying each `.desktop` file. I automated it with a systemd `.path` unit that ran the command when something changed in the directory, and that did the trick. I don't need to do this with native apps, so it was a bit annoying to have to do it with Flatpak apps.
- **Icons missing from Progressive Web Apps**: I use many PWAs, and it's important for me that the icons are set correctly, so I can see them in the dock or while Alt-Tabing. The Flatpak version of Chrome doesn't set the icons for PWAs correctly, the issue is known [1](https://discussion.fedoraproject.org/t/chrome-pwas-in-gnome-grouping-into-the-same-browser-icon/81714), [2](https://github.com/flathub/com.google.Chrome/issues/205), and [3](https://github.com/flathub/org.chromium.Chromium/issues/216). It's caused by an unmatching `StartupWMClass`, but there seems to be little interest in fixing it, so I had to manually modify the `.desktop` files for each PWA to set the correct `StartupWMClass`.
- **Poor permission's system**: Flatpak's permissions system is a bit too obscure for my taste; sometimes you install an app, and it works, but then you realize that it can't use certain features because it doesn't have the required permissions, and the only way to figure it out is going to the developer documentation, to find that you need [some override](https://docs.usebottles.com/flatpak/cant-enable-steam-proton-manager) to make it work. That's simply not user-friendly, even for veteran Linux users. I hope that we had some Android-like permissions system, where apps can request permissions at runtime for specific features.
- **Local updates interfere with remote updates**: on my current system (ArchLinux), when an update for an app is taking too long, I can just modify the `PKGBUILD`, change the version number, build and install it, and call it a day. If the same app is updated to a higher version, then it will be updated the next time I ran an system update. With Flatpak, I can't do that, I have to wait for the Flathub maintainers to update the app, or build it myself, which is not a big deal, but then I have to remember to update it manually or reinstall it again when updated on the remote, as the Flathub updates will not apply to my custom build.
- **Unable to set custom options for my graphic drivers**: Flatpak uses its own version of the graphics stack, which means that options applied to my native graphics driver are not applied to flatpak apps. It's a big deal for me, Nvidia does have [an issue](https://github.com/NVIDIA/egl-wayland/issues/126) with VRAM allocation, which is fixed by adding a custom profile to `/etc/nvidia/nvidia-application-profiles-rc.d/` to change the `GLVidHeapReuseRatio` parameter, but this couldn't be made to work with Flatpak apps, as confirmed by the devs. To give you an idea, my current VRAM usage with native apps is around 1.5GB, while with Flatpak apps it goes up to 3.5GB and counting, which is a big difference for my use case. This was the main reason I had to stop the experiment, as I will run out of VRAM for my games and other apps at some point. I know that it's not a flatpak issue per se, and what are the motivations behind it, but it was a dealbreaker for me.

These issues were not showstoppers, but they were annoying enough to **make me** reconsider the use of Flatpak apps as my main apps.

### The good things

Despite the issues I faced, there are **many** good things about using Flatpak apps:

- **Self-contained**: Flatpak apps are isolated from the host system, which means they don't interfere with other applications or the system itself, while providing a clean dependency tree for each app, so, no dependency nightmares.
- **Sandboxed**: Flatpak apps run in a sandboxed environment, which enhances security by limiting their access to the host system. This means that even if an app is compromised, it cannot easily affect the rest of the system. Of course, if proper permissions are granted.
- **Portable**: Flatpak apps can be easily moved between systems, making them ideal for portable setups. Just rsync the Flatpak directories, and you can run your apps on any compatible system, using the same configuration and data.
- **Easy installation**: Flatpak apps can be installed and managed using a simple command-line interface (Flatpak) or graphical tools (like Gnome Software), making them accessible to users of all skill levels.
- **Consistent**: Flatpak apps provide a consistent user experience across different distributions and versions of Linux, which means users and developers can expect the same behavior and functionality regardless of the underlying system.
- **Theming**: the apps respect the system theme, which is a big plus for me, as I like to have a consistent look and feel across all my apps.
- **Wayland support**: Most Flatpak apps work very well with Wayland, which is great performance and security-wise.
- **Great audio and video support**: Flatpak apps have really good support for audio and video, sometimes even better than native apps (Slack).

And many more...

## Conclusion

In conclusion, my two-day adventure with Flatpak apps was a really cool experience, mostly marred by personal needs and the tight control I have over my system. For anyone wondering: **I encourage you to try Flatpak apps**, it's a great technology that provides many benefits for everyone, especially for new users. However, I will not be using Flatpak apps as my main apps, at least not for now, until the issues I faced are resolved, or I find a way to work around them.

Happy sandboxing!
