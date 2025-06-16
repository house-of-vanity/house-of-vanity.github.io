+++
title = "Shadowsocks on OpenWRT"
date = "2025-06-16"
description = "Setup shadowsocks on OpenWRT for all clients"

[taxonomies]
tags = ["linux", "networking", "openwrt"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

# Shadowsocks-libev + OpenWRT + Hardware Switch on GL.iNet

## 1. Install packages

```sh
opkg update
opkg install \
  luci-app-shadowsocks-libev \
  shadowsocks-libev-ss-redir \
  shadowsocks-libev-config
```

---

## 2. Add server + redir instance

```sh
SERVER_NAME='Bulgaria'
SERVER_ADDRESS='1.1.1.1'
SERVER_PORT=38583
SERVER_PROTO='chacha20-ietf-poly1305'
SERVER_PASS='YoUr_pASS'
LOCAL_PORT=12345

uci set shadowsocks-libev.$SERVER_NAME=server
uci set shadowsocks-libev.$SERVER_NAME.server="$SERVER_ADDRESS"
uci set shadowsocks-libev.$SERVER_NAME.server_port="$SERVER_PORT"
uci set shadowsocks-libev.$SERVER_NAME.method="$SERVER_PROTO"
uci set shadowsocks-libev.$SERVER_NAME.password="$SERVER_PASS"

uci set shadowsocks-libev.VPN_redir=ss_redir
uci set shadowsocks-libev.VPN_redir.disabled='0'
uci set shadowsocks-libev.VPN_redir.mode='tcp_and_udp'
uci set shadowsocks-libev.VPN_redir.fast_open='1'
uci set shadowsocks-libev.VPN_redir.no_delay='1'
uci set shadowsocks-libev.VPN_redir.reuse_port='1'
uci set shadowsocks-libev.VPN_redir.server="$SERVER_NAME"
uci set shadowsocks-libev.VPN_redir.local_port="$LOCAL_PORT"
```

---

## 3. Enable switch

```sh
uci set switch-button.@main[0].func='shadowsocks'
uci commit
```

Create `/etc/gl-switch.d/shadowsocks.sh`:

```sh
#!/bin/sh
action=$1
port=12345
chain=SHADOWSOCKS

if [ "$action" = "on" ]; then
    # Start ss-redir service
    /etc/init.d/shadowsocks-libev start

    # Add iptables rules
    iptables -t nat -N $chain 2>/dev/null
    iptables -t nat -F $chain
    iptables -t nat -A $chain -d 192.168.0.0/16 -j RETURN
    iptables -t nat -A $chain -p tcp -j REDIRECT --to-ports $port
    iptables -t nat -A PREROUTING -i br-lan -p tcp -j $chain

    # Drop existing connections
    conntrack -F
else
    # Delete iptables rules
    iptables -t nat -D PREROUTING -i br-lan -p tcp -j $chain
    iptables -t nat -F $chain
    iptables -t nat -X $chain

    # Stop ss-redir service
    /etc/init.d/shadowsocks-libev stop
fi
```

```sh
chmod +x /etc/gl-switch.d/shadowsocks.sh
```

Now you can enable Shadowsocks VPN using hardware switch on router. Also it's possible to start and stop VPN by running `/etc/gl-switch.d/shadowsocks.sh on/off`
