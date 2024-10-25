+++
title = "WireGuard over udp2raw"
date = "2024-10-25"
description = "Running WireGuard over udp2raw"

[taxonomies]
tags = ["linux", "vpn", "wireguard"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++
# Running WireGuard over udp2raw

## Introduction

In certain network environments, establishing a direct WireGuard connection may be challenging due to firewalls, NAT restrictions, or ISPs blocking UDP traffic. **udp2raw** provides a solution by encapsulating WireGuard's UDP traffic into encrypted packets that mimic TCP or ICMP protocols. This method helps bypass UDP blocking and can make your VPN connection appear as regular TCP or ICMP traffic.

## Prerequisites

- **Server**: A remote server with root access where WireGuard is installed and configured.
- **Client**: A local machine with root access where WireGuard is installed and configured.
- **udp2raw**: Downloaded on both the server and client from the [udp2raw releases page](https://github.com/wangyu-/udp2raw/releases).
- **A shared secret key**: A plain text string used by udp2raw for authentication (e.g., `SecReT-StrinG`).

## How It Works

```
[WireGuard Client]
    |
    | UDP traffic to 127.0.0.1:6666
    v
+---------------------+
| udp2raw Client      |
| Listening on:       |
| 127.0.0.1:6666      |
| Connecting to:      |
| SERVER_IP:7777      |
+---------------------+
    |
    | Encrypted UDP/TCP/ICMP packets to SERVER_IP:7777
    v
~~~~~~~~~~~~~ Internet ~~~~~~~~~~~~~
    |
    v
+---------------------+
| udp2raw Server      |
| Listening on:       |
| 0.0.0.0:7777        |
| Forwarding to:      |
| 127.0.0.1:51820     |
+---------------------+
    |
    | UDP traffic to 127.0.0.1:51820
    v
[WireGuard Server]
```

### Data Flow Explanation

1. **WireGuard Client** sends UDP packets to `127.0.0.1:6666`, which is the local udp2raw Client.
2. **udp2raw Client** encapsulates these UDP packets into encrypted UDP/TCP/ICMP packets and sends them over the internet to `SERVER_IP:7777` (the udp2raw Server).
3. **udp2raw Server** listens on `0.0.0.0:7777`, decrypts the incoming packets, and forwards them as UDP packets to `127.0.0.1:51820` (the WireGuard Server).
4. **WireGuard Server** processes the packets and sends responses back through the same path in reverse.

## Steps

### 1. Download and Install udp2raw

On both the **server** and **client**:

1. Navigate to the [udp2raw releases page](https://github.com/wangyu-/udp2raw/releases).
2. Download the appropriate binary for your system (e.g., `udp2raw_amd64` for 64-bit Linux).
3. Make the binary executable and move it to a directory in your `$PATH`:

   ```bash
   chmod +x udp2raw_amd64
   sudo mv udp2raw_amd64 /usr/local/bin/
   ```

### 2. Configure udp2raw on the Server

Create a systemd service to run udp2raw as a daemon.

#### Command Explanation

- `-s`: Run in server mode.
- `-l 0.0.0.0:7777`: Listen on all interfaces on port `7777`.
- `-r 127.0.0.1:51820`: Forward packets to `127.0.0.1` on port `51820` (WireGuard listens here).
- `-k <PLAIN_TEXT_SECRET>`: Use the provided key for authentication.
- `--raw-mode udp`: Use UDP mode for raw sockets.
- `-a`: Auto-adjust TCP options.

#### Create Systemd Service

Create a file `/etc/systemd/system/udp2raw.service` with the following content:

```ini
[Unit]
Description=udp2raw Server
After=network.target

[Service]
ExecStart=/usr/local/bin/udp2raw_amd64 -s -l 0.0.0.0:7777 -r 127.0.0.1:51820 -k SecReT-StrinG --raw-mode udp -a
Restart=always
User=root
RestartSec=3

[Install]
WantedBy=multi-user.target
```

#### Start the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable udp2raw
sudo systemctl start udp2raw
```

### 3. Configure udp2raw on the Client

#### Command Explanation

- `-c`: Run in client mode.
- `-l 127.0.0.1:6666`: Listen on local address `127.0.0.1` on port `6666` (WireGuard will connect here).
- `-r SERVER_IP:7777`: Connect to the udp2raw server at `SERVER_IP` on port `7777`.
- `-k <PLAIN_TEXT_SECRET>`: Use the same key as the server for authentication.
- `--raw-mode udp`: Use UDP mode for raw sockets.
- `-a`: Auto-adjust TCP options.

#### Create Systemd Service

Replace `SERVER_IP` with your server's IP address.

Create a file `/etc/systemd/system/udp2raw.service` with the following content:

```ini
[Unit]
Description=udp2raw Client
After=network.target

[Service]
ExecStart=/usr/local/bin/udp2raw_amd64 -c -l 127.0.0.1:6666 -r SERVER_IP:7777 -k SecReT-StrinG --raw-mode udp -a
Restart=always
User=root
RestartSec=3

[Install]
WantedBy=multi-user.target
```

#### Start the Service

```bash
sudo systemctl enable --now udp2raw
```

### 4. Configure WireGuard to Use udp2raw

On the **client**, modify your WireGuard configuration to connect to the local udp2raw port.

#### Example WireGuard Client Configuration

```ini
[Interface]
PrivateKey = <Client_Private_Key>
Address = 10.0.0.2/32
DNS = 8.8.8.8

[Peer]
PublicKey = <Server_Public_Key>
Endpoint = 127.0.0.1:6666
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

- **Endpoint**: Set to `127.0.0.1:6666`, the local udp2raw client's listening address.
- **AllowedIPs**: Set to `0.0.0.0/0` to route all traffic through the VPN, or specify desired subnets.

## Conclusion

By wrapping WireGuard traffic with udp2raw, you can bypass network restrictions that prevent standard UDP traffic. This setup encapsulates your VPN traffic in a way that appears as regular TCP, UDP or ICMP packets, enhancing connectivity in restrictive environments.

## References

- [udp2raw GitHub Repository](https://github.com/wangyu-/udp2raw)
- [WireGuard Official Website](https://www.wireguard.com/)
