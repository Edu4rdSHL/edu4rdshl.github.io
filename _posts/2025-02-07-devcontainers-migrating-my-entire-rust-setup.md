---
layout: post
title: 'DevContainers: Migrating my entire Rust setup'
date: 2025-02-07 00:18 -0500
categories: ["rust", "devcontainers", "vscode"]
tags: ["rust", "devcontainers", "vscode"]
author: edu4rdshl
image:
  path: /devcontainers.jpeg
  alt: A image of Docker + VsCode. Image from internet.
excerpt: Migrating my entire Rust development setup to DevContainers was a great decision.
---

# Introduction

I've been using Rust for a long time, and I love it, we all know what it offers so lets not focus on it. I'm also a VSCode enjoyer, and recently I found about DevContainers on a release note of VSCode. I was curious about it, so I decided to investigate more, and it definitely caught my attention.

# What are DevContainers?

As per the [official documentation](https://containers.dev/): A development container (or dev container for short) allows you to use a container as a full-featured development environment. It can be used to run an application, to separate tools, libraries, or runtimes needed for working with a codebase, and to aid in continuous integration and testing.

If there's something that I loved about DevContainers, is that they do have [a specification](https://containers.dev/implementors/spec/). That's all. Having a specification means that you can define your development environment in a declarative way, and then you can share it with others, or use it on different projects. Basically, work one time, use it forever.

The idea is to have a container with all the tools you need to work on a project, and everyone using it will have the same environment, no matter the OS they are using. That's great for teams, but also for personal projects, as you can have a clean environment for each project you are working on, or a shared one that doesn't mess with your bare system.

# Why I decided to migrate?

I have a lot of projects in Rust, and I ended up with a lot of tools installed on my system using `cargo` (which means that they aren't tracked by the system's package manager), that's not something I like. There are also projects that require a specific version of Rust, adding another layer of complexity to the setup. All of this, on a non-declarative way, which means that I can't just clone a project and start working on it, I need to remember to install the tools, and the correct version of Rust.

So, I decided to give DevContainers a try, and I'm very happy with the results.

# How I did it?

The first thing, was reading the specification and understanding how it works, then I created a `.devcontainer/devcontainer.json` file on my project's root, and started playing with it. There are a lot of [available templates](https://containers.dev/templates) to work with (including a Rust + Postgres one), which I highly recommend you to check out, but as I was more interested on learning how it works, I decided to start from scratch.

The specification documentation is great, and I was able to create a basic setup in a few minutes. I took the official Rust image from Docker Hub, created a `Dockerfile` that installs the tools I need, and then created the `devcontainer.json` file that points to the `Dockerfile` on the build step, and added some custom settings.

The Dockerfile looks like this:

```Dockerfile
FROM rust:latest
# Install the development dependencies
RUN apt-get update && apt-get upgrade -y && apt-get install -y build-essential curl git libssl-dev pkg-config make postgresql-client \
    postgresql clang lld sudo vim bash-completion && apt-get clean && rm -rf /var/lib/apt/lists/*
# Create a non-root user
ENV USER_NAME=vscode
RUN useradd -m $USER_NAME -s /bin/bash
USER $USER_NAME
# Install nightly toolchain and rustfmt for it and the stable version
RUN rustup toolchain install nightly --component rustfmt clippy && rustup component add rustfmt clippy \
    && cargo install diesel_cli cargo-edit cargo-update cargo-audit cargo-udeps && mkdir -p /home/$USER_NAME/workspace
# Copy .bash_aliases, it contains several useful aliases for cargo
COPY configs/.bash_aliases /home/$USER_NAME/.bash_aliases
# Detect the postgres version and set the volume
ENV POSTGRES_VERSION=15
VOLUME /var/lib/postgresql/$POSTGRES_VERSION/main
# trust all local connections to postgres
COPY configs/pg_hba.conf /etc/postgresql/$POSTGRES_VERSION/main/pg_hba.conf
# Allow the user to run sudo without password, generate locales and set the default one
USER root
RUN echo "$USER_NAME ALL=(ALL:ALL) NOPASSWD: ALL" | tee /etc/sudoers.d/$USER_NAME && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && locale-gen && update-locale
# Delete all the cargo cache and the apt cache
RUN rm -rf /usr/local/cargo/registry && apt-get clean
```
And the `devcontainer.json` file looks like this:

```json
{
    // For format details, see https://aka.ms/devcontainer.json. For config options, see the README.md of this repo.
    "name": "rust_devcontainer",
    "build": {
        "dockerfile": "Dockerfile"
    },
    "customizations": {
        "vscode": {
            // Add the settings to use bash as the default shell
            "settings": {
                "terminal.integrated.shell.linux": "/bin/bash"
            },
            // Add the extensions that I use
            "extensions": [
                "rust-lang.rust-analyzer",
                "ms-vscode.cpptools",
                "vadimcn.vscode-lldb",
                "ms-vscode.cmake-tools",
                "twxs.cmake",
                "fill-labs.dependi",
                "tamasfe.even-better-toml",
                "GitLab.gitlab-workflow",
                "ms-ossdata.vscode-postgresql",
                "mtxr.sqltools",
                "mtxr.sqltools-driver-pg"
            ]
        }
    },
    // Set the workspace folder and the mount
    "workspaceFolder": "/home/vscode/workspace",
    "workspaceMount": "source=/var/local/development,target=/home/vscode/workspace,type=bind,consistency=cached",
    // Handle CARGO_HOME=/usr/local/cargo creating a volume for the cargo data to persist between runs
    "mounts": [
        {
            "source": "cargo-cache-rust_devcontainer",
            "target": "/usr/local/cargo",
            "type": "volume"
        },
        // Handle the postgres data directory
        {
            "source": "postgres-rust_devcontainer",
            "target": "/var/lib/postgresql/15/main",
            "type": "volume"
        }
    ],
    // Add additional arguments to the container
    "runArgs": [
        "--restart=always",
        "--name=rust_devcontainer"
    ],
    // Start the postgres service on the postStartCommand
    "postStartCommand": "sudo service postgresql start",
    // Tell VSCode to use the non-root user
    "remoteUser": "vscode"
}
```
As you can see in the `workspaceMount`, I wanted a shared container for most of my projects, so I created a symlink from the folder where these Rust projects were to `/var/local/development`, installed [devcontainer-cli](https://github.com/devcontainers/cli), ran `devcontainer up --workspace-folder .`, and finally attached to the container on VSCode. Everything was working as expected, just opened my workspace and started working on it.

**Note:** The ideal setup would be a container per specific Rust version required, then attach all the matching projects to it. That's what I ended up doing. Using a container per project is also a good idea, just have in mind that you will have a lot of VSCodes running because you can only attach to one container per instance.

That can be easily achieved by just adding a `.devcontainer/devcontainer.json` to a project, then modifying the options that you want, take a look at [my example](https://github.com/Edu4rdSHL/rust-postgres-devcontainer/blob/main/README.md).

# Conclusion

I'm very happy with the results, I can now work on my projects without worrying about the tools I need, the version of Rust that I need to use, or the OS where I'm at (not gonna lie, I use the same machine 99% of the time, but still). I can also share the environment with others, and they will have the same environment as me. I highly recommend you to give DevContainers a try, it's a great tool that will make your development a lot easier.

All the resulting code for this project is available on [Github](https://github.com/Edu4rdSHL/rust-postgres-devcontainer), and the resulting image is available on [GHCR](https://github.com/users/Edu4rdSHL/packages/container/package/rust-postgres-devcontainer).

**Important:** this project is just an showcase, and even when I'm going to keep this updated, and you are free to use, modify, and share it, I highly recommend you to check the [official templates](https://containers.dev/templates) to get started.

Happy coding! 🦀
