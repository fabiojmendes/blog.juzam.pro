+++
title = 'LXD: Rocky Linux image gets stuck during boot'
draft = true
date = '2025-08-29T17:24:28-04:00'
tags = []
[cover]
image = "cover.svg"
+++

I've been through a few iterations on which tool to use to manage the VMs on my
homelab. VirtualBox, Vagrant, Proxmox, you name it. Even a cobbled together
solution with Libvirt, qcow2 images and a
[Makefile](https://github.com/fabiojmendes/shell-goodies/blob/master/libvirt/config/Makefile).

Over the years I honed in on some requirements that made sense for my use cases:

- Integrate well with my workflow, i.e. has to have a good CLI
- Support name resolution
- Easy to download images
- Good support for cloud-config
- Support for OpenZFS

With my previous attempts, something was always missing or wasn't fully
supported. LXD turned out to check all these boxes for me. I'm not a huge fan of
the snap installation, and after the hug pull I'll migrate to
[Incus](https://linuxcontainers.org/incus/) when time allows. But I am pretty
happy with the tool itself.

I use LXD mostly to manage virtual machines. Personally I don't have much use
for the OS level containers. That's because most of the services that I run on
my homelab are application containers running with podman. I spin up a VM
whenever I want to experiment with something new and I want the full isolation
it provides. So I play a lot with different distros like Ubuntu, Debian, Fedora,
Rocky, Alma, CentOS and so on.

## VMs hanging during boot

But what's the fun in technology without an itch to scratch. I noticed an
interesting issue that a few of those distros sometimes would get stuck during
the boot-up process. I would see a message like this on the `lxc console`:

```text
[  *** ] A start job is running for /dev/loop4p2 (1w 15h 22min 46s / no limit)
```

In particular, that would only happen with Red Hat based distros like Fedora or
Rocky Linux. As you can see, this VM has been stuck for more than a week on this
process. Yes, I hear you. I should have better monitoring for my homelab. But
like I said, these are usually ephemeral boxes that I use for testing.

After digging a little, I found that some people were having a similar
[issue](https://discuss.linuxcontainers.org/t/couldnt-boot-up-rocky-linux-9-vm/17185)
with brand new VMs, and it had to do with how the images were created. That was
not my case, these VMs were working fine for a while, and then all of a sudden
this would happen.

But looking more carefully at that forum post, the very
[last message](https://discuss.linuxcontainers.org/t/couldnt-boot-up-rocky-linux-9-vm/17185/24)
states that after a "yum update" the user would experience the same behavior.
Experimenting with a few installations, I was able to zero in on the root cause.
When there's a Kernel update, the generated
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

```text {hl_lines=[5]}
title Rocky Linux (5.14.0-570.25.1.el9_6.x86_64) 9.6 (Blue Onyx)
version 5.14.0-570.25.1.el9_6.x86_64
linux /boot/vmlinuz-5.14.0-570.25.1.el9_6.x86_64
initrd /boot/initramfs-5.14.0-570.25.1.el9_6.x86_64.img
options root=UUID=3f7470e9-ff2f-448f-8bd7-97f573f9b597 ro console=tty1 console=ttyS0
grub_users $grub_users
grub_arg --unrestricted
grub_class rocky
```

The fix is quite simple, you need to replace the `root=/dev/loopXpY` property in
the `options` section with the correct UUID for your root device. In my case
this would be:

```text
options root=UUID=3f7470e9-ff2f-448f-8bd7-97f573f9b597 ro console=tty1 console=ttyS0
```

You can use an existing working entry, usually the previous kernel before the
update, as a reference to update the value.

## Mounting a ZVOL on the host system

Now the tricky part is, how to update the value if the OS wont boot. If you are
using LXD with a OpenZFS storage pool, you have to mount the VMs ZVOL to make
those changes. To that end, you have to follow a few steps:

Set the `volmode` to `full` so it can be read and mounted by the host system:

```shell
zfs set volmode=full tank/lxd/virtual-machines/rocky.block
```

Mount the new device to a working folder:

```shell
mount /dev/zvol/tank/lxd/virtual-machines/rocky.block-part2 /mnt
```

Edit the entry file and update the root property:

```shell
vi /mnt/boot/loader/entries/a53795788afc466894e149f71508881b-5.14.0-570.26.1.el9_6.x86_64.conf
```

Unmount the filesystem and cleanup all changes:

```shell
umount /mnt
zfs set volmode=none tank/lxd/virtual-machines/rocky.block
```

That's it, you should be able to boot the VM as usual now:

```shell
lxc start rocky
```

This is a quick workaround to get you up and running again. Ideally, the script
that generates the entry file would do the right thing, but I'm not sure where
this is configured. I'll update this post in the future if I find a definitive
solution.
