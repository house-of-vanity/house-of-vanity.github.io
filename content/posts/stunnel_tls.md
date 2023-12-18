+++
title = "Expose service via TLS stunnel"
date = "2023-12-18"
description = "How to expose any TCP application securely via TLS tunnel"

[taxonomies]
tags = ["linux", "tools", "selfhosting"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

First of all, I encountered an issue with the Outline VPN server, which exposes Prometheus metrics on 127.0.0.1 with no option to change it. As a solution, I used stunnel4. Essentially, it works as a TLS proxy, listening on a configured port and forwarding traffic to another.

[Server1 (stunnel server)] <==> [Server2 (stunnel client)]

## Server side
Install stunnel and create configs:
```shell
ab@cy:/etc/stunnel$ cat outline_prom.conf
debug = 5
output = /var/log/stunnel.log

[outline_prom]
accept = 0.0.0.0:9095
connect = 127.0.0.1:9092
PSKsecrets = /etc/stunnel/psk.txt
```

`psk.txt` is a credentials file and looks like:
```shell
# I used `openssl rand -hex 32` to generate secret
ab@cy:/etc/stunnel$ cat psk.txt
user:secret_string
```

## Client side
`psk.txt` the same and config looks like:
```shell
ab@home:/etc/stunnel$ cat /etc/stunnel/outline_prom.conf
[outline_prom_cy]
client = yes
accept = 0.0.0.0:9095
connect = cy.hexor.cy:9095
PSKsecrets = /etc/stunnel/psk.txt
```
---
