+++
title = "qBittornt web via VPN"
date = "2023-08-25"
description = "Installing qBittornt web and VPN only download"

[taxonomies]
tags = ["linux", "torrent", "network", "selfhosting"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

## Requirements
* Ubuntu Linux server (tested on 18.04 and 20.04)
* NGINX
* Wireguard VPN config (easy to change to any other vpn)
---

## Installing

Install `qbittorrent-nox` for headless qBittorent package:
```sh
sudo apt install -y qbittorrent-nox
```

## Configuring VPN Network Namespace
Create `/usr/bin/torrent_ns` script and make it exucutable. It configures Network Namespace for qBittorent.
```sh
VPN_CFG_NAME=torrent
VPN_COMMAND="wg-quick up ${VPN_CFG_NAME}"
export SCRIPT=$(cat <<-END
#!/bin/bash
ip netns del torrent
sleep 2
ip netns add torrent
ip link add veth0 type veth peer name veth1
ip link set veth1 netns torrent
ip address add 10.99.99.1/24 dev veth0
ip netns exec torrent ip address add 10.99.99.2/24 dev veth1
ip link set dev veth0 up
ip netns exec torrent ip link set dev veth1 up
ip netns exec torrent ip route add default via 10.99.99.1
mkdir -p /etc/netns/torrent
echo nameserver 8.8.8.8 > /etc/netns/torrent/resolv.conf
sleep 3
ip netns exec torrent ${VPN_COMMAND}
sleep 3
ip netns exec torrent sudo -u ${USER} qbittorrent-nox
END
)

sudo -E -E bash -c 'cat > /usr/bin/torrent_ns << EOF
${SCRIPT}
EOF
'

sudo chmod +x /usr/bin/torrent_ns
```

## Systemd Autostart
Systemd unit to enable autostart:
```sh
export SERVICE=$(cat <<-END
[Unit]
Description=qBittorrent via vpn
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
ExecStart=/usr/bin/torrent_ns
ExecStop=/usr/bin/ip netns del torrent
END
)

sudo -E bash -c 'cat > /etc/systemd/system/qbittorrent.service << EOF
${SERVICE}
EOF
'

sudo systemctl enable --now qbittorrent.service
```

## Nginx Reverse Proxy

```js
# /etc/nginx/sites-enabled/tr.hexor.cy.conf 
server {
        listen 443 ssl http2;
        server_name tr.hexor.ru;
        include ssl.conf; # my own ssl config
        location / {
                proxy_pass http://10.99.99.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_hide_header   Referer;
                proxy_hide_header   Origin;
                proxy_set_header    Referer                 '';
                proxy_set_header    Origin                  '';
        }
}
server {
        listen 80;
        server_name tr.hexor.cy;
        listen [::]:80;
        return 302 https://$host$request_uri;
}
```
