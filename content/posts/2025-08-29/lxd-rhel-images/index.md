+++
title = 'LXD: Rocky Linux VM gets stuck during boot'
date = '2025-08-29T17:24:28-04:00'
tags = ["LXD", "RockyLinux", "Fedora", "OpenZFS"]
summary = """
RHEL-based LXD VMs (Rocky, Fedora) can hang at boot after kernel updates.
This post explains the root cause and shows how to fix the boot entry, and get the VM booting again.
"""
[cover]
image = "cover.svg"
+++

> Update 2025-09-05: Added a section for fixes for Fedora and Rocky Linux

I've gone through a few iterations of tools to manage the VMs in my homelab:
VirtualBox, Vagrant, Proxmox—you name it. Even a cobbled‑together solution with
Libvirt, qcow2 images, and a
[Makefile](https://github.com/fabiojmendes/shell-goodies/blob/master/libvirt/config/Makefile).

Over the years, I've homed in on a few requirements that make sense for my use
cases:

- Integrates well with my workflow (i.e., has a good CLI)
- Supports name resolution
- Makes downloading images easy
- Good support for cloud-config
- Supports OpenZFS

With my previous attempts, something was always missing or wasn't fully
supported. LXD checks all these boxes for me. I'm not a huge fan of the Snap
installation, and after
[Canonical’s takeover](https://linuxcontainers.org/lxd/), I'll migrate to
[Incus](https://linuxcontainers.org/incus/) when time allows. But I am pretty
happy with the tool itself.

I use LXD mostly to manage virtual machines. I don't have much use for OS‑level
containers because most of the services that I run on my homelab are application
containers running with Podman. I spin up a VM whenever I want to experiment
with something new and need the full isolation it provides. So I play around
with different distros like Ubuntu, Debian, Fedora, Rocky, Alma, and CentOS.

## VMs hanging during boot

But what's the fun in technology without an itch to scratch? I noticed an issue
where some of those distros would sometimes get stuck during boot. I would see a
message like this in the `lxc console`:

```text
[  *** ] A start job is running for /dev/loop4p2 (1w 15h 22min 46s / no limit)
```

In particular, this happens with Red Hat–based distros like Fedora and Rocky
Linux. As you can see, this VM has been stuck on this job for more than a week.
Yes, I hear you—I should have better monitoring for my homelab. But like I said,
these are usually ephemeral boxes that I use for testing.

After digging around a bit, I found that some people were having a similar
[issue](https://discuss.linuxcontainers.org/t/couldnt-boot-up-rocky-linux-9-vm/17185)
with brand‑new VMs, and it had to do with how the images were created. That
wasn't my case—these VMs were working fine for a while, and then, all of a
sudden, this would happen.

Looking more closely at that forum post, the
[last message](https://discuss.linuxcontainers.org/t/couldnt-boot-up-rocky-linux-9-vm/17185/24)
states that after running `yum update`, the user would experience the same
behavior. Experimenting with a few installations, I was able to zero in on the
root cause. When there's a kernel update, the generated
`/boot/loader/entries/<uuid>-<version>-<arch>.conf` looks a little like this:

```text {hl_lines=[5]}
title Rocky Linux (5.14.0-570.26.1.el9_6.x86_64) 9.6 (Blue Onyx)
version 5.14.0-570.26.1.el9_6.x86_64
linux /boot/vmlinuz-5.14.0-570.26.1.el9_6.x86_64
initrd /boot/initramfs-5.14.0-570.26.1.el9_6.x86_64.img
options root=/dev/loop4p2 ro console=tty1 console=ttyS0
grub_users $grub_users
grub_arg --unrestricted
grub_class rocky
```

Comparing it to the previous version, the error is clear:

```text {hl_lines=[5]}
title Rocky Linux (5.14.0-570.25.1.el9_6.x86_64) 9.6 (Blue Onyx)
version 5.14.0-570.25.1.el9_6.x86_64
linux /boot/vmlinuz-5.14.0-570.25.1.el9_6.x86_64
initrd /boot/initramfs-5.14.0-570.25.1.el9_6.x86_64.img
options root=UUID=11111111-2222-3333-4444-555555555555 ro console=tty1 console=ttyS0
grub_users $grub_users
grub_arg --unrestricted
grub_class rocky
```

The fix is quite simple: you need to replace the `root=/dev/loopXpY` parameter
in the `options` section with the correct UUID for your root device. In my case,
this would be:

```text
options root=UUID=11111111-2222-3333-4444-555555555555 ro console=tty1 console=ttyS0
```

You can use a working entry—usually the previous kernel's—as a reference to
update the value.

## Mounting a ZVOL on the host system

Here's the tricky part: how do you update the value if the OS won't boot? If you
are using LXD with an OpenZFS storage pool, you need to mount the ZVOL on the
host to make those changes. Follow these steps:

Make sure to stop the machine before executing these steps:

```shell
lxc stop rocky
```

Set the `volmode` to `full` so it can be read and mounted by the host system:

```shell
zfs set volmode=full tank/lxd/virtual-machines/rocky.block
```

Mount the device at a temporary mount point:

```shell
mount /dev/zvol/tank/lxd/virtual-machines/rocky.block-part2 /mnt
```

Edit the entry file and update the root parameter:

```shell
vi /mnt/boot/loader/entries/a53795788afc466894e149f71508881b-5.14.0-570.26.1.el9_6.x86_64.conf
```

Unmount the filesystem and clean up:

```shell
umount /mnt
zfs set volmode=none tank/lxd/virtual-machines/rocky.block
```

That's it—you should be able to boot the VM as usual now:

```shell
lxc start rocky
```

This is a quick workaround to get you up and running again. Ideally, whatever
generates the entry file would set the correct root parameter, but I'm not sure
where this is configured. I'll update this post in the future if I find a
definitive solution.

## Fix

After spending some time experimenting with kernel upgrades on both Fedora and
Rocky, I managed to come up with a solution.

First, determine if the prefix name of the configuration files under
`/boot/loader/entries` match the output of `systemd-machine-id-setup --print`. I
believe that due to the way the image gets generated the machine id gets reset
on first boot of the actual VM. That causes an unrelated issue where after the
kernel update, the machine boots on the older kernel.

Moving to the root filesystem issue, you have to determine the UUID of the
`rootfs` device using `blkid`:

```shell
blkid
```

Take note of the `UUID` value for the partition labeled as `rootfs`.

### Rocky

On rocky or similar distros you can use `grubby` to update the boot
configuration.

```shell
grubby --update-kernel ALL --args 'root=UUID=11111111-2222-3333-4444-555555555555'
```

### Fedora

Fedora doesn't ship with grubby by default, but the same can achieve the same by
updating `/etc/default/grub` and add the following key:

```text
# Set device UUID
GRUB_DEVICE_UUID="11111111-2222-3333-4444-555555555555"
```
