# microvms

QEMU has the ability to boot in "microvm" mode, meaning, and I'm sort of guessing here, that it emulates less hardware, and it boots directly into a Linux kernel, skipping all that nasty bootloader stuff, and saving memory.
Similar to containers, microvm images do not need a kernel installed, and they cannot change the loaded drivers (XXX well maybe they could with fragile hacks involving keeping the in-VM copy of /lib/modules in sync with the running kernel version).

Combined with Copy-on-Write disk images, this (should) let QEMU scale to thousands of VMs on a single host.

## Setup

I've implemented this on Arch. It should be portable to other distros I just haven't tried that. Let me know if you port it!

## Ingredients

* `qemu-system-x86`
* `qemu-img`
* hardware accelerated VMs enabled: you should have /dev/kvm on the host
* `arch-install-scripts` (the equiv for debian is `debbootstrap`, I think; idk about other distros)
* `e2fsprogs`


## Bootstrap


### Kernel

Edit /etc/mkinitcpio.d/linux.preset to add:

```diff
-PRESETS=('default' 'fallback')
+PRESETS=('default' 'fallback' 'microvm')
+
+microvm_config="/etc/mkinitcpio-microvm.conf"
+microvm_image="/boot/initramfs-microvm.img"
+
```


> [!NOTE]
> (unfortunately there's no drop in files for the contents of these .preset files,you have to manage them by hand)

and install the minimal config in this repo:
```
sudo cp mkinitcpio-microvm.conf /etc/mkinitcpio-microvm.conf
```

This means that on every kernel upgrade, you will updated

* /boot/vmlinuz-linux (the default)
* /boot/initramfs-microvm.img (the addition)

Go ahead and bootstrap this:

```
sudo mkinitcpio -P linux
```

> [!NOTE]
> If you use and/or want your guest VMs to use a custom kernel package like linux-lts or linux-rt, edit /etc/mkinitcpio.d/linux-rt.preset instead

### Userland

Bootstrap a disk image with the Arch userland installed.

Create and open the disk image:

```
sudo modprobe nbd
qemu-img create -f qcow2 base.qcow2 500G # tune 500G as you like; this is an **upper** limit; only disk actually allocated by users will end up used
sudo qemu-nbd -c /dev/nbd0 base.qcow2
sudo mkfs.ext4 /dev/nbd0
sudo mount /dev/nbd0 /mnt
```

or if you don't want to use exotic drivers:

```
qemu-img create base.img 500G
sudo mkfs.ext4 base.img
sudo mount -o loop base.img /mnt
```

Either way you now have an empty disk at /mnt. Bootstrap the userland into it

```
sudo pacstrap -K /mnt base

# Set the root password on first login
sudo arch-chroot /mnt passwd -de root

# Trigger "first boot" behaviour #XXX this has to be done *after* the first boot happens, if you do a first boot
sudo arch-chroot /mnt/ truncate -s0 /etc/machine-id

# Make sure networking works
sudo systemctl --root=/mnt enable systemd-networkd systemd-resolved
sudo ln -sf ../run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
sudo tee /mnt/etc/systemd/network/20-wired.network <<EOF
[Match]
Name=eth*

[Network]
DHCP=yes
EOF

# Clean up nested gpg-agent; this is a result of using pacstrap -K: the chroot spawns its own keyring from scratch. Stupid GPG!
sudo gpgconf --homedir /mnt/etc/pacman.d/gnupg --kill all
````

Detach the disk

```
sudo umount /mnt
sudo qemu-nbd -d /dev/nbd0
```

If you used `base.img` above, you should convert it to qcow2 now:

```
qemu-img convert -o qcow2 base.img base.qcow2 && rm base.img
```

Protect the base image:
```
chmod ugo-w base.qcow2
```

At this point this system is extremely super minimal. It doesn't even have a text editor. But it can boot.


## Launch!

The best reason for using qcow2 is so that we can spawn cheap copies from a golden base image. `microvm-spawn` is a short helper to help you do this.

Boot into a child instance with:
```
microvm-spawn $(uuidgen)
```

This will make a new, small, disk image in ${some_uuid}/user.qcow2 that's backed by base.qcow2 and boot into it. Anything you do is recorded in user.qcow2 without touching base.qcow2.

You should end up, after a few seconds, at a Arch console login prompt.
Login as 'root' and you should be prompted to set the root password.

## Customizations

To do customizations -- like installing a text editor, enabling ssh, adding config files, etc -- you can either:

1. Make changes in `arch-chroot` to `base.qcow2` **before** write-protecting it and **before** spawning any child instances or...
2. Add an intermediate layer, which is described next.

Say you want to make an image to run Selenium (in CI or whatever):

```
qemu-img create -f qcow2 -b $(pwd)/base.qcow2 -B qcow2 ./selenium.qcow2
```

Boot in:
```
./microvm selenium.qcow2
```

while booted, install packages, add config files, whatever you need.

**When you're done**, but **before you shut it down** you should **reset the system to a blank slate** so that child instances see themselves as first booting. Unfortunately I don't know if there is an exhaustive or aesy list of what counts as "first boot"; it probably depends on the packages you have installed; I can recommend at least:

```
guest# passwd -de root # OR passwd -l root, depending on how you want to lock things down
guest# truncate -s0 /etc/machine-id # truncate, not rm, for this one
guest# rm /etc/ssh/ssh_host_*
guest# rm /etc/hostname
guest# rm /var/lib/systemd/random-seed
````

Finally, shut it down:

```
guest# poweroff
```


Write-protect it:

```
chmod ugo-w selenium.qcow2
```

Then, spawn instances with:

```
MICROVM_BASE=selenium.qcow2 microvm-spawn $(uuidgen)
```

## Kernel upgrades

Like a container, the image does not contain a kernel or kernel modules.
(well you could `pacman -S linux` but it wouldn't do anything).

Instead, kernel upgrades happen **on the host**. `pacman -Syu` updates /boot/vmlinuz-linux and /boot/initramfs-microvm.img and the next time a VM is started or rebooted it will pick up the new files and boot from them. In that sense, all microvms on a host share a kernel -- though not a running kernel, and they only share it once they're all restarted to pick up the new one.

> [!WARNING]
> That rebooting across an upgrade on the host also upgrades the guest VM is **unverified**. It's just sort of tricky to test, I have to notice that a kernel upgrade is coming and have a long-running microvm active, ready to test.


## Related Work

- [AWS's Firecracker](https://firecracker-microvm.github.io/)
- OpenBSD's `vmd(8)`; see [`vmctl start -b path/to/kernel`](https://man.openbsd.org/vmctl#b~2)
- [Kata containers](https://katacontainers.io/) -- behave like containers, implemented as microVMs

Commercial implementations:

- Amazon lambda
- [Boat](https://boat.dev)
