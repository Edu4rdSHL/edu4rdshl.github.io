---
layout: post
title: 'Setting up an Ejabberd XMPP server with Video Calls and Multimedia support'
date: 2026-01-02 15:07 -0500
categories: ["ejabberd", "xmpp", "video calls", "multimedia", "docker"]
tags: ["ejabberd", "xmpp", "video calls", "multimedia", "docker"]
author: edu4rdshl
image:
  path: /2026-01-02/ejabberd-logo.jpeg
  alt: Ejabberd logo. Taken from https://www.ejabberd.im/
excerpt: A step-by-step guide on how to set up an Ejabberd XMPP server with Video Calls and Multimedia support using Docker and PostgreSQL.
---

## Introduction

XMPP (Extensible Messaging and Presence Protocol) is something that has always caught my attention. I will always wonder why it never became mainstream and the de-facto standard for Instant Messaging, given its capabilities and extensibility. I have been testing and using XMPP since ~2013, but of course I had always used public servers, never setting up my own server. Last December, while the Christmas holidays were taking place, I decided to set up my own XMPP server (who said that it was not fun for the whole family?).

**Note:** big kudos to the [Ejabberd's XMPP room](xmpp:ejabberd@conference.process-one.net?join) for all the help and guidance they provided me while setting up the server!

## Choosing the XMPP server software

There are several well-known XMPP server implementations available, such as Ejabberd, Prosody, and Openfire. I decided to go with Ejabberd basically because the best servers I had used in the past were running Ejabberd (conversations.im was my reference server). Additionally, it has a very active community, good documentation, and has been very well maintained over the years.

## Setting up Ejabberd with Docker and PostgreSQL

Here's where the pain starts. While Ejabberd provides an official Docker image and great documentation, now you need to start checking which module, with which options, and with which dependencies you need to make a certain XEP (XMPP Extension Protocol) work. In my case, I wanted to have Calls, Video Calls, File Transfer support, status indicators for uses (e.g., typing, online, offline), mesage delivery receipts, and other features that are expected from a modern IM platform. After a few hours, I got a server that was at least being able to send and receive messages, but I couldn't send files or make calls.

Trying to find a working configuration for Ejabberd with all the required modules, I searched for "ejabberd.yml configuration file" and to my surprise, the Ejabberd repository already provides an [example file](https://github.com/processone/ejabberd/blob/master/ejabberd.yml.example) that already works very well. With that on my hands and the experience from the previous attempts, I just had to tweak a few options to make it work the way I wanted.

**Note:** from now on, replace `jabber.edu4rdshl.dev` with your own domain, IP addresses with your own ones, and set strong passwords for the admin user and the PostgreSQL database. I will also assume that you're using a folder named `ejabberd` in your home directory to store all the required files.

### The ejabberd.yml file

Here's the `ejabberd.yml` file that I ended up using:

{% raw %}
```yaml
hosts:
  - "jabber.edu4rdshl.dev"

loglevel: 4

acl:
  admin:
    user: "admin@jabber.edu4rdshl.dev"
  local:
    user_regexp: ""

access_rules:
  local:
    allow: local
  c2s:
    deny: blocked
    allow: all
  announce:
    allow: admin
  configure:
    allow: admin
  muc_create:
    allow: local
  pubsub_createnode:
    allow: local
  trusted_network:
    allow: loopback

## PostgreSQL Configuration
sql_type: pgsql
sql_server: postgres
sql_port: 5432
sql_database: ejabberd
sql_username: ejabberd
sql_password: SuperSecretPostgresPassword
default_db: sql

# Global S2S settings
s2s_use_starttls: optional

# Listening ports
listen:
  - port: 5222
    ip: "::"
    module: ejabberd_c2s
    shaper: c2s_shaper
    access: c2s
    max_stanza_size: 262144
    starttls_required: true

  - port: 5223
    ip: "::"
    module: ejabberd_c2s
    shaper: c2s_shaper
    access: c2s
    max_stanza_size: 262144
    tls: true

  - port: 5269
    ip: "::"
    module: ejabberd_s2s_in
    max_stanza_size: 524288
    shaper: s2s_shaper


  - port: 5270
    ip: "::"
    module: ejabberd_s2s_in
    max_stanza_size: 524288
    shaper: s2s_shaper
    tls: true

  - port: 5280
    ip: "::"
    module: ejabberd_http
    request_handlers:
      /admin: ejabberd_web_admin

  - port: 5443
    ip: "::"
    module: ejabberd_http
    tls: true
    request_handlers:
      /api: mod_http_api
      /bosh: mod_bosh
      /captcha: ejabberd_captcha
      /upload: mod_http_upload
      /ws: ejabberd_http_ws
      /.well-known/host-meta: mod_host_meta
      /.well-known/host-meta.json: mod_host_meta

  - port: 5478
    ip: "::"
    transport: udp
    module: ejabberd_stun
    use_turn: true
    turn_min_port: 49152
    turn_max_port: 50000
    ## The server's public IPv4 address:
    turn_ipv4_address: "158.69.216.228"
    ## The server's public IPv6 address:
    turn_ipv6_address: "2607:5300:205:200::6ac7"

certfiles:
  - "/etc/letsencrypt/live/jabber.edu4rdshl.dev/privkey.pem"
  - "/etc/letsencrypt/live/jabber.edu4rdshl.dev/fullchain.pem"

# Module configuration
modules:
  mod_adhoc: {}
  mod_adhoc_api: {}
  mod_admin_extra: {}
  mod_announce:
    access: announce
  mod_avatar: {}
  mod_blocking: {}
  mod_bosh: {}
  mod_caps: {}
  mod_carboncopy: {}
  mod_client_state: {}
  mod_configure: {}
  mod_disco: {}
  mod_host_meta:
    bosh_service_url: "https://@HOST@:5443/bosh"
    websocket_url: "wss://@HOST@:5443/ws"
  mod_http_api: {}
  mod_http_upload:
    put_url: "https://@HOST@:5443/upload"
    docroot: /opt/ejabberd/upload
    max_size: 524288000
    custom_headers:
      "Access-Control-Allow-Origin": "https://@HOST@"
      "Access-Control-Allow-Methods": "GET,HEAD,PUT,OPTIONS"
      "Access-Control-Allow-Headers": "Content-Type"
  mod_last: {}
  mod_mam:
    db_type: sql
    assume_mam_usage: true
    default: always
  mod_muc:
    access:
      - allow
    access_admin:
      - allow: admin
    access_create: muc_create
    access_mam:
      - allow
    access_persistent: muc_create
    default_room_options:
      mam: true
    max_users: 10000
  mod_muc_admin: {}
  mod_muc_occupantid: {}
  mod_offline:
    access_max_user_messages: max_user_offline_messages
  mod_ping: {}
  mod_privacy: {}
  mod_private: {}
  mod_proxy65:
    access: local
    max_connections: 5
  mod_pubsub:
    access_createnode: all
    plugins:
      - flat
      - pep
    force_node_config:
      ## Avoid buggy clients to make their bookmarks public
      storage:bookmarks:
        access_model: whitelist
  mod_push: {}
  mod_roster:
    versioning: true
  mod_s2s_bidi: {}
  mod_s2s_dialback: {}
  mod_shared_roster: {}
  mod_stream_mgmt:
    resend_on_timeout: if_offline
  mod_stun_disco: {}
  mod_vcard: {}
  mod_vcard_xupdate: {}
  mod_version:
    show_os: false
```
{% endraw %}

### The SSL certificates

It's highly recommended to use valid SSL certificates for your XMPP server, as most XMPP clients will refuse to connect to servers with self-signed or invalid certificates. You can use [Let's Encrypt](https://letsencrypt.org/) to obtain free SSL certificates for your domain. Make sure to mount the certificate files into the Docker container, as shown in the `docker-compose.yml` file below.

To automate the process of obtaining and renewing Let's Encrypt certificates, you can use [Certbot](https://certbot.eff.org/), which even has [DNS plugins](https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins) for most popular DNS providers (in cases where you can't or don't want to bind port 80 on your server). In my case, I used the Cloudflare DNS plugin to obtain the certificates. You need a Cloudflare API token with permissions to manage DNS records for your domain, and then create a file named `cloudflare-dns.ini` inside the current folder where you are working at. It needs to use the following format:

```ini
# Cloudflare API token used by Certbot
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN_HERE
```

After that, you can run the following command to obtain the certificates:

```bash
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials ~/ejabberd/cloudflare-dns.ini -d jabber.edu4rdshl.dev -d "*.jabber.edu4rdshl.dev"
```

That will create the required certificates, and also create an automatic renewal process using cron jobs. After each renewal, you need to reload the Ejabberd server to make it use the new certificates. You can do that by adding a deploy hook in the `/etc/letsencrypt/renewal-hooks/deploy` directory with the following content:

```bash
#!/bin/bash

# Fix ownership so ejabberd user can read the new certs
chown -R 9000:9000 /etc/letsencrypt/live/jabber.edu4rdshl.dev
chown -R 9000:9000 /etc/letsencrypt/archive/jabber.edu4rdshl.dev

# Reload ejabberd to apply the new certificate
docker compose -f /home/ubuntu/ejabberd/docker-compose.yml exec ejabberd ejabberdctl reload_config

echo "$(date) - Certbot deploy hook: Completed successfully" >> /var/log/certbot-deploy.log
```

### Docker dual-stack networking (IPv4 and IPv6)

As we are going to use both IPv4 and IPv6 on our XMPP server, we need to configure Docker to work with both IPv4 and IPv6 inside the containers. Add the following configuration to your Docker daemon configuration file, usually located at `/etc/docker/daemon.json`:

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd12:3456:789a:1::/64"
}
```

### The docker-compose.yml file

The docker image requires a [few volumes](https://docs.ejabberd.im/CONTAINER/#volumes) to persist data, in addition to a PostgreSQL (if you want to use it - recommended) as the database. You can use the official PostgreSQL Docker image for that.

Here's how the full `docker-compose.yml` file looks like:

{% raw %}
```yaml
services:
  ejabberd:
    image: ghcr.io/processone/ejabberd:latest
    container_name: ejabberd
    restart: unless-stopped
    depends_on:
      - postgres
    volumes:
      - ./ejabberd.yml:/opt/ejabberd/conf/ejabberd.yml:ro
      - ejabberd_database:/opt/ejabberd/database
      - ejabberd_logs:/opt/ejabberd/logs
      - ejabberd_upload:/opt/ejabberd/upload
      - /etc/letsencrypt/live/jabber.edu4rdshl.dev:/etc/letsencrypt/live/jabber.edu4rdshl.dev:ro
      - /etc/letsencrypt/archive/jabber.edu4rdshl.dev:/etc/letsencrypt/archive/jabber.edu4rdshl.dev:ro
    ports:
      - "5222:5222"    # C2S STARTTLS
      - "5223:5223"    # C2S TLS
      - "5269:5269"    # S2S STARTTLS
      - "5270:5270"    # S2S TLS
      - "5280:5280"    # HTTP (admin – preferably an internal bind, but published if needed)
      - "5443:5443"    # HTTPS (API, BOSH, upload, WS, etc.)
      - "5478:5478/udp" # STUN/TURN UDP
      - "49152-50000:49152-50000/udp" # TURN media relay
    environment:
      - EJABBERD_MACRO_HOST=jabber.edu4rdshl.dev
      - REGISTER_ADMIN_PASSWORD=SuperSecretAdminPassword
    networks:
      - ejabberd_net
  postgres:
    image: postgres:18-alpine
    container_name: ejabberd_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ejabberd
      POSTGRES_USER: ejabberd
      POSTGRES_PASSWORD: SuperSecretPostgresPassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - ejabberd_net

networks:
  ejabberd_net:
    enable_ipv6: true

volumes:
  postgres_data:
  ejabberd_database:
  ejabberd_logs:
  ejabberd_upload:
```
{% endraw %}

### The firewall configuration

Depending on what you want to achieve, you need to open the required ports in your firewall. In my case, I opened the following ports:

- `5222/tcp`: for C2S connections with STARTTLS
- `5223/tcp`: for C2S connections with TLS
- `5269/tcp`: for S2S connections with STARTTLS
- `5270/tcp`: for S2S connections with TLS
- `5443/tcp`: for HTTPS connections (API, BOSH, upload, WS, etc.)
- `5478/udp`: for STUN/TURN UDP
- `49152-50000/udp`: for TURN media relay

Additionally I have port 5280/tcp open, but only for my wireguard VPN network.

So, the `sudo ufw status` command shows something like this:

```
Status: active
To                         Action      From
--                         ------      ----
5222/tcp                   ALLOW       Anywhere
5223/tcp                   ALLOW       Anywhere
5269/tcp                   ALLOW       Anywhere
5270/tcp                   ALLOW       Anywhere
5443/tcp                   ALLOW       Anywhere
5478/udp                   ALLOW       Anywhere
49152:50000/udp            ALLOW       Anywhere
5280/tcp on wg0            ALLOW IN    Anywhere
# also the same rules for IPv6
```

### The DNS configuration

You need to set up the required DNS records for your XMPP server to work properly. At a minimum, you need to set up an `A` record for your domain pointing to your server's IP address and optionally an `AAAA` record if you have an IPv6 address. Additionally, you might want to set up the following SRV records:

```
_xmpp-client._tcp.jabber.edu4rdshl.dev. 3600 IN SRV 5 0 5222 jabber.edu4rdshl.dev.
_xmpp-server._tcp.jabber.edu4rdshl.dev. 3600 IN SRV 5 0 5269 jabber.edu4rdshl.dev.
_xmpps-client._tcp.jabber.edu4rdshl.dev. 3600 IN SRV 5 0 5223 jabber.edu4rdshl.dev.
_xmpps-server._tcp.jabber.edu4rdshl.dev. 3600 IN SRV 5 0 5270 jabber.edu4rdshl.dev.
_stun._udp.jabber.edu4rdshl.dev.        3600 IN SRV 5 0 5478 jabber.edu4rdshl.dev.
_turn._udp.jabber.edu4rdshl.dev.        3600 IN SRV 5 0 5478 jabber.edu4rdshl.dev.
```

### Testing the server

After setting up everything, you can start the Ejabberd server using Docker Compose:

```bash
docker compose up -d
```

Now, you can access your admin panel at `https://your_private_ip:5443/admin` using the admin user and password you set in the `docker-compose.yml` file and create a new user for testing. After that, you can use an XMPP client like [Conversations](https://conversations.im/) (Android) or [Gajim](https://gajim.org/) (Linux, Windows) to connect to your server and test its functionality. If everything is set up correctly, you should be able to send messages, make calls, and transfer files between users and other XMPP servers.

## Conclusion

While it took me some time to set up everything correctly, having my own XMPP server with Video Calls and Multimedia support has been a rewarding experience. Ejabberd's flexibility and extensibility make it a great choice for anyone looking to set up their own XMPP server. I hope that you enjoy this straightforward guide to set up your own Ejabberd server using Docker and PostgreSQL!

Happy chatting!
