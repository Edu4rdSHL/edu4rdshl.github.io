---
layout: post
title: 'New Year, New sudo: using systemd''s run0 as a sudo replacement'
date: 2026-01-01 14:04 -0500
categories: ["systemd", "linux", "sudo", "security", "run0"]
tags: ["systemd", "linux", "sudo", "security", "run0"]
author: edu4rdshl
image:
  path: /2026-01-01/systemd-run0.jpg
  alt: The systemd logo with the run0 word
excerpt: systemd 256 introduced run0, a new tool that can be used as a sudo replacement. In this post, I'll explain how to use it and why it's a great replacement to sudo.
---

## Introduction

systemd 256 [introduced](https://github.com/systemd/systemd/commit/7aed43437175623e0f3ae8b071bbc500c13ce893) `run0`. It isn't a new tool per se, but rather a symbolic link to `systemd-run` that changes its behavior to run commands as root, similar to `sudo`. Some of the motivations to replace `sudo` with `run0` are:

- `run0` doesn't rely on the [setuid](https://en.wikipedia.org/wiki/Setuid) bit.
- `run0` uses [Polkit](https://github.com/polkit-org/polkit) for authorization and privilege management.
- `run0` can be used to run commands in a more controlled environment, with better isolation and resource management.
- `run0` automatically integrates with D-Bus when running as a different user, making it easier to run commands that require D-Bus access (No more "Failed to connect to bus: No such file or directory" messages!).
- `run0` can be used to run commands in transient systemd services, which can be useful for running long-running commands or services.
- `run0` uses systemd's logging capabilities natively.
- By default, `run0` has a **very cool** red background color in the terminal to indicate that you are running a command as root! (take that, sudo!) /s
... and others.

I want to make a special mention to [Luca Boccassi (bluca)](https://github.com/bluca) who has been involved on many improvements to `run0` and [Polkit](https://github.com/polkit-org/polkit), big kudos to him for that!

## Configuration

To use `run0`, you need to have systemd 256 or later installed on your system. After that, you may want to add some [Polkit](https://github.com/polkit-org/polkit) rules to make the experience smoother, similar to what you would do for `sudo`.

### Password-based authentication with password caching

Create a file named `/etc/polkit-1/rules.d/90-run0-password-caching.rules` with the following content:

```javascript
/* Allow password-based authentication for run0 with caching like sudo */
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units") {
        return polkit.Result.AUTH_ADMIN_KEEP;
    }
});
```

**Notes:**

- This requires polkit 127 or later to work. See the [commit](https://github.com/polkit-org/polkit/commit/7d44d62a02f090a5a71c4ca10d58c7c2b54881eb) that introduced this feature.
- The default timeout for password caching is 5 minutes. You can change it by setting the `ExpirationSeconds` option in the `[Polkitd]` section of `polkitd.conf`. See the [polkitd.conf](https://man.archlinux.org/man/polkitd.conf.5.en) manpage and the [pull request](https://github.com/polkit-org/polkit/pull/604) that introduced this option for more details. Requires polkit 127 or later.

### Allow users in the wheel group to use run0 without a password

Create a file named `/etc/polkit-1/rules.d/90-run0-wheel-nopasswd.rules` with the following content:

```javascript
/* Allow members of the wheel group to use run0 without a password */
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units" &&
        subject.isInGroup("wheel")) {
        return polkit.Result.YES;
    }
});
```

### Allow specific users to use run0 without a password

Create a file named `/etc/polkit-1/rules.d/90-run0-specific-users-nopasswd.rules` with the following content, replacing `username1` and `username2` with the actual usernames:

```javascript
/* Allow specific users to use run0 without a password */
polkit.addRule(function(action, subject) {
    var allowedUsers = ["username1", "username2"];
    if (action.id == "org.freedesktop.systemd1.manage-units" &&
        allowedUsers.indexOf(subject.user) >= 0) {
        return polkit.Result.YES;
    }
});
```

### Aliasing run0

To make it easier to use `run0`, you can create an alias in your shell configuration file (e.g., `.bashrc`, `.zshrc`, etc.):

```sh
alias sudo='run0'
```

This way, you can use `sudo` as you normally would, but it will actually use `run0` under the hood.

## Usage

Using `run0` is very similar to using `sudo`. Here are some examples:

```sh
# Open a root shell
run0
# Run a command as root
run0 pacman -Syu
# Run a command as another user (e.g., 'postgres')
run0 -u postgres psql
# Run a command in a transient systemd service
run0 --unit=my-service --description="My Service" sleep 3600 # You can systemctl status my-service.service
# Run a command with a specific amount of memory, CPU, and other resource limits
run0 --property=MemoryMax=500M --property=CPUQuota=50% my-command # Note that run0 inherits resource limits from the parent slice, if you want more control, use systemd-run directly.
```

## Conclusion

`run0` is a great replacement for `sudo`, especially for those who want to avoid the security risks associated with setuid binaries. It leverages systemd and [Polkit](https://github.com/polkit-org/polkit) to provide a more secure and flexible way to run commands as root or other users. If you are a systemd lover like me, I highly recommend giving `run0` a try in your workflow!
