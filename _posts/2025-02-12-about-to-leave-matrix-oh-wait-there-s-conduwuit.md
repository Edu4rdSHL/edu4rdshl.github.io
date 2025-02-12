---
layout: post
title: 'About to leave Matrix... Oh wait, there''s Conduwuit!'
date: 2025-02-12 05:15 +0000
categories: ["matrix", "conduwuit", "synapse", "self-hosting"]
tags: ["matrix", "conduwuit", "synapse", "self-hosting"]
author: edu4rdshl
image:
  path: /conduwuit-matrix.jpeg
  alt: A image of Conduwuit and Matrix. Image created by the author.
excerpt: I was about to leave Matrix one week ago because of the high resource and disk space usage of Synapse. But then I found Conduwuit, a lightweight but featureful alternative to Synapse. And now I'm staying with Matrix.
---

# Introduction

I was an early adopter of Matrix around 2018, at that time there were still numerous issues with the protocol and the clients, but mainly with the federation. Sometimes, messages were lost, rooms were not synced, and there were plenty of issues with the homeserver. That made me leave Matrix around 2020.

In 2024, I decided to give Matrix another try, and I was surprised by the improvements in the protocol, the clients, and the servers. But there was a problem: the high resource and disk space usage of Synapse, the reference implementation of the Matrix homeserver. From my point of view, it's unacceptable that a homeserver with a single user on not more than 10 rooms (yes, I was on #matrix:matrix.org for ~4 months) gets **a database of 125GB** in less than a year, or several GB on first sync. I will not talk about RAM and CPU usage, but it was not the best. However, my server does have 32GB of RAM and 12 cores, which was enough for Synapse, but I know that not everyone can afford a server like that just for a single-user instance.

## Things I tried on Synapse

I tried to reduce the resource and disk usage of Synapse by:

- [Switching to PostgreSQL from SQLite](https://github.com/element-hq/synapse/blob/develop/docs/postgres.md).
- [Optimizing the PostgreSQL database](https://github.com/element-hq/synapse/blob/develop/docs/postgres.md#tuning-postgres).
- [Compressing Synapse state tables](https://github.com/matrix-org/rust-synapse-compress-state).
- [Purging rooms that I don't use anymore](https://matrix-org.github.io/synapse/v1.40/admin_api/purge_room.html).

They helped a bit, but the database size was still ~45GB after performing all of these optimizations and growing. Again, not to mention the RAM and CPU usage.

## About to leave Matrix

All the previously mentioned issues made me think about [leaving Matrix](https://mastodon.social/@edu4rdshl/113942912283975543). One of the reasonings on the linked toot, was:

> I know that federation and all have its caveats, but what about XMPP? XMPP is totally federated and doesn't have these issues at all. There's simply too much metadata (useless in 90% of the cases) in the rooms state changes, and there's no way to limit/prevent that by design.
>
> Good luck for now, conduit.rs seems promising but still lacks several basic things.

And [conduwuit](https://github.com/girlbossceo/conduwuit) (big kudos to conduit.rs for inspiring conduwuit) proved that I was right; it's possible to have a lightweight and featureful Matrix homeserver.

# Conduwuit

[Conduwuit](https://github.com/girlbossceo/conduwuit) is a lightweight but featureful Matrix homeserver, it's a fork of [conduit.rs](https://conduit.rs/), which is written in Rust.

As per the official documentation:

> conduwuit is technically a hard fork of Conduit, which is in beta. The beta status initially was inherited from Conduit, however the huge amount of codebase divergance, changes, fixes, and improvements have effectively made this beta status not entirely applicable to us anymore.
>
> conduwuit is very stable based on our rapidly growing userbase, has lots of features that users expect, and very usable as a daily driver for small, medium, and upper-end medium sized homeservers.

The differences between Conduit and Conduwuit are detailed [in their website](https://conduwuit.puppyirl.gay/differences.html).

### Installation and configuration

The installation and configuration of Conduwuit are very straightforward, especially if you're going to use the Docker image. You can find the installation instructions [here](https://conduwuit.puppyirl.gay/deploying/docker.html).

At the time of writing it, the following did the trick for me:

**Note:** replace `matrix.domain.example` with your domain, and always check the official documentation for the latest instructions.

- docker-compose.yml

```yaml
services:
    caddy:
        image: lucaslorentz/caddy-docker-proxy:ci-alpine
        ports:
            - 80:80
            - 443:443
        environment:
            - CADDY_INGRESS_NETWORKS=caddy
        networks:
            - caddy
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - ./data:/data
        restart: unless-stopped
        labels:
            caddy: matrix.domain.example
            caddy.0_respond: /.well-known/matrix/server {"m.server":"matrix.domain.example:443"}
            caddy.1_respond: /.well-known/matrix/client {"m.server":{"base_url":"https://matrix.domain.example"},"m.homeserver":{"base_url":"https://matrix.domain.example"},"org.matrix.msc3575.proxy":{"url":"https://matrix.domain.example"}}

    homeserver:
        image: girlbossceo/conduwuit:latest
        restart: unless-stopped
        volumes:
            - db:/var/lib/conduwuit
            - db-backups:/opt/conduwuit-db-backups
            - ./conduwuit.toml:/etc/conduwuit.toml
        environment:
            CONDUWUIT_CONFIG: '/etc/conduwuit.toml'
        networks:
            - caddy
        labels:
            caddy: matrix.domain.example
            caddy.reverse_proxy: "{{upstreams 6167}}"

volumes:
    db:
    db-backups:

networks:
    caddy:
        external: true
```
- conduwuit.toml
  
**Note:** the following are only the values that I changed to get it working, you can find the full configuration file [here](https://github.com/girlbossceo/conduwuit/blob/main/conduwuit-example.toml).

```toml
[global]
server_name = "matrix.domain.example"
address = ["0.0.0.0"]
port = 6167
database_path = "/var/lib/conduwuit"
database_backup_path = "/opt/conduwuit-db-backups"
database_backups_to_keep = 1
new_user_displayname_suffix = "🐧"
allow_federation = true
trusted_servers = ["matrix.org"]
```
After that, you can run:

- `docker network create caddy`
- `docker-compose up -d`

And you're good to go.

### Results

After running Conduwuit for a week, I can say that it's a great alternative to Synapse; it's lightweight, featureful, and easy to configure. The database size is ~500MB, and the RAM and CPU usage are not even noticeable. I'm happy with Conduwuit, and I'm staying with Matrix.

# Conclusion

This is the great thing about open-source software: if you don't like something, you can search for an alternative, or you can fork it and enhance it.

You can find me on Matrix at [@edu4rdshl:matrix.edu4rdshl.dev](https://matrix.to/#/@edu4rdshl:matrix.edu4rdshl.dev).

Happy chatting!
