+++
title = "Run arm64 VM on amd64"
date = "2024-10-12"
description = "Simple way to test arm64 workflow on amd64"

[taxonomies]
tags = ["linux", "virtualization", "arm64", "qemu"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

## Install QEMU

```sh
sudo apt install qemu-system-arm
```

## Create necessary support files

Next, create a VM-specific flash volume for storing NVRAM variables, which are necessary when booting EFI firmware:

```sh
truncate -s 64m varstore.img
truncate -s 64m efi.img
dd if=/usr/share/qemu-efi-aarch64/QEMU_EFI.fd of=efi.img conv=notrunc
```

## Fetch the Ubuntu cloud image

You need to fetch the ARM64 variant of the Ubuntu cloud image you would like to use in the virtual machine. You can go to the official [Ubuntu cloud image website](https://cloud-images.ubuntu.com/), select the Ubuntu release, and then download the variant whose filename ends in -arm64.img. For example, if you want to use the latest Jammy cloud image, you should download the file named jammy-server-cloudimg-arm64.img.

## Run QEMU natively on an ARM64 host

If you have access to an ARM64 host, you should be able to create and launch an ARM64 virtual machine there. Note that the command below assumes that you have already set up a network bridge to be used by the virtual machine.

```sh
sudo qemu-system-aarch64 \
 -enable-kvm \
 -m 1024 \
 -cpu host \
 -M virt \
 -nographic \
 -drive if=pflash,format=raw,file=efi.img,readonly=on \
 -drive if=pflash,format=raw,file=varstore.img \
 -drive if=none,file=jammy-server-cloudimg-arm64.img,id=hd0 \
 -device virtio-blk-device,drive=hd0 -netdev type=tap,id=net0 \
 -device virtio-net-device,netdev=net0
```

## Run an emulated ARM64 VM on x86

You can also emulate an ARM64 virtual machine on an x86 host. To do that:

```sh
sudo qemu-system-aarch64 \
 -m 2048 \
 -cpu max \
 -M virt \
 -nographic \
 -drive if=pflash,format=raw,file=efi.img,readonly=on \
 -drive if=pflash,format=raw,file=varstore.img \
 -drive if=none,file=jammy-server-cloudimg-arm64.img,id=hd0 \
 -device virtio-blk-device,drive=hd0 \
 -netdev type=tap,id=net0 \
 -device virtio-net-device,netdev=net0
```

## To set default password for image

```sh
sudo apt-get install cloud-image-utils

cat >user-data <<EOF
#cloud-config
password: ubuntu
chpasswd: { expire: False }
ssh_pwauth: True
EOF

cloud-localds user-data.img user-data

# user-data.img MUST come after the rootfs. 
sudo qemu-system-aarch64 \
 -m 2048 \
 -cpu max \
 -M virt \
 -nographic \
 -drive if=pflash,format=raw,file=efi.img,readonly=on \
 -drive if=pflash,format=raw,file=varstore.img \
 -drive if=none,file=jammy-server-cloudimg-arm64.img,id=hd0 \
 -drive file=user-data.img,format=raw \ 
 -device virtio-blk-device,drive=hd0 \
 -netdev type=tap,id=net0 \
 -device virtio-net-device,netdev=net0


```
