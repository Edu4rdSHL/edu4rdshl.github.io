---
layout: post
title: 'From Docker to Podman: full migration to rootless'
date: 2025-02-21 06:31 +0000
categories: ["docker", "podman"]
tags: ["docker", "podman", "rootless"]
author: edu4rdshl
image:
  path: /docker-to-podman.png
  alt: Image of Docker and Podman logos. From the internet.
excerpt: Migrate from Docker to rootless Podman, including volumes, networks, ports settings, container's metadata (mounts, restart policy, etc), and more.
---

# Introduction

There are many posts about migrating from Docker to Podman, but most of them are focused on the basics... `docker export` / `podman import`. This post is about a full migration, including volumes data, container's metadata, and more. Everything, into a rootless Podman environment.

I have created a tool called [fly-to-podman](https://github.com/Edu4rdSHL/fly-to-podman) that automates the migration process from what I'm going to explain here, please use that tool instead of the manual process because the post will not be updated with the latest changes.

# Prerequisites

Start with the obvious, you need to have Docker and Podman installed, and Docker needs to be running.

Appart from that, you need to have `jq`, `rsync`, and `bash`.

# Migration

For rootless Podman, all the data is stored in `~/.local/share/containers/` and the configuration can be changed in `~/.config/containers/`. They will change if you aren't using rootless Podman, so adjust the paths in the post accordingly.

## Images

First, let's migrate the images. We will use `docker save` and `podman load`:

{% raw %}
```bash
migrate_images() {
    echo "Migrating Docker images to Podman..."
    # Get a list of all Docker images (name:tag)
    docker images --format "{{.Repository}}:{{.Tag}}" | while read -r image; do
        # Skip <none>:<none> images
        if [[ "$image" == "<none>:<none>" ]]; then
            continue
        fi

        # Replace slashes in repository names with underscores for filenames
        filename=$(echo "$image" | tr '/' '_').tar

        echo "Exporting $image..."
        docker save -o "$filename" "$image" &&
            podman load -i "$filename" &&
            echo "Image $image migrated to Podman" || echo "Failed to migrate image $image"

        # Remove temporary file
        rm -f "$filename"
    done
}
```
{% endraw %}

This small script will export all Docker images to a tarball, then import them into Podman. It will skip `<none>:<none>` images.

## Volumes data

Volumes data is stored in `/var/lib/docker/volumes/` for Docker and in `~/.local/share/containers/storage/volumes/` for Podman. We can create the same volumes names in Podman so that they are tracked in the database and then copy the data, and chown it accordingly.

{% raw %}
```bash
migrate_volumes() {
    echo "Migrating Docker volumes to Podman..."
    # Get the path to the Podman volumes directory
    PODMAN_VOLUMES_PATH=$(podman info --format json | jq -r '.store.volumePath')
    DOCKER_VOLUMES_PATH="/var/lib/docker/volumes"

    RSYNC_OPTS=""
    if [[ "$UID" -ne 0 ]]; then
        # If not running as root, make sure to chown the files to the current user
        RSYNC_OPTS+=" --chown=$(id -u):$(id -g)"
    fi

    for volume in $(docker volume ls --format json | jq -r '.Name'); do
        echo "Migrating volume: $volume"
        podman volume create "$volume" &&
            sudo rsync -a "$RSYNC_OPTS" "$DOCKER_VOLUMES_PATH/$volume/_data/" "$PODMAN_VOLUMES_PATH/$volume/_data"
    done
}
```
{% endraw %}

## Networks

We are going to migrate the networks while preserving:

- The network name
- The network driver
- The subnet
- The gateway
- The IP range

{% raw %}
```bash
migrate_networks() {
    echo "Migrating Docker networks to Podman..."

    # Get all Docker networks
    for network in $(docker network ls --format '{{json . }}' | jq -r '.Name'); do
        echo "Processing network: $network"

        # Skip default Docker networks
        if [[ "$network" == "host" || "$network" == "none" ]]; then
            echo "Skipping network: $network (Podman does not need it)"
            continue
        fi

        # Extract network details
        NETWORK_JSON=$(docker network inspect "$network" | jq '.[0]')
        DRIVER=$(echo "$NETWORK_JSON" | jq -r '.Driver')
        SUBNET=$(echo "$NETWORK_JSON" | jq -r '.IPAM.Config[0].Subnet // empty')
        GATEWAY=$(echo "$NETWORK_JSON" | jq -r '.IPAM.Config[0].Gateway // empty')
        IP_RANGE=$(echo "$NETWORK_JSON" | jq -r '.IPAM.Config[0].IPRange // empty')

        # Check if the network already exists in Podman
        if podman network exists "$network"; then
            echo "Network $network already exists in Podman. Skipping creation."
            continue
        fi

        # Build the Podman network create command
        PODMAN_NET_CMD="podman network create"

        # Add driver (Podman supports `bridge`, `ipvlan` and `macvlan`)
        case "$DRIVER" in
        "bridge")
            PODMAN_NET_CMD+=" --driver bridge"
            ;;
        "macvlan")
            PODMAN_NET_CMD+=" --driver macvlan"
            ;;
        "ipvlan")
            PODMAN_NET_CMD+=" --driver ipvlan"
            ;;
        *)
            echo "Warning: Unsupported network driver '$DRIVER' in Podman. Using default bridge."
            PODMAN_NET_CMD+=" --driver bridge"
            ;;
        esac

        # Add subnet configuration if available
        if [[ -n "$SUBNET" ]]; then
            PODMAN_NET_CMD+=" --subnet $SUBNET"
        fi

        # Add gateway if available
        if [[ -n "$GATEWAY" ]]; then
            PODMAN_NET_CMD+=" --gateway $GATEWAY"
        fi

        # Add IP range if available
        if [[ -n "$IP_RANGE" ]]; then
            PODMAN_NET_CMD+=" --ip-range $IP_RANGE"
        fi

        # Finalize the command with network name
        PODMAN_NET_CMD+=" $network"

        # Create the Podman network
        echo "Creating Podman network: $network"
        eval "$PODMAN_NET_CMD" && echo "Network $network migrated successfully." ||
            echo "Failed to migrate network: $network"

    done
}
```
{% endraw %}

## Containers

Containers data is stored in `/var/lib/docker/containers/` for Docker and in `~/.local/share/containers/storage/` for Podman. We require a more complex script to migrate the containers data, including metadata like mounts, restart policy, etc.

{% raw %}
```bash
migrate_containters() {
    echo "Migrating Docker containers to Podman..."
    for container in $(docker container ls -a --format json | jq -r '.Names'); do
        # Convert container name to lowercase
        container_lc=$(echo "$container" | tr '[:upper:]' '[:lower:]')
        # Tag for the image to be created from the container
        MIGRATION_CONTAINER_TAG="podman.local/${container_lc}-to-podman:latest"
        # Get Running status from Docker
        WAS_RUNNING=$(docker container inspect -f '{{.State.Running}}' "$container")
        # Get RestartPolicy from Docker
        RESTART_POLICY=$(docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' "$container")

        # Pass the restart policy to Podman
        case "$RESTART_POLICY" in
        "no") PODMAN_RESTART="" ;;
        "always") PODMAN_RESTART="--restart=always" ;;
        "unless-stopped") PODMAN_RESTART="--restart=unless-stopped" ;;
        "on-failure") PODMAN_RESTART="--restart=on-failure" ;;
        *) PODMAN_RESTART="" ;;
        esac

        echo "Processing container: $container"

        # Commit container to an image. It lets us start a new container with the _same_ state and add additional options
        docker commit "$container" "$MIGRATION_CONTAINER_TAG" &&
            docker save -o "$container_lc".tar "$MIGRATION_CONTAINER_TAG" &&
            podman load -i "$container_lc".tar || {
            echo "Failed to migrate image for $container"
            continue
        }

        # Extract volume/bind mount information from Docker container
        MOUNT_OPTS=""
        while read -r mount; do
            MOUNT_TYPE=$(echo "$mount" | jq -r '.Type')
            SOURCE=$(echo "$mount" | jq -r '.Source')
            DESTINATION=$(echo "$mount" | jq -r '.Destination')
            READ_WRITE=$(echo "$mount" | jq -r '.RW')

            # Pass the RW/RO setting to Podman
            if [[ "$READ_WRITE" == "true" ]]; then
                MODE="rw"
            else
                MODE="ro"
            fi

            if [[ "$MOUNT_TYPE" == "volume" ]]; then
                # Use :U to ensure right permissions inside the container.
                # It tells Podman to use the correct host UID and GID based on the UID and GID within the <<container|pod>>
                MODE+=",U"
                # Attach existing named volume
                VOLUME_NAME=$(echo "$mount" | jq -r '.Name')
                MOUNT_OPTS+=" -v $VOLUME_NAME:$DESTINATION:$MODE"
            elif [[ "$MOUNT_TYPE" == "bind" ]]; then
                # Use :Z if you're using SELinux to ensure right permissions inside the container
                # MODE+=",Z"
                # Ensure the source path exists before mounting
                [[ -e "$SOURCE" ]] && MOUNT_OPTS+=" -v $SOURCE:$DESTINATION:$MODE"
            fi
        done < <(docker inspect "$container" | jq -c '.[0].Mounts[]')

        # Extract port mappings
        PORT_OPTS=""
        while read -r port_mapping; do
            HOST_IP=$(echo "$port_mapping" | jq -r '.HostIp')
            HOST_PORT=$(echo "$port_mapping" | jq -r '.HostPort')
            CONTAINER_PORT=$(echo "$port_mapping" | jq -r '.ContainerPort')
            PROTOCOL=$(echo "$port_mapping" | jq -r '.Protocol')

            # Construct `-p` option (exclude 0.0.0.0 for readability)
            if [[ "$HOST_IP" == "0.0.0.0" || -z "$HOST_IP" ]]; then
                PORT_OPTS+=" -p $HOST_PORT:$CONTAINER_PORT/$PROTOCOL"
            else
                PORT_OPTS+=" -p $HOST_IP:$HOST_PORT:$CONTAINER_PORT/$PROTOCOL"
            fi
        done < <(docker inspect "$container" | jq -c '.[] | .NetworkSettings.Ports | to_entries[] | {ContainerPort: .key, Protocol: (if .key | contains("udp") then "udp" else "tcp" end), HostMappings: .value} | select(.HostMappings != null) | .HostMappings[] | {HostIp, HostPort, ContainerPort, Protocol}')

        # Extract network information
        NETWORK_OPTS=""
        while read -r network; do
            NETWORK_NAME=$(echo "$network" | jq -r 'keys[0]')
            NETWORK_IP=$(echo "$network" | jq -r ".$NETWORK_NAME.IPAddress")

            if [[ -n "$NETWORK_NAME" ]]; then
                NETWORK_OPTS+=" --network=$NETWORK_NAME"
            fi

            if [[ -n "$NETWORK_IP" ]]; then
                NETWORK_OPTS+=" --ip=$NETWORK_IP"
            fi
        done < <(docker inspect "$container" | jq -c '.[0].NetworkSettings.Networks')

        # Run the container with the same name and mounts, including RW/RO options
        podman run -d --name "$container" $PODMAN_RESTART "$MOUNT_OPTS" "$PORT_OPTS" "$NETWORK_OPTS" "$MIGRATION_CONTAINER_TAG" &&
            echo "Container $container migrated successfully" ||
            echo "Failed to migrate container: $container"

        # Stop the container if this was not running, this allow us to keep the container ready to `podman container start $container`
        if [[ "$WAS_RUNNING" == "false" ]]; then
            podman stop "$container"
        fi
    done
}
```
{% endraw %}

After running the scripts on that order, you should have all your Docker containers, networks, images, and volumes migrated to rootless Podman with the same state as before.

# Things I like about Podman

- **Rootless**: Podman is rootless by default, which is a big plus for security.
- **Docker compatibility**: Podman is mostly compatible with Docker, so you can use the same commands.
- **Autoupdates**: On Docker, you need to use something like [Watchtower](https://github.com/containrrr/watchtower) to update your containers, but Podman has [podman auto-update](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html) built-in.
- **Podman-compose**: Podman has a built in `podman compose` command that is compatible with `docker-compose.yml` files and with the `docker-compose` provider.

# Caveats

After migrating from Docker to Podman, you may encounter some issues:

- **SELinux**: Podman is more strict than Docker with SELinux, so you may need to adjust the labels on your files.
- **Networks**: Podman does not support all the network drivers that Docker does, so you may need to adjust the network driver in the script.
- **Volumes**: Podman requires some additional options like `:U` or `:Z` to ensure the right permissions inside the container.

Some minor but still annoying issues:

- The system journal will gets spammed with [podman-events](https://docs.podman.io/en/stable/markdown/podman-events.1.html) messages. The only way to disable them is to set `events_logger = "none"` in `~/.config/containers/containers.conf`, but then you won't be able to use `podman container logs`, which is a big downside.
- ~~Even if you set a default network in `~/.config/containers/containers.conf`, [Podman will ignore it](https://github.com/containers/podman/issues/25377).~~ The documentation was not clear, you can actually set a default network in `~/.config/containers/containers.conf` using the `netns` option.

I ended up with the following configuration in `~/.config/containers/containers.conf` to workaround the issues:

```conf
[engine]
compose_warning_logs = false
healthcheck_events = false
events_logfile_path="/home/edu4rdshl/.local/share/containers/containers.log"
events_logger = "file"
events_logfile_max_size = "1m"

[containers]
default_sysctls = [
  "net.ipv4.ping_group_range=0 10000",
]
netns = "podman_rootless" # The default network that you want to use for containers, can be overriden with --network
```

# Tips

- Install [podman-docker](https://github.com/containers/podman/tree/main/docker) to continue using `docker` commands with Podman. Or simply create and alias for `docker=podman`.
- If you use `docker-compose.yml` files, you can install the `docker-compose` package and then run `podman compose`. See the [official documentation](https://docs.podman.io/en/stable/markdown/podman-compose.1.html) for more information.

# Conclusion

It's still too early to say if Podman is a better alternative to Docker, but being rootless by default is nice. There may be some things that I haven't covered in this post, feel free to open an issue or pull request on the [fly-to-podman](https://github.com/Edu4rdSHL/fly-to-podman) repository.
