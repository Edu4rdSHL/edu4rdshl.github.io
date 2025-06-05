---
layout: post
title: 'Smart suggestions on your Fish shell, without AI'
date: 2025-06-03 17:52 -0500
categories: ["fish", "shell", "plugins"]
tags: ["fish", "shell", "plugins", "based.fish"]
author: edu4rdshl
image:
  path: /based_fish.png
  alt: Based.fish logo, created with AI. It's a blue fish with a big brain.
excerpt: Based.fish is a Fish shell plugin that provides smart suggestions for commands based on your usage patterns, directory context, and command history. It uses SQLite for storage and fzf for selection, making it a powerful tool for enhancing your command line experience.
---

## Introduction

A few days ago I was looking for a way to improve my Fish shell experience, exactly I wanted to have smart suggestions based on my usage patterns, directory context, and command history. The reasons are simple: if I always use the same commands in the same directories, why not suggest them first? After looking for context-based suggestions, I found [Atuin](https://github.com/atuinsh/atuin), which is very well-known. It's a great tool, but doesn't provide what I wanted, it basically shows you the most used commands, ~~but not based on the current directory or the command you are typing~~ actually, it does provide per-directory completions, but it's very limited. Commands are always shown in the recent order, not per-directory repeats (unless you type something or set the "directory" filter, but then that limits the history to that directory only), and even with the filters, these filters works separately and you have to switch between them for history, they can't be combined and it's unlikely to happen https://github.com/atuinsh/atuin/issues/1611#issuecomment-1908451910, this is a dealbreaker **for me**. I discovered it trying to switch to Atuin.

So I decided to create my own plugin, which I called [based.fish](https://github.com/Edu4rdSHL/based.fish), because it's **very based**. A full explanation of its features and how it works can be found in the [README](https://github.com/Edu4rdSHL/based.fish/blob/main/README.md), but basically, it provides full context-based autocompletions for commands, using data such as the frequency of use, date of use, and the context of the current command line such as the path where you are, the command you are typing, etc.

## Features

- Context-aware autocompletion for commands, options, and arguments.
- Integration with fzf for a better selection experience.
- Easy installation and setup with Fisher.
- Support for importing existing Fish history.
- Customizable configuration options to tailor the behavior of the plugin.
- Directory-aware completions that take into account the current working directory.
- Statistics about command usage, such as most frequently used commands, options, and arguments.
- Customizable keybindings for navigating and selecting completions.
- Support for disabling fuzzy matching and confirmation prompts.

## How it works

based.fish uses a SQLite database to store command history and statistics. It analyzes the command history to provide context-aware suggestions based on the current command line input or simply based on the current working directory. The logic is as follows:

- It tracks the frequency of commands, options, and arguments used in the past.
- It tracks the working directory where the commands were executed.
- Smart suggestions are created with that information.
- fzf is used for fuzzy matching and selection of completions.
- It provides a user-friendly interface for navigating and selecting completions.

### Suggestions algorithm

- It considers the date of the last use, the frequency of use, and the context of the current command line input. Suggestions are made based on this information.
- If you are in a directory where you have previously used a command multiple times, it will suggest that command first, then the second most used command, and so on.
- For convenience, during the current session, it will always suggest your previous command first and then append the most used commands in the current directory, and then then global ones. It's to replicate the "normal" behavior of most shells.

## Installation

See the [installation section](https://github.com/Edu4rdSHL/based.fish?tab=readme-ov-file#installation) in the README, but basically, you can install based.fish using [Fisher](https://github.com/jorgebucaran/fisher) (more plugin managers can be supported in the future). Run the following command:

```fish
$ fisher install Edu4rdSHL/based.fish
```

## Usage and configuration

See the [usage](https://github.com/Edu4rdSHL/based.fish?tab=readme-ov-file#usage) and [configuration](https://github.com/Edu4rdSHL/based.fish?tab=readme-ov-file#configuration) sections in the README.

## Demo

![Demonstration of based.fish](../assets/gifs/based_fish_plugin.gif)

## Conclusion

based.fish is a powerful and lightweight Fish shell plugin that provides true smart suggestions for commands based on several metrics. Try it, it's free!

Feel free to contribute to the project, report issues, or suggest improvements. You can find the source code on [GitHub](https://github.com/Edu4rdSHL/based.fish).

Happy Fishing! 🐟