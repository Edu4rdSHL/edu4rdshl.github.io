---
layout: post
title: 'Introducing Rusnapshot: Handy Btrfs snapshot tool in Rust'
date: 2024-02-06 23:27 -0500
categories: ['btrfs', 'rust', 'filesystem']
tags: ['btrfs', 'rust', 'filesystem']
author: edu4rdshl
image:
  path: /rusnapshot.jpeg
  alt: Rusnapshot logo, generated by Google Bard
---

Rusnapshot is a handy tool to manage Btrfs snapshots, written in Rust.

## Motivation

I've been using Btrfs for a long time, and of course I use snapshots a lot, but I always felt that the tools available for managing snapshots were not very flexible, some of the limitations I found were: hardcoded snapshot's path, allow to take snapshots of certain subvolumes only, not centralized metadata for easily managing snapshots across multiple computers, etc. So, I decided to create a tool that would make my life easier, and that's how Rusnapshot was born.

## What is Rusnapshot?

Rusnapshot is a tool to manage Btrfs snapshots, it allows you to take snapshots of subvolumes, list snapshots, delete snapshots, and restore snapshots. It also has support for configuration files that allows you to define the snapshot's path, the subvolumes to snapshot, the number of snapshots to keep, and more.

## Features

- Allows you to specify the origin and destination of snapshots at will of the user.
- Track snapshots using SQLite as backend database.
- Easy setup using [small templates](https://github.com/Edu4rdSHL/rusnapshot/tree/master/examples/config-templates) instead of confusing long files.
- Ability to create read-only or read-write snapshots.
- Ability to use the same SQLite database for everything.
- Ability to specify the prefix of the name for the snapshots for better identification.
- Ability to specify a `kind` identifier. Useful if you plan to have hourly, weekly, monthly or more `kind` of snapshots of the same subvolume(s).
- Ability to specify the maximum number of snapshots to keep for automatic cleanup.
- Supports restoration of snapshots in the original directory or a specific one.
- Supports machine name identification for better tracking when sharing the same metadata database in multiple machines.
- User-friendly CLI output to see the status and details of snapshots.

## Installation

If you're using Arch Linux, you can install Rusnapshot from [the AUR](https://aur.archlinux.org/packages/rusnapshot-git/).

otherwise, you can build it from source using the following commands:

```bash
$ git clone https://github.com/Edu4rdSHL/rusnapshot.git
$ cd rusnapshot
$ cargo build --release
$ sudo cp target/release/rusnapshot /usr/local/bin/
```

## Usage

See the [Setup and usage](https://github.com/Edu4rdSHL/rusnapshot/blob/master/docs/SETUP_AND_USAGE.md) guide for more information and the [README](https://github.com/Edu4rdSHL/rusnapshot/blob/master/README.md) file for more details.

## What about using Rusnapshot on multiple machines?

Rusnapshot has support for using the same SQLite database across multiple machines, you just need a shared directory where the SQLite database will be stored, and then you can use the `-d/--dfile` option to specify the path to the SQLite database.

## Conclusion

Rusnapshot is a handy tool to manage Btrfs snapshots, it's written in Rust, and it's easy to use. I hope you find it useful, and if you have any questions or suggestions, feel free to open an issue on the [GitHub repository](https://github.com/Edu4rdSHL/rusnapshot/issues).