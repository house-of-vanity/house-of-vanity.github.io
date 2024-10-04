+++
title = "WireGuard over xRay Vless protocol"
date = "2024-10-04"
description = "How to Maintain the WireGuard Network in Censored Countries"

[taxonomies]
tags = ["linux", "wireguard", "vpn", "vless"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

# Setting Up Xray with WireGuard over Reality Protocol

In this guide, we'll walk through the steps to set up Xray-core to proxy WireGuard traffic using the Reality protocol over HTTP/2. This configuration can help bypass network restrictions and enhance privacy.

## Installing Xray-core

Install the latest beta version of Xray-core with root privileges:

```shell
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install --beta -u root
```

## Generating Configurations

First, generate the necessary keys and IDs:

```shell
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

```shell
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
                "network": "h2",
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

This configuration sets up an inbound VLESS listener over HTTP/2 with Reality security, using the generated private key and short IDs.

### Client Configuration

Create the client configuration file `client.json`:

```shell
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
                "network": "h2",
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
# Server configuration: /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.0.1/24
ListenPort = 6666
PrivateKey = <server_private_key>
PostUp = iptables -A FORWARD -i %i -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -o %i -j ACCEPT
SaveConfig = false
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

---

By combining Xray with WireGuard and the Reality protocol, you create a secure and obfuscated tunnel that can help bypass network restrictions. Remember to replace placeholder values like `<server_private_key>`, `<client_private_key>`, and `<server_public_key>` with your actual keys.