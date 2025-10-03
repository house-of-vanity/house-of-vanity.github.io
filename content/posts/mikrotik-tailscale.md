+++
title = "Tailscale on Mikrotik containers"
date = "2025-10-03"
description = "Easy way to connect Mikrotik and its networks to Tailscale network"

[taxonomies]
tags = ["linux", "tools", "networking", "containers"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

---

## Create a tag in Tailscale

Open **Visual ACLs → Tags** and create a tag (e.g. `tag:mikrotik-lms`). We’ll assign this tag via container args so your ACLs/policies apply to the node.

* Visual ACL Tags: [https://login.tailscale.com/admin/acls/visual/tags](https://login.tailscale.com/admin/acls/visual/tags)

---

## Create an OAuth client (auth_keys scope)

Create an OAuth client with the **`auth_keys`** scope. You’ll use the **client secret** to mint *auth keys* via API (do **not** put the OAuth secret directly into the container).

* OAuth Clients: [https://login.tailscale.com/admin/settings/oauth](https://login.tailscale.com/admin/settings/oauth)

---

## Storage on `usb1` (container root + state)

Put both the container’s root storage and Tailscale state on `usb1` to avoid filling the internal flash.

```bash
# Point RouterOS container system to usb1 for all image/layer data.
/container/config/set root-dir=/usb1/container tmpdir=/usb1/container/tmp
/file/make-dir usb1/container/tmp

# Create a persistent state dir for Tailscale on usb1.
/file/make-dir usb1/tailscale-state
```

> Make sure `usb1` is mounted and has enough free space. The destination path of the mount **must match** `TS_STATE_DIR` (`/var/lib/tailscale`).

---

## Environment variables (RouterOS env list)

Required: `TS_AUTHKEY`, `TS_STATE_DIR`, and `--advertise-tags` matching your ACL tag. Optional vars go after.

```bash
# --- REQUIRED ---
/container/envs/add name=TAILSCALE_VARS key=TS_AUTHKEY   value="tskey-REPLACE_ME" \
  comment="Headless auth key; do not use the raw OAuth secret"

/container/envs/add name=TAILSCALE_VARS key=TS_STATE_DIR value="/var/lib/tailscale" \
  comment="Persistent state dir inside container; must match the bind mount dst"

/container/envs/add name=TAILSCALE_VARS key=TS_EXTRA_ARGS \
  value="--advertise-tags=tag:mikrotik-lms" \
  comment="Attach the policy tag so ACLs apply"

# --- OPTIONAL ---
/container/envs/add name=TAILSCALE_VARS key=TS_ROUTES        value="10.0.5.0/24" \
  comment="Advertise routed subnet(s) if acting as a subnet router"

/container/envs/add name=TAILSCALE_VARS key=TS_ENABLE_METRICS value="true" \
  comment="Expose /debug/metrics for Prometheus"

/container/envs/add name=TAILSCALE_VARS key=TS_ACCEPT_DNS     value="true" \
  comment="Accept MagicDNS/control-plane DNS"

/container/envs/add name=TAILSCALE_VARS key=TS_USERSPACE      value="false" \
  comment="Kernel mode if TUN available; set true to force userspace"
```

---

## Create the mount and veth, then the container (pinned image)

Mount from `usb1` to **the same path** as `TS_STATE_DIR`. Create a `veth` for networking. Pin the image tag (don’t use `latest`).

```bash
# --- MOUNT ON usb1 (matches TS_STATE_DIR) ---
/container/mounts/add name=TS_state src=/usb1/tailscale-state dst=/var/lib/tailscale

# --- VETH FOR CONTAINER NETWORKING ---
/interface/veth/add name=veth-tailscaled address=172.31.0.2/24 gateway=172.31.0.1
/ip/address/add address=172.31.0.1/24 interface=veth-tailscaled

# --- CONTAINER (PINNED VERSION) ---
# Replace tag with a current one from Docker Hub.
/container/add name=tailscaled \
  remote-image=tailscale/tailscale:v1.76.0 \
  envlist=TAILSCALE_VARS \
  mounts=TS_state \
  interfaces=veth-tailscaled \
  start-on-boot=yes

/container/start tailscaled
```

<img width="50%" alt="image" src="https://github.com/user-attachments/assets/c70ff6b9-0f09-4021-aea7-78069cdbe194" />


Reference docs:

* ManagevACL Tags: [https://login.tailscale.com/admin/acls/visual/tags](https://login.tailscale.com/admin/acls/visual/tags)
* OAuth Clients: [https://login.tailscale.com/admin/settings/oauth](https://login.tailscale.com/admin/settings/oauth)
* All container variables: [https://tailscale.com/kb/1282/docker](https://tailscale.com/kb/1282/docker)
* Image tags: [https://hub.docker.com/r/tailscale/tailscale/tags](https://hub.docker.com/r/tailscale/tailscale/tags)

---
