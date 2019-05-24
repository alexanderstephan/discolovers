---
layout: post
title:  "Getting started with Arch Linux"
date:   2018-10-22 15:07:19
categories: [tutorial]
comments: true
share: true
---

## Warranty Disclaimer:

I am not responsible for data loss, possible hardware failures, thermonuclear war, or you getting fired because your arch driven atomic plant failed.

If you have any concerns regarding the installation don’t hesitate to do some research on your own to prevent possible failures or bugs.

After booting the arch iso you’ll will have to proceed with the following steps:


## Prerequisits:
- A working [Arch Linux](https://www.archlinux.org/download/) boot stick

``` shell
sudo dd if=<path-to-image.iso> of=/dev/<name-of-usb> bs=4M status=progress
```
- A BIOS that is capable of UEFI


## Step 1: Prepare root file system


Temporarily load your preferred keyboard layout.

``` shell
loadkeys de
```

Find out your block device name (`-Sp` makes it easier to recognize the drive).

``` shell
lsblk -Sp
```

For quick formatting use `parted`, otherwise you can proceed using `fdisk` or `gdisk`.

### Option 1:

``` shell
parted
mkpart primary fat32 1MiB 261MiBi # Create EFI system partition
set 1 esp on
mkpart primary ext4 261MiB 100% # Use the rest as ext4
```

So this command creates a primary GPT partition on your device and creates a 260MiB boot partition at the start sector that will later become the EFI partiton for UEFI boot. Then we use the rest of the drive for installing the system. Of course you can tweak these commands according to your specific needs.

### Option 2:

Type in `fdisk` and navigate your way through the dialogs.

Now you can format both partitions you just created as you desire.

``` shell
mkfs.vfat /dev/<devicename>1
mkfs.ext4 /dev/<devicename>2
```
Mount devices so we can install the base root system on our drive

``` shell
mkdir /mnt/boot/
mount /dev/<devicename>2 /mnt
mount /dev/<devicename>1 /mnt/boot/
```
Install base system including developer tools and patches for Intel processors on target device.

``` shell
pacstrap /mnt base base-devel intel-ucode
```

Generate `fstab` (contains all drives that will be mounted during booting).

``` shell
genfstab -p /mnt > /mnt/etc/fstab
```

## Step 2: Setup the new root system

``` shell
arch-chroot /mnt
```

Now, as we moved to our new system, we can proceed to set all the basic parameters and settings. A good start is to set the host name:

``` shell
echo myhost > /etc/hostname
```

Now we want to set our language. We do that by writing the LANG	variable to `/etc/locale.conf`.

``` shell
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

Afterwards we want to uncomment the disired entry in `/etc/locale.gen` with the editor of your choice, so we can generate it in the next step with the following command:

``` shell
locale-gen
```

Now we can set our [keyboard layout](https://wiki.archlinux.org/index.php/Linux_console/Keyboard_configuration). You can check your current layout with `localectl status` and to print all options use `localectl list-keymaps`.

``` shell
echo KEYMAP=de-latin1 > /etc/vconsole.conf
```

In the next step we should set our [time zone](https://wiki.archlinux.org/index.php/System_time#Time_zone). To check out the current time zone use `timedatectl status`. To list all time zones use `timedatectl list-timezones`. Later we can install an NTP client that automatically syncs the clock over the network.

``` shell
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
```

It's also a good idea to rank the mirros, so we can have faster download speeds when installing software. You can either manually sort them or use [Reflector](https://wiki.archlinux.org/index.php/Reflector). This command for example sorts the 5 most recent mirrors by download speed. If you want only mirrors for your specific country use `-- country`.

``` shell
reflector --verbose --latest 50 --sort rate --protocol https --save /etc/pacman.d/mirrorlist
```

To continously refresh the mirrorlist we can create the file `/etc/pacman.d/hooks/mirrorupgrade.hook`.

```
[Trigger]
Operation = Upgrade
Type = Package
Target = pacman-mirrorlist

[Action]
Description = Updating pacman-mirrorlist with reflector and removing pacnew...
When = PostTransaction
Depends = reflector
Exec = /bin/sh -c "reflector --country 'United Germany' --latest 50 --age 24 --sort rate --save /etc/pacman.d/mirrorlist; rm -f /etc/pacman.d/mirrorlist.pacnew"
```

Now we should set up our local domains, so we can things like `localhost`.

``` shell
reflector --verbose --latest 50 --sort rate --protocol https --save /etc/pacman.d/mirrorlist
```

Also for compabilities sake it's a good idea to enable 32-bit libraries. Do that by uncommenting  the`[multilib]` entry in `/etc/pacman.conf`.
To refresh the mirrorlist use `pacman -Sy`.

A root password is always a good idea. Create one by using `passwd`.

Create an initramfs that loads programs very early in the `userspace`.

``` shell
mkinitcpio -p linux
```
Now we need to install a boot loader. If you use UEFI you can do the following after installing it with `pacman -S grub`.

``` shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

Now we can create a non-root user and add him to any group for example `wheel` and `audio`:

``` shell
useradd -m -g users -s /bin/bash alex
gpasswd -a alex wheel audio
```

Automatically set the system time:

``` shell
systemctl enable --now systemd-timesyncd.service
```
## Step 3: Moving to a graphical envirement

Now, as we have a layed down a solid foundation for our system we can move on by installing a window manager or a desktop environment to finally feel at home. For better performance install a sufficient driver. The open source drivers are `xf86-video-intel` for Intel, `xf86-video-amdgpu` for AMD and finally `xf86-video-nouveau` for Nvidia users.

If you don't rely on any X depenend software, I'd recommend you to give [sway](https://swaywm.org/) a try. Installing it via pacman will also install all important dependencies.

``` shell
sudo pacman -S sway dmenu rxvt-unicode xorg-server-xwayland
```

You can start sway by adding it to the end of your `~/.bashrc`.

For further customization please refer to my [dotfiles repo](https://github.com/alexanderstephan/dotfiles).

I hope this helped!


