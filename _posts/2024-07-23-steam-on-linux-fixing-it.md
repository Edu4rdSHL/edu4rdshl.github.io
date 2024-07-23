---
layout: post
title: 'Steam on Linux: fixing it!'
date: 2024-07-23 08:23 -0500
categories: ['linux', 'gaming', 'steam']
tags: ['linux', 'gaming', 'steam']
author: edu4rdshl
image:
    path: /gaming-on-linux.jpeg
    alt: Gaming on Linux. Image created with Microsoft Designer.
excerpt: Steam is a popular gaming platform, but sometimes it doesn't work as expected on Linux. Here's how to fix it.
---

## Introduction

Steam is the favorite gaming platform on Linux, and it's not a surprise: it's the only platform with a native Linux client, has put a lot of efforts on Proton/Wine so that Windows games run without issues on Linux, and has a lot of games available. However, sometimes it doesn't work as expected, small things like slow download speeds, shaders compilation taking forever, or some artifacts when navigating the store. In this post, I'll show you how to fix some of these issues.

## Most common issues

### Slow download speeds

If you are experiencing slow download speeds on Steam, you can try it:

- **Change the download server**: Steam has a lot of servers around the world, and sometimes the one you are using is overloaded. To change the download server, go to `Steam > Settings > Downloads > Download Region` and select a different region.
- **Disable HTTP2**: HTTP2 should improve download speeds, except that [sometimes it doesn't](https://github.com/ValveSoftware/steam-for-linux/issues/10248). To disable HTTP2, create a file called `$HOME/.steam/steam/steam_dev.cfg` with the following content:

```ini
@nClientDownloadEnableHTTP2PlatformLinux 0
```
- **Increase the number of maximum initial connections**: By default, Steam starts connected to only 1 server, and then it increases the number of connections as needed. You can increase the number of maximum initial connections by modifying the `$HOME/.steam/steam/steam_dev.cfg` file, appending the following content:

```ini
@cMaxInitialDownloadSources 15 # Note: you can change the number to whatever you want or test until you find the best value for you.
```

### Shaders compilation taking forever

If you are experiencing shaders compilation taking forever, you can try it:

- **Increase the number of threads**: By default, Steam uses only 1 thread to compile shaders, you can increase the number of threads by modifying the `$HOME/.steam/steam/steam_dev.cfg` file, adding the following content:

```ini
unShaderBackgroundProcessingThreads 6 # Note: make sure to NOT use more threads than your CPU has, and always use a lower number than the number of threads your CPU has. Otherwise, your system will start lagging during the process.
```

### Artifacts when navigating the store

If you are experiencing artifacts when navigating the store, you can try it:

- **Disable the Steam webpages hardware acceleration**: Steam uses a web browser to show the store, and sometimes the hardware acceleration can cause artifacts. To disable the hardware acceleration, go to `Steam > Settings > Interface` and disable the `Enable GPU accelerated rendering in web view` option.

## Conclusion

Happy gaming! 🎮