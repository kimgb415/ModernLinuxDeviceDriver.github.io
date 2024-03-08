---
layout: post
title:  "Chapter 4. Setup QEMU vm as a Linux sandbox"
date:   2024-03-06 18:32:15 +0900
categories: Chapter 4
---

## Why bother?

The lesson I have learned throughout the scull device driver implementation is that linux kernel memory is vulnerable to the malfunction of device driver, invalid memory access of device driver code can lead to undefined behavior of kernel. Should kernel become unresponsive any more, rebooting the entire machine is so cumbersome that it slows down the development iteration.

Though UML(User Mode Linux) is mentioned in the book, I found QEMU more popular and developer friendly to setup a sandbox for the linux development. With QEMU setup properly, the whole development environment will look as follows.

## Create QEMU vm

### Install necessary packages

> FYI, following setup assumes ubuntu / debian environment.

Firstly, we have to install packages related to QEMU, samba (shared filesystem between host and guest).

{%- highlight shell -%}
sudo apt install qemu-system qemu-user-static samba smbclient ssh
{%- endhighlight -%}

### Create disk image for vm

Next step is to create the disk image for our vm, which is really simple as below.

{%- highlight shell -%}
# qemu-img create -f {format of your disk image} {name of my disk image file} {size of disk image}
qemu-img create my-qemu-img 15G
{%- endhighlight -%}

There are many formats of disk images can be created through `qemu-img`, characteristics of each format can be found in [qemu disk image documentation](https://qemu-project.gitlab.io/qemu/system/images.html). The default image format is `qcow2`, which provides balance between performance and features, like encryption and snapshots.

### Prepare Linux distro

The Linux distro I used for my qemu vm is ubuntu server, which can be download in [ubuntu website](https://ubuntu.com/download/server). The reason I chose ubuntu server is that normal ubuntu is way too heavy for a vm, while arch linux is cumbersome to get the distro installed, however, the ubuntu server distro is lightweight and beginner-friendly.

### Set vm up

With disk image and distro in our hand, we are ready to set up the qemu vm.

{%- highlight shell -%}
qemu-system-x86_64 -enable-kvm -cdrom /path/to/downloaded/ubuntu-22.04.4-live-server-amd64.iso -boot order=c -drive file=my-qemu-img -m 4G
{%- endhighlight -%}

- by specifying `-enable-kvm`, the host can leverage the kvm module to enhance the performance of virtualization.
- `-cdrom /path/to/downloaded/ubuntu-22.04.4-live-server-amd64.iso` tells qemu to attach the installed linux server distro with a virtual cdrom, which will behave as a bootable usb
- the last `c` in `-boot order=c` means the qemu vm will prioritize the boot up from the cdrom, which is the linux server distro we have specified in the front.
- `-m 4G`, as you might have already noticed, specifies the amount of memory for our vm

### SSH into VM

Though it is not really necessary, we don't want to switch back and forth between the currently working terminal and qemu gui, hence it would be better if we can ssh into the guest vm from the host. The simplest way to ssh into the guest is through port forwarding, details in [qemu Networking](https://wiki.qemu.org/Documentation/Networking).

{%- highlight shell -%}
# boot up the guest, configuring the port forwarding between host tcp port 12345 and guest port 22
qemu-system-x86_64 -nic user,hostfwd=tcp::12345-:22 -enable-kvm -m 4G -smp cores=4,cpus=4 my-qemu-img

# ssh into guest from host through port 12345
ssh {your-id}@localhost -p 12345
{%- endhighlight -%}

> `-device e1000,netdev=net0` option is crucial to virtualize the network device, however, it is configured by default for x86 qemu vms, hence not specified above.


## Share folder between host and guest