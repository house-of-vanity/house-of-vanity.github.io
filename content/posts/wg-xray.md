+++
title = "WireGuard over Xray VLESS Protocol"
date = "2024-10-04"
description = "How to Maintain the WireGuard Network in Censored Countries"

[taxonomies]
tags = ["linux", "wireguard", "vpn", "vless"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

# Setting Up Xray with WireGuard over Reality Protocol

In this guide, we'll walk through the steps to set up Xray-core to proxy WireGuard traffic using the Reality protocol over TCP. This configuration can help bypass network restrictions and enhance privacy.

## Installing Xray-core

Install the latest beta version of Xray-core with root privileges:

```shell
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install --beta -u root
```

## Generating Configurations

First, generate the necessary keys and IDs:

```sh
# Generate X25519 keys using Xray's built-in command
_x25519=$(xray x25519)
PRIVATE_KEY=$(echo "$_x25519" | awk -F': ' '/Private key/{print $2}')
PUBLIC_KEY=$(echo "$_x25519" | awk -F': ' '/Public key/{print $2}')

# Generate a unique UUID for the client
CLIENT_UUID=$(uuidgen)

# Generate a random short ID
SHORT_IDS=$(openssl rand -hex 8)

# Define server address and port
SERVER_ADDRESS="k8s.hexor.cy"
PORT=8443
```

### Server Configuration

Create the server configuration file `server.json`:

```sh
# /usr/local/etc/xray/config.json
cat > server.json <<EOF
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "listen": "0.0.0.0",
            "port": ${PORT},
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "${CLIENT_UUID}",
                        "flow": ""
                    }
                ],
                "decryption": "none"
            },
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                    "show": false,
                    "dest": "www.google.com:443",
                    "xver": 0,
                    "serverNames": [
                        "www.google.com"
                    ],
                    "privateKey": "${PRIVATE_KEY}",
                    "shortIds": [
                        "${SHORT_IDS}"
                    ]
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom",
            "tag": "direct"
        }
    ]
}
EOF
```

This configuration sets up an inbound VLESS listener over TCP with Reality security, using the generated private key and short IDs.

### Client Configuration

Create the client configuration file `client.json`:

```sh
# /usr/local/etc/xray/config.json
cat > client.json <<EOF
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "tag": "wireguard",
            "port": 6666,
            "protocol": "dokodemo-door",
            "settings": {
                "address": "127.0.0.1",
                "port": 6666,
                "network": "udp"
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "vless",
            "settings": {
                "vnext": [
                    {
                        "address": "${SERVER_ADDRESS}",
                        "port": ${PORT},
                        "users": [
                            {
                                "id": "${CLIENT_UUID}",
                                "encryption": "none",
                                "flow": ""
                            }
                        ]
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                    "show": false,
                    "fingerprint": "chrome",
                    "serverName": "www.google.com",
                    "publicKey": "${PUBLIC_KEY}",
                    "shortId": "${SHORT_IDS}",
                    "spiderX": ""
                }
            },
            "tag": "proxy"
        }
    ]
}
EOF
```

This client configuration captures local UDP traffic (from WireGuard) and forwards it to the Xray server using the VLESS protocol with Reality security.

## Example WireGuard Setup

### Server Configuration

Set up WireGuard on the server:

```ini
# Server configuration: /etc/wireguard/homenet.conf
[Interface]
Address = 10.0.0.1/24
ListenPort = 6666
PrivateKey = <server_private_key>
PostUp = iptables -A FORWARD -i %i -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -o %i -j ACCEPT
SaveConfig = false
MTU = 1300

[Peer]
PublicKey = <peer_public_key>
AllowedIPs = 10.0.0.2/32
Endpoint = 127.0.0.1:6666  # Local UDP port proxied by Xray
PersistentKeepalive = 10
```

### Client Configuration

Set up WireGuard on the client:

```ini
# Client configuration: /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.0.2/24
PrivateKey = <client_private_key>
MTU = 1300

[Peer]
PublicKey = <server_public_key>
AllowedIPs = 10.0.0.0/24
Endpoint = 127.0.0.1:6666  # Local UDP port proxied by Xray
PersistentKeepalive = 10
```

In this setup, WireGuard traffic is sent to a local port (`6666`), which is proxied by Xray over the Reality protocol to the server.

## Routing a Single Client's Traffic through the VPN on Mikrotik

To route a specific client's traffic through the VPN using a Mikrotik router, follow these steps:

1. **Create a New Routing Table:**

   ```shell
   /routing table add fib name=vpn
   ```

   This command creates a new routing table named `vpn`, which will be used to direct traffic through the VPN interface.

2. **Mark Routing for the Specific Client:**

   ```shell
   /ip firewall mangle add action=mark-routing chain=prerouting new-routing-mark=vpn passthrough=yes src-address=192.168.90.234
   ```

   This firewall mangle rule marks all traffic originating from the client with IP address `192.168.90.234`. The `new-routing-mark=vpn` ensures that packets from this client use the `vpn` routing table.

3. **Add a Route in the VPN Routing Table:**

   ```shell
   /ip route add disabled=no distance=1 dst-address=0.0.0.0/0 gateway=homenet routing-table=vpn
   ```

   This adds a default route (`0.0.0.0/0`) to the `vpn` routing table, directing marked traffic to the `homenet` gateway (which should be the VPN interface).
