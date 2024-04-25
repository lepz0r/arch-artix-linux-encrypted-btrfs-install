# Arch/Artix Linux Encrypted Btrfs Install

[Tutorial ini juga tersedia dalam Bahasa Indonesia](./README_id.md)

This is a guide to install Arch/Artix Linux with full disk encryption (including `/boot`), Btrfs, snapper and booting from snapshot.

## Preparing drive

### Partitioning

You can use `fdisk` or `cfdisk` to partition your drive, if you are using UEFI system also create an ESP partition. \
If you are dual booting just create a LUKS partition & jump to [Formatting Partitions](#formatting-partitions) but don't format the ESP.

<b>Example Partition for EFI</b>

| Partition | Size  | Type             | Notes                                                                                 |
| --------- | ----- | ---------------- | ------------------------------------------------------------------------------------- |
| ESP       | 200M  | EFI system       | If you already installed an OS on UEFI system this partition already has been created |
| LUKS      | >20GB | Linux filesystem |                                                                                       |

#### Formatting Partitions

Format ESP partition if you're using UEFI system, in this tutorial it's /dev/sda1. \
`# mkfs.vfat /dev/sda1`
<br/><br/>

Format LUKS partition, in this example we use `/dev/sda2` as our partition. \
`# cryptsetup luksFormat --pbkdf=pbkdf2 /dev/sda2`
<br/><br/>

Open LUKS partition, we use `btw` as the name of the LUKS & LVM partition. \
`# cryptsetup open /dev/sda2 btw`
<br/><br/>

#### Creating an LVM on LUKS partition & formatting volumes

We use LVM on LUKS so both root & swap is on the same LUKS partition. \
For swap volume create with the size of your RAM, in this tutorial it's 4GB.
<br/><br/>

Create LVM volume group (VG) on LUKS, in this case we also use btw as VG's name.

```
# vgcreate btw /dev/mapper/btw
```

<br/><br/>
Create root & swap logical volumes (LV) on VG.

```
# lvcreate -L 4G -n swap btw
# lvcreate -l +100%FREE -n root btw
```

<br/><br/>
Format LVs.

```
# mkswap /dev/btw/swap
# mkfs.btrfs /dev/btw/root
```

#### Creating btrfs subvolumes & mounting

Use swap volume as swap. \
`# swapon /dev/btw/swap`

<br/><br/>
Mount root volume. \
`# mount -o compress=zstd,noatime /dev/btw/root /mnt`

<br/><br/>
Create btrfs subvolumes.

```
# btrfs subvol create /mnt/@
# btrfs subvol create /mnt/@var
# btrfs subvol create /mnt/@home
# btrfs subvol create /mnt/@snapshots
```

| Subvolume    | Mountpoint     | Description                |
| ------------ | -------------- | -------------------------- |
| `@`          | `/`            | Root subvolume             |
| `@var`       | `/var`         | `/var` subvolume           |
| `@home`      | `/home`        | Home subvolume             |
| `@snapshots` | `/.snapshots/` | Root's subvolume snapshots |

<b>Optional subvolumes</b>
|Subvolume |Mountpoint |Description |
|- |- |- |
|`@srv` |`/srv` | |
|`@libvirt-image` |`/var/lib/libvirt/images/` |libvirt image pool |
|`@mysql` |`/var/lib/mysql` |MariaDB/MySQL database |

<br/><br/>
Disable Copy on write (COW) for the `@var` subvolume. \
`# chattr +C /mnt/@var`

<br/><br/>
Unmount btrfs volume & mount the root (`@`) subvolume.

```
# umount /mnt
# mount -o compress=zstd,noatime,subvol=@ /dev/btw/root /mnt
```

<br/><br/>
Mount rest of the subvolumes & ESP.

```
# mkdir -p /mnt/{boot/efi,var,home,.snapshots}
# mount /dev/sda1 /mnt/boot/efi
# mount -o compress=zstd,noatime,subvol=@home /dev/btw/root /mnt/home
# mount -o compress=zstd,noatime,subvol=@var /dev/btw/root /mnt/var
# mount -o compress=zstd,noatime,subvol=@snapshots /dev/btw/root /mnt/.snapshots
```

## Installation

### Installing base packages

(Optional, Arch only) Use reflector to generate mirrorlist or if you want to choose mirrors manually and/or you're installing Artix you can edit `/etc/pacman.d/mirrorlist`.

> Note: For the country you may change `--country` to your country and/or nearest countries but in this tutorial we use `--country Indonesia,Singapore` to include mirrors in Indonesia & Singapore as an example.

`# reflector --save /etc/pacman.d/mirrorlist --latest 20 --sort rate --verbose --protocol http,https --country Indonesia,Singapore  `

<br/><br/>
Install base packages.

- For Arch:

```
# pacstrap -K /mnt base linux linux-firmware lvm2 cryptsetup btrfs-progs snapper sudo
```

- For Artix:
  > Note: you can choose your init which are `openrc`,`runit`,`s6` or `dinit`, for s6 init package (after linux-firmware) use `s6-init` instead of `s6`.
  <pre>

# basestrap /mnt base linux linux-firmware <b>init</b> elogind-<b>init</b> lvm2-<b>init</b> cryptsetup snapper sudo

</pre>

### Configuring Arch Linux

#### Configuring fstab

Generate fstab.

- For Arch: \
  `# genfstab -U /mnt >> /mnt/etc/fstab`
- For Artix: \
  `# fstabgen -U /mnt >> /mnt/etc/fstab`

<br/><br/>
Edit fstab file.

> Note: you can use vim or nano to edit files, nano is the easiest to use but we use vim in this tutorial. Artix can use vi instead of vim as Artix Live ISO only include Vi instead of Vim.

`# vim /mnt/etc/fstab`

Remove all `subvolid=xxx` from the options for btrfs mounts.

> Tip: if you use vim you can use `%s/subvolid=[0-9][0-9][0-9],//g` to remove all subvolid options.

For example, `/mnt/etc/fstab` on my installation look like this:

```
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
# /dev/mapper/btw-root
UUID=bd4b4ae8-007b-41d2-8518-0a2414a6cf70       /               btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvol=/@     0 1

# /dev/sda1
UUID=F1FA-F389          /boot/efi       vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro   0 2

# /dev/mapper/btw-root
UUID=bd4b4ae8-007b-41d2-8518-0a2414a6cf70       /home           btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvol=/@home 0 2

# /dev/mapper/btw-root
UUID=bd4b4ae8-007b-41d2-8518-0a2414a6cf70       /var            btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvol=/@var  0 2

# /dev/mapper/btw-root
UUID=bd4b4ae8-007b-41d2-8518-0a2414a6cf70       /.snapshots     btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvol=/@snapshots    0 2

# /dev/mapper/btw-swap
UUID=6c55e2c4-09a6-4836-b67f-c0a25c460098       none            swap            defaults        0 0
```

#### Change pacman's database location

Move pacman's database directory from `/mnt/var/lib/pacman/` to `/mnt/usr/var/lib/pacman` so it will be included in snapshot.

```
# mkdir -p /mnt/usr/var/lib/
# mv /mnt/var/lib/pacman /mnt/usr/var/lib
```

Then edit `/mnt/etc/pacman.conf` and change `DBPath      = /var/lib/pacman/` to `DBPath      = /usr/var/lib/pacman/`.

#### Change root into the Arch Linux installation

- For Arch: \
  `# arch-chroot /mnt /bin/bash`
- For Artix: \
  `# artix-chroot /mnt /bin/bash`

#### Configuring Locale & Time Zone

Link `/etc/localtime` to `/usr/share/zoneinfo/<Region>/<City>`, in this tutorial we use Barnaul's time zone. \
`# ln -sf /usr/share/zoneinfo/Asia/Barnaul /etc/localtime`

<br/><br/>
Generate `/etc/adjtime` \
`# hwclock --systohc`

<br/><br/>
Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other necessary locales.

> Note: We didn't install an editor yet, to install editor use `pacman -S vim` for vim or `pacman -S nano` for nano.

<br/><br/>
Generate locale. \
`# locale-gen`

<br/><br/>
Create `/etc/locale.conf` to set locale systemwide. \
In this example `/etc/locale.conf` on my installation look like this:

```
LANG=en_US.UTF-8
```

#### Configure hostname

Change your hostname in this tutorial we use `i-use-arch-btw` as our hostname. \
`echo i-use-arch-btw > /etc/hostname`

#### Generate initramfs & generate second LUKS key

Generate second LUKS key to `/etc/lukskeys/root`.

```
# mkdir /etc/lukskeys
# chmod 700 /etc/lukskeys
# dd if=/dev/random of=/etc/lukskeys/root bs=1024 count=128
```

<br/><br/>
Add second key to LUKS partition. \
`# cryptsetup luksAddKey /dev/sda2 /etc/lukskeys/root`

<br/><br/>
Edit `/etc/mkinitcpio.conf` to add second key to initramfs & add encrypt, LVM2 & resume hook to initramfs.

- systemd initramfs hook (Arch only): \
  `HOOKS=(base systemd autodetect keyboard modconf block sd-encrypt lvm2 filesystems fsck)`
- udev initramfs hook (Arch & Artix): \
  `HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems resume fsck)`

And add second LUKS key to initramfs. \
`FILES=(/etc/lukskeys/root)`

<br/><br/>
(systemd initramfs) Edit `/etc/crypttab.initramfs` and add your LUKS partition.

> Note: you can get UUID of each partition/volume using `blkid <device>`.
>
> ```
> # blkid /dev/sda2
> /dev/sda2: UUID="cc1b673c-159d-4224-9af0-5465373b63a9" TYPE="crypto_LUKS" PARTUUID="408fb50d-bb3b-2344-b928-ebcee8d2cc61"
> # blkid /dev/btw/swap
> /dev/btw/swap: UUID="ed071d58-59cf-4c38-8fc4-3d775207c5c2" TYPE="swap"
> ```

<pre>
btw             UUID=<b>your-LUKS-partition-UUID</b> 	/etc/lukskeys/root
</pre>

Or if you're installing on SSD or VM you may want to enable trim support.

<pre>
btw             UUID=<b>your-LUKS-partition-UUID</b> 	/etc/lukskeys/root 	discard
</pre>

<br/><br/>
(udev initramfs) Configure GRUB to unlock the LUKS partition and change `GRUB_CMDLINE_LINUX_DEFAULT=`.

> Note: if you're using udev init you must install GRUB first, follow [Installing Chaotic-AUR repo](#installing-chaotic-aur-repo) & [Installing & configuring GRUB](#installing--configuring-grub) section first before configuring GRUB.

<pre>
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=<b>your-LUKS-partition-UUID</b>:btw cryptkey=rootfs:/etc/lukskeys/root resume=UUID=<b>your-swap-LV-UUID</b> loglevel=3 quiet"
</pre>

Or if you're installing on SSD or VM you may want to enable trim support.

<pre>
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=<b>your-LUKS-partition-UUID</b>:btw:allow-discards cryptkey=rootfs:/etc/lukskeys/root resume=UUID=<b>your-swap-LV-UUID</b> loglevel=3 quiet"
</pre>

<br/><br/>
Generate GRUB Config \
`# grub-mkconfig -o /boot/grub/grub.cfg`

<br/><br/>
Generate Initramfs. \
`# mkinitcpio -P`

#### Installing Chaotic-AUR repo

Since we use LUKS2 to encrypt our Arch Linux installation we need to install modified GRUB bootloader (`grub-improved-luks2-git`) which is on AUR, but we can install it from Chaotic-AUR repo which build some packages from AUR repository.

<br/><br/>
(Artix) Add universe repo by appending this repository list to `/etc/pacman.conf`:

```
[universe]
Server = https://universe.artixlinux.org/$arch
Server = https://mirror1.artixlinux.org/universe/$arch
Server = https://mirror.pascalpuffke.de/artix-universe/$arch
Server = https://artixlinux.qontinuum.space/artixlinux/universe/os/$arch
Server = https://mirror1.cl.netactuate.com/artix/universe/$arch
Server = https://ftp.crifo.org/artix-universe/
```

<br/><br/>
(Artix) Install Arch repository support.

```
# pacman -Syu artix-archlinux-support
```

<br/><br/>
(Artix) Add Arch repository by appending this repository list to `/etc/pacman.conf`:

```
# Arch Linux repositories.

#[testing]
#Include = /etc/pacman.d/mirrorlist-arch


[extra]
Include = /etc/pacman.d/mirrorlist-arch


#[community-testing]
#Include = /etc/pacman.d/mirrorlist-arch


[community]
Include = /etc/pacman.d/mirrorlist-arch


#[multilib-testing]
#Include = /etc/pacman.d/mirrorlist-arch


#[multilib]
#Include = /etc/pacman.d/mirrorlist-arch
```

<br/><br/>
Import Chaotic-AUR key & install Chaotic-AUR repository.

```
pacman-key --recv-key FBA220DFC880C036 --keyserver keyserver.ubuntu.com
pacman-key --lsign-key FBA220DFC880C036
pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-keyring.pkg.tar.zst' 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-mirrorlist.pkg.tar.zst'
```

<br/><br/>
Append Chaotic-AUR repository config to `/etc/pacman.conf`:

```
[chaotic-aur]
Include = /etc/pacman.d/chaotic-mirrorlist
```

#### Installing & configuring GRUB

Install `grub-improved-luks2-git`.\
`# pacman -Syu grub-improved-luks2-git`

<br/><br/>
(UEFI only) Install `efibootmgr`. \
`# pacman -S efibootmgr`

<br/><br/>
Edit `/etc/default/grub` & uncomment `GRUB_ENABLE_CRYPTODISK=y`

<br/><br/>
Install GRUB.

- For UEFI: `# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Arch Linux"`
- For BIOS: `# grub-install --target=i386-pc /dev/sda`

<br/><br/>
Generate GRUB Config \
`# grub-mkconfig -o /boot/grub/grub.cfg`

#### Creating new user & setup sudo

Edit sudoers file with `visudo` & uncomment `%wheel ALL=(ALL:ALL) ALL`.

> Note: If you're using nano you can use `EDITOR=nano visudo` instead.

<br/><br/>
Create new user, we create a new user called `cincinmasukmangkok` in this tutorial.

```
# useradd -mG wheel cincinmasukmangkok
# passwd cincinmasukmangkok
```

#### Configure Snapper

Create `root` Snapper config.

```
# umount /.snapshots
# rm -rf /.snapshots
# snapper --no-dbus -c root create-config /
# btrfs subvol delete /.snapshots
# mkdir /.snapshots
# mount /.snapshots
```

<br/><br/>
Edit `/etc/snapper/configs/root` and change `TIMELINE_CREATE="yes"` to `TIMELINE_CREATE="no"` so snapshots are only automatically created by pacman hook before and after installing or updating packages.

<br/><br/>
Install `snap-pac` & `grub-btrfs`.

```
# pacman -S snap-pac grub-btrfs
```

<br/><br/>
(Arch) enable GRUB btrfs service.

```
# systemctl enable grub-btrfs.path
```

<br/>

Or you can install grub-btrfs hook \
`# pacman -S snap-pac-grub`

### Installing snapper-rollback

`snapper-rollback` is on AUR but it's not on Chaotic-AUR when this tutorial is created, but we can use `paru` to install `snapper-rollback`.

```
# pacman -S base-devel paru
# sudo -u cincinmasukmangkok paru -S snapper-rollback
```

<br/><br/>
Edit `/etc/snapper-rollback.conf` and change `dev` to `dev = /dev/<VG>/<LV>`.

### Configure Network

#### systemd-networkd (Arch Linux only)

Find out your network interface name which is usually `enpXsY` or `ethX` for ethernet.

```
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:86:35:4f brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.105/24 brd 192.168.1.255 scope global dynamic noprefixroute enp1s0
       valid_lft 40615sec preferred_lft 35215sec
```

<br/><br/>
Create `/etc/systemd/network/20-wired.network` file, in this example the network name is `enp1s0` :

```
[Match]
Name=enp1s0

[Network]
DHCP=yes
```

<br/><br/>
Link `/etc/resolv.conf` to `/run/systemd/resolve/stub-resolv.conf`. \
`ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`

<br/><br/>
Enable systemd-networkd & systemd-resolved service

```
# systemctl enable systemd-networkd
# systemctl enable systemd-resolved
```

#### dhcpcd

Install dhcpcd

> Note: if you're installing Artix also install it's service depending on your init like `dhcpcd-openrc`, `dhcpcd-runit`, `dhcpcd-s6` or `dhcpcd-dinit`.

`# pacman -S dhcpcd`

<br/><br/>
Enable dhcpcd service

- systemd (Arch) : `systemctl enable dhcpcd`
- OpenRC (Artix) : `rc-update add dhcpcd`
- runit (Artix) : `ln -s /etc/runit/sv/dhcpcd /etc/runit/runsvdir/default`
- s6 (Artix) : `touch /etc/s6/adminsv/default/contents.d/dhcpcd && s6-db-reload`
- dinit (Artix) : `ln -s ../dhcpcd /etc/dinit.d/boot.d/`

#### Reboot

Unmount Arch Linux install volume.

```
# exit
# umount -R /mnt
```

<br/><br/>
Reboot. \
`# reboot`
