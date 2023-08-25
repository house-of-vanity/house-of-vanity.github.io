+++
title = "KVM/QEMU self hosted hypervisor"
date = "2020-07-14"
description = "Installing home hypervisor with remote control"

[taxonomies]
tags = ["linux", "kvm", "selfhosting"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

## Requirements
* Ubuntu Linux server (tested on 18.04 and 20.04)
* CPU with virtualisation enabled
---

## Installing

Installing VT staff

```sh
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils
```
I'd like to assign IPs for my VMs in the same network as server.

Here is `netplan` config:
```yaml
# /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    enp2s0f0:
      dhcp4: false
      dhcp6: false
  bridges:
    br0:
      interfaces: [enp2s0f0]
      dhcp4: true
      dhcp6: true
  version: 2
```

Generate and apply network config:
```sh
sudo netplan generate
sudo netplan --debug apply

# Check bridge
sudo networkctl
IDX LINK       TYPE     OPERATIONAL SETUP
  1 lo         loopback carrier     unmanaged
  2 enp2s0f0   ether    enslaved    configured
  3 br0        bridge   routable    configured
  4 virbr0     bridge   no-carrier  unmanaged
  5 virbr0-nic ether    off         unmanaged

# Check DHCP lease on new bridge
sudo ip a 
2: enp2s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether xxx brd ff:ff:ff:ff:ff:ff
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether xxx brd ff:ff:ff:ff:ff:ff
    inet 192.168.88.28/24 brd 192.168.88.255 scope global dynamic br0
       valid_lft 535sec preferred_lft 535sec
```
---

## Managing VMs

Grant permissions to use virtmanager to your user on server:
```sh
sudo adduser $USER libvirt-qemu
sudo adduser $USER libvirt
```

Use virt-manager GUI utility on client or virsh CLI tool for managing VMs and data pools.
