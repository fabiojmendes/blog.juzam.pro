+++
title = 'Lxd RHEL Images'
draft = true
date = '2025-08-29T17:24:28-04:00'
tags = []
[cover]
image = "cover.svg"
+++

I've been through a few iterations on which tool to use to manage the VMs on my
homelab. VirtualBox, Vagrant, Proxmox, you name it. Even a gobbled together
solution with Libvirt, qcow2 images and a
[Makefile](https://github.com/fabiojmendes/shell-goodies/blob/master/libvirt/config/Makefile).

Over the years I honed in on some requirements that made sense for my use cases:

- Integrate well on my workflow, i.e. has to have a good CLI
- Support name resolution
- Easy to download images
- Good support for cloud-config
- Support for OpenZFS

With my previous attempts, something was always missing or wasn't fully
supported. LXD turned out to check all these boxes for me. I'm not a huge fan of
the snap installation, and after the hug pull I'll migrate to
[Incus](https://linuxcontainers.org/incus/) when time allows. But I am pretty
happy with the tool itself.

I use LXD mostly to manage virtual machines. Personally I don't have much use of
the OS level containers. That because most of the services that I run on my
homelab are application containers running with podman. I spin up a VM whenever
I want to experiment with something new and I want the full isolation it
provides. So I play a lot with different distros like Ubuntu, Debian, Fedora,
Rocky, Alma, CentOS and so on.

## 

But what's the fun in technology without a itch to scratch. But I noticed an
interesting issue that a few of those distros sometimes would get stuck during
the boot-up process. I would see a message like this on the `lxc console`:

```text
[  *** ] A start job is running for /dev/loop4p2 (1w 15h 22min 46s / no limit)
```

In particular, that would only happen with Red Hat based distros like Fedora or
Rocky Linux. As you can see, this VM has been stuck for more than a week on this
process. Yes, I hear you. I should have better monitoring for my homelab. But
like I said, these are usually ephemeral boxes that I use for testing.

After digging a little, I found some that people were having a similar
[issue](https://discuss.linuxcontainers.org/t/couldnt-boot-up-rocky-linux-9-vm/17185)
with brand new VMs, and it had to do with how the images were created. That was
not my case, these VMs were working fine for a while, and them all of the sudden
this would happen.

But looking more carefully at that forum post, the very
[last message](https://discuss.linuxcontainers.org/t/couldnt-boot-up-rocky-linux-9-vm/17185/24)
states that after a "yum update"
