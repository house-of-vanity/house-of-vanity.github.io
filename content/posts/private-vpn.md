+++
title = "Private WireGuard telegram bot"
date = "2023-08-25"
description = "Your own telegram bot for managing WireGuard peers"

[taxonomies]
tags = ["linux", "torrent", "network", "selfhosting", "wireguard", "vpn"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

# Wireguard-Peer-Manager
![image](https://user-images.githubusercontent.com/4666566/117325184-56f7f800-ae45-11eb-9003-b85aadbf5ff0.png)

That bot can add Wireguard peers to config, reload it and send client config back via Telegram. 

<mark>**FYI: That tool stores client private keys into server config as comments.**</mark>

How to use:

```ini
# create initial wg config or use your own.
# P.S. Keep in mind that WPM can't manage peers created manually
# due to absence of client private key.

export CONFIG=$(cat <<-END
[Interface]
Address = 10.150.200.1/24
ListenPort = 51820
PrivateKey = $(wg genkey)
PostUp = iptables -A FORWARD -i %i -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -o %i -j ACCEPT
SaveConfig = false
END
)

sudo -E bash -c 'cat > /etc/wireguard/private.conf << EOF
${CONFIG}
EOF
'

cd /etc/wiregurad
sudo git clone https://github.com/house-of-vanity/Wireguard-Peer-Manager wpm
cd wpm

# install python and system requirements.
apt install qrencode python3-pip
pip3 install -r requirements.txt

# Create config
cp wpm_example.conf wpm.conf

# CLI usage. Client configs saved into `clients/peer_name.{conf,-qr.png,-qr.txt}`
python3 gen.py --peer my-pc   # add a new peer `my-pc`
python3 gen.py --delete my-pc # delete peer `my-pc`
python3 gen.py --update       # just regenerate all configs in `clients/`
python3 gen.py --json         # show WG status in JSON

# Telegram bot usage
TG_TOKEN=1292121488:AAG... TG_ADMIN=<comma separated list of usernames> python3 bot.py

```

## Config
Key | Default | Description
------------ | ------------- | ------------
allowed_ips | 0.0.0.0 | allowed_ips for generated peer configs.
dns | 8.8.8.8 | DNS for peer configs
hostname | $(hostname -f):51820 | server address for peer configs. May be an IP.
config | wg0 | WireGuard config to work with. 


## Telegram Interface

<img src="https://user-images.githubusercontent.com/4666566/117370133-cc31f000-ae7a-11eb-93fd-a390d2616da8.png" alt="drawing" width="450"/> <img src="https://user-images.githubusercontent.com/4666566/117377076-48323500-ae87-11eb-9602-a0cd3072ff53.png" alt="drawing" width="350"/>


