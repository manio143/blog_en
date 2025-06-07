---
title: 1. Mini server setup
images: []
---

# 1. Mini server setup

## The idea

I was browsing reddit, as one does, and stumbled over the [r/MiniPC](https://www.reddit.com/r/minipc/) and found a spreadsheet with a ton of models of small computers.
I already was thinking of having a small server at home and decided to find some nice small computer that wouldn't be very expensive.
Finally I managed to find Beelink EQR5 on Amazon with a discount of like 30%.
I decided to get it and in turn stop paying for a cheap VPS (8 euro a month).
I get more power, more ram, more storage.

What would I use it for?
For one, I need a server for hosting my dev projects by the rule of having a deployment as part of the development lifecycle.
And then I took inspiration from [r/SelfHosted](https://www.reddit.com/r/selfhosted) and decided I want to run NextCloud so that I can store my photos (especially of my son) without storage limitations.

## Step 1: Operating system

I chose Debian LTS. I'm familiar with the way Debian family systems are working and want something reliable.
I installed it without GUI, just with SSH server.

While I used WiFi during installation, I wanted to use ethernet later but it was disabled (check with `lshw -c network -sanitize`).
I could temporarily turn it on with `ip link set <name> up` and to enable it permanently I edited `/etc/network/interfaces` to add `auto <name>` and `iface <name> inet dhcp` lines.

## Step 2: Extra storage

The Beelink came with 500GB SSD and 16GB ram.
I have a small 2TB hard drive over USB3.0 which I decided to plug in there for storing my files.

Decided to try and configure `autofs` to mount that hard drive automatically.
I had used `fstab` in the past and had difficulties when the drive was not available at boot.

```sh
# /etc/auto.master
# Cutom mount point under /mnt directory
/mnt /etc/auto.mnt --timeout=60

# /etc/auto.mnt
# This will mount /mnt/media
# in async mode (faster but data could be lost if server crashed)
# preventing device file nodes and preventing setuid (executing file as its owner)
media -fstype=auto,async,nodev,nosuid :/dev/sda1
```

## Step 3: Software management

I decided to try to use [Coolify](https://coolify.io/) to manage what systems I would be running on the server.
Had to rnn the install script a few times, but it was worth it cause it sends jokes like this:

> Your momma is so fat, you need to switch to NTFS to store a picture of her.

When it installed I saved data from `/data/coolify/source/.env` to my password manager.
Then I accessed the dashboard to create an account.
Configured the notifications with Telegram via BotFather over a group chat.

## Step 4: Dynamic DNS

My ISP (Vergin Media IE) is providing IPv6 prefix leases. Once the lease expires I can get a new prefix which will cause the connected device to get a new public address.

I'm going to leverage Cloudflare as my DNS provider and follow [this guide](https://coolify.io/docs/knowledge-base/cloudflare/tunnels/all-resource) to configure a Cloudflare tunnel.
This way I get easy TLS, don't have to worry about making my IP public, and by using a wildcard domain I can dynamically create service domain without editing the DNS table each time.

## Step 5: Single Sign On

I want to protect my services with SSO using Authentik. It's easy to add it with Coolify.
I used Mailersend to get SMTP service for sending emails. Mailersend easily configured my DNS records on Cloudflare.

A thing to note here is that once a TXT DNS record for email was created on a subdomain, it no longer falls under the wildcard and I had to manually add a CNAME record for it and point it at the tunnel.

A second thing to note is that Authentik populates the protocol (http vs https) in links in the page content, so while the Cloudflare was upgrading me to HTTPS on the top level the browser was then blocking requests to API endpoints.
To mitigate this I had to follow [this doc](https://coolify.io/docs/knowledge-base/cloudflare/tunnels/full-tls) and create a HTTPS rule in the tunnel just for the specific subdomain, as well as change service to listen on https://*:9000 to trick proxy to into using http port 9000 while using https for resources in HTML.

TODO: setting up flow for users

## Step 6: Monitoring

I found out about [Victoria Metrics](https://victoriametrics.com/) and decided to try and use it for me home lab monitoring.

I used [Docker compose environment for VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics/tree/master/deployment/docker#docker-compose-environment-for-victoriametrics) to set up the metrics, logs and Grafana.

TODO: Describe the setup once we figure out the proxy issue with Grafana

Setup: [/homeserver/telemetry](https://github.com/manio143/homeserver/tree/main/telemetry)

## Extra 1: DNS + DHCP

I wanted to try running AdGuard Home to proxy DNS requests and allow blocking some ads domains and potentially adult content (for when my baby becomes a child).
It was very easy to install with Coolify, but then the challenge was to get all devices on my network to connect to it.

Because I have a Virgin Media Hub router with no setting for DNS, I had to resort to disabling DHCP on the router and instead running my own server.
Luckily that's built into AdGuard.

What I had to figure out is to create a docker network using [`macvlan` driver](https://docs.docker.com/engine/network/tutorials/macvlan/) which would make my container become part of the outer network of my router and start responding to DHCP requests.

```
docker network create -d macvlan \
  --subnet=192.168.0.0/24 \
  --gateway=192.168.0.1 \
  -o parent=eth0 \
  local_macvlan
```

```yaml
services:
  adguard:
    image: adguard/adguardhome:latest
    restart: unless-stopped
    volumes:
      - ${WORK_DIR}:/opt/adguardhome/work
      - ${CONF_DIR}:/opt/adguardhome/conf
    ports:
      - "53:53/udp" #DNS
      - "53:53/tcp"
      - "67:67/udp" #DHCP
      - "68:68/udp" #DHCP
      - "68:68/tcp"
    networks:
      local_macvlan:
        ipv4_address: 192.168.0.2 # static IP

networks:
  local_macvlan:
    name: local_macvlan
    external: true
```

And it works very nicely. I kept IPv6 stateless distribution within the router, while IPv4 and DNS location is provided by my server.
