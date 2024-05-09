---
layout: post
title: Keeping your Rust toolchain and binaries up-to-date, the easy way
date: 2024-05-09 04:18 -0500
categories: ['rust', 'linux', 'cli']
tags: ['rust', 'linux', 'cli']
author: edu4rdshl
image:
  path: /rust-update.png
  alt: Rust crab with a cable connector. Image from internet.
excerpt: It's important to keep your Rust toolchain and binaries up-to-date. Here's how to do it the easy way.
---

## Introduction

Rust is a programming language that has been gaining a lot of popularity in the last few years, and it's not a surprise, it's a very powerful language, with a lot of features that make it very attractive for developers, like zero-cost abstractions, move semantics, guaranteed memory safety, threads, and more. As usual, it's important to keep your tools up-to-date, and Rust is no exception. In this post, I'll show you how to keep your Rust toolchain and binaries up-to-date, in a very easy way.

## Updating the Rust toolchain

When you install Rust, you get a tool called `rustup`, which is a command-line tool that manages Rust versions and associated tools. With `rustup`, you can install, update, and switch between different versions of Rust, and it's the recommended way to install Rust for development purposes. If you don't have `rustup` installed, you can install it by following the instructions on the [official website](https://rustup.rs/).

To update the Rust toolchain(s) that you have installed, run the following command:

```bash
$ rustup update && rustup self update
```
The first part of the command (`rustup update`) updates the Rust toolchain, and the second part (`rustup self update`) updates `rustup` itself.

## What about additional tools?

It's very common that you ends up installing additional tools that are part of the Rust ecosystem depending on the projects you are working on. Some common tools are `diesel_cli`, `cargo-edit`, `cargo-watch`, `cargo-audit`, and more. To make your life easier, you can install a tool called [cargo-update](https://github.com/nabijaczleweli/cargo-update), which is a cargo subcommand for checking and applying updates to installed executables. To install `cargo-update`, run the following command:

```bash
$ cargo install cargo-update
```
and then you can update all your installed tools by running:

```bash
$ cargo install-update -ag
```

## Putting it all together

As you can see, it's now a lot easier to keep everything Rust-related up-to-date. However, running it manually every time is not something you want to do, do you? Let's create a simple systemd service and timer to run the updates automatically. Here's how you can do it:

1. Create the service file:

```bash
$ systemctl --user edit --force --full rust-update.service
```
and put the following content:

```ini
[Unit]
Description=Check Rust updates.

[Service]
Type=oneshot
WorkingDirectory=/home/edu4rdshl
ExecStart=/usr/bin/bash -c "rustup update"
ExecStart=/usr/bin/bash -c "rustup self update"
ExecStart=/usr/bin/bash -c "cargo install-update -ag |& grep -q 'error while loading shared libraries' && cargo install cargo-update --force && cargo install-update -ag || true"

[Install]
WantedBy=default.target
```
**Note:** The last `ExecStart` line is a bit tricky, it checks if there's an error while loading shared libraries -hello ArchLinux users-, and if so, it force-installs `cargo-update` so that it gets rebuild and runs the updates again. This is basically a self-healing mechanism.

2. Create the timer file:

```bash
$ systemctl --user edit --force --full rust-update.timer
```
and put the following content:

```ini
[Unit]
Description=Check for rust updates every day.

[Timer]
OnCalendar=*-*-* 03:00
Persistent=true

[Install]
WantedBy=timers.target
```
3. Enable and start the timer:

```bash
$ systemctl --user enable --now rust-update.timer
```
from now on, your Rust toolchain and binaries will be updated automatically every day at 3:00 AM.

## Conclusion

Keeping your Rust toolchain and binaries up-to-date is important, and with the tools and techniques I've shown you in this post, you can do it in a very easy way. If you have any questions, feel free to reach me via any of the accounts in the left sidebar.

Happy coding! 🦀