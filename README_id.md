# Menginstall Arch/Artix Linux di filesystem Btrfs dengan enkripsi

Disini kita akan menginstall Arch/Artix Linux dengan full disk encryption (termasuk `/boot`), Btrfs, snapper dan booting dari snapshot.


## Mempersiapkan drive

### Pemartisian
Anda dapat menggunakan `fdisk` atau `cfdisk` untuk memartisi drive anda, apabila anda menggunakan sistem UEFI jangan lupa buat partisi ESP. \
Apabila anda melakukan dual booting buat saja partisi baru & langsung saja ke [Memformat partisi](#memformat-partisi) tetapi jangan memformat partisi ESP. 

<b>Contoh pemartisian untuk EFI</b>

| Partisi	| Ukuran | Tipe | Catatan |
|- 		|- 	|- 			|- 	|
| ESP 		| 200M 	| EFI system 		| Apabila sudah melakukan instalasi OS lain di sistem UEFI, partisi ini sudah dibuat oleh proses instalasi OS sebelumnya. |
| LUKS 		| >20GB | Linux filesystem 	| 	|

#### Memformat partisi

Format partisi ESP apabila anda menggunakan sistem UEFI disini partisi tersebut adalah `/dev/sda1`. \
`# mkfs.vfat /dev/sda1`
<br/><br/>

Format partisi LUKS yang baru dibuat, disini partisi tersebut adalah `/dev/sda2`. \
`# cryptsetup luksFormat --pbkdf=pbkdf2 /dev/sda2`
<br/><br/>

Buka partisi LUKS, kita akan menggunakan nama `btw` sebagai nama partisi LUKS. \
`# cryptsetup open /dev/sda2 btw`
<br/><br/>

#### Membuat LVM di partisi LUKS & memformat volume
Kita akan menggunakan LVM on LUKS sehingga root & swap akan berada di partisi LUKS yang sama. \
Untuk volume swap buat sebesar ukuran RAM anda, di tutorial ini ukuran RAMnya 4GB.
<br/><br/>

Buat volume group (VG) lvm di partisi LUKS, disini kita juga menggunakan `btw` sebagai nama VG. \
```
# vgcreate btw /dev/mapper/btw
```
<br/><br/>
Buat logical volume (LV) root & swap di VG.
```
# lvcreate -L 4G -n swap btw
# lvcreate -l +100%FREE -n root btw
```
<br/><br/>
Format kedua LV yang tadi dibuat.
```
# mkswap /dev/btw/swap
# mkfs.btrfs /dev/btw/root
```

#### Membuat subvolume btrfs & melakukan mounting
Aktifkan swap yang tadi dibuat. \
`# swapon /dev/btw/swap`
<br/><br/>
Mount LV root yang tadi dibuat. \
`# mount -o compress=zstd,noatime /dev/btw/root /mnt`
<br/><br/>
Buat subvolume btrfs.
```
# btrfs subvol create /mnt/@
# btrfs subvol create /mnt/@var
# btrfs subvol create /mnt/@home
# btrfs subvol create /mnt/@snapshots
```

|Subvolume 	|Mountpoint 	|Deskripsi|
|- 		|- 		|- 				|
|`@` 		|`/` 		|Subvolume root |
|`@var` 	|`/var` 	|Subvolume `/var` |
|`@home` 	|`/home` 	|Subvolume home |
|`@snapshots` 	|`/.snapshots/` |Snapshot untuk subvolume root (`@`)|

<b>Subvolume tambahan</b>
|Subvolume 		|Mountpoint 			|Description 			|
|- 			|- 				|- 				|
|`@srv`			|`/srv` 			| 				|
|`@libvirt-image` 	|`/var/lib/libvirt/images/` 	|Lokasi pool libvirt|
|`@mysql` 		|`/var/lib/mysql` 		|Lokasi database MariaDB/MySQL |

<br/><br/>
Matikan Copy on write (COW) Untuk subvolume `@var`. \
`# chattr +C /mnt/@var`

<br/><br/>
Unmount btrfs volume & mount subvolume root (`@`).
```
# umount /mnt
# mount -o compress=zstd,noatime,subvol=@ /dev/btw/root /mnt
```

<br/><br/>
Mount ESP & subvolume btrfs selain subvolume root (`@`).
```
# mkdir -p /mnt/{boot/efi,var,home,.snapshots}
# mount /dev/sda1 /mnt/boot/efi
# mount -o compress=zstd,noatime,subvol=@home /dev/btw/root /mnt/home
# mount -o compress=zstd,noatime,subvol=@var /dev/btw/root /mnt/var
# mount -o compress=zstd,noatime,subvol=@snapshots /dev/btw/root /mnt/.snapshots
```
## Instalasi

### Instalasi package dasar.

(Optional, Arch only) Gunakan reflector untuk membuat mirrorlist atau edit file `/etc/pacman.d/mirrorlist` apabila anda ingin menginstall artix.

> Catatan: Anda bisa menggunakan `--country` untuk mengganti lokasi mirror ke negara anda dan atau negara yang terdekat, disini kita akan menggunakan mirror yang berlokasi di Indonesia & Singapore. 

`# reflector --save /etc/pacman.d/mirrorlist --latest 20 --sort rate --verbose --protocol http,https --country Indonesia,Singapore  `

<br/><br/>
Install package dasar.

* Untuk Arch:
```
# pacstrap -K /mnt base linux linux-firmware lvm2 cryptsetup btrfs-progs snapper sudo
```
* Untuk Artix:
> Catatan: Anda dapat memilih init yang ingin anda gunakan (`openrc`,`runit`,`s6` atau `dinit`), untuk package init s6 (setelah linux-firmware) ganti `s6` menjadi `s6-init` .
<pre>
# basestrap /mnt base linux linux-firmware <b>init</b> elogind-<b>init</b> lvm2-<b>init</b> cryptsetup snapper sudo
</pre>

### Konfigurasi Arch Linux
#### Konfigurasi fstab
Buat fstab.
* Untuk Arch: \
`# genfstab -U /mnt >> /mnt/etc/fstab`
* Untuk Artix: \
`# fstabgen -U /mnt >> /mnt/etc/fstab`

<br/><br/>
Edit file fstab.
> Catatan: Anda dapat menggunakan vim atau nano untuk mengedit file, nano adalah text editor yang paling mudah digunakan tetapi disini kita akan menggunaka vim sebagai contoh. Untuk Artix dapat menggunakan vi dikarenakan hanya vi & nano yang terinstall di Artix Live ISO.

`# vim /mnt/etc/fstab`

Hapus semua `subvolid=xxx` dari opsi untuk mount btrfs.
> Tip: Apabila menggunakan vim Anda dapat menggunakan `%s/subvolid=[0-9][0-9][0-9],//g` untuk menghapus semua opsi subvolid.

Contoh `/mnt/etc/fstab` di instalasi saya:
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

#### Memindahkan lokasi database pacman
Pindahkan lokasi database pacman dari  `/mnt/var/lib/pacman/` ke `/mnt/usr/var/lib/pacman` agar dapat dimasukkan ke snapshot.
```
# mkdir -p /mnt/usr/var/lib/
# mv /mnt/var/lib/pacman /mnt/usr/var/lib
```
Kemudian edit `/mnt/etc/pacman.conf` dan ganti `DBPath      = /var/lib/pacman/` menjadi `DBPath      = /usr/var/lib/pacman/`. 

#### Chroot ke instalasi Arch/Artix
* Untuk Arch: \
`# arch-chroot /mnt /bin/bash`
* Untuk Artix: \
`# artix-chroot /mnt /bin/bash`

#### Konfigurasi locale & zona waktu
Link `/etc/localtime` ke `/usr/share/zoneinfo/<Wilayah>/<Kota>`, disini kita akan menggunakan zona waktu Jakarta (WIB). \
`# ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime`

<br/><br/>
Buat `/etc/adjtime` \
`# hwclock --systohc`

<br/><br/>
Edit `/etc/locale.gen` dan hapus tanda `#` didepan `en_US.UTF-8 UTF-8` dan locale lainnya. 
> Catatan: Kita belum menginstall text editor, untuk menginstall editor gunakan `pacman -S vim` untuk vim atau `pacman -S nano` untuk nano.

<br/><br/>
Buat locale. \
`# locale-gen`

<br/><br/>
Buat file `/etc/locale.conf` untuk mengeset locale. \
Contoh `/etc/locale.conf` di instalasi saya:
```
LANG=en_US.UTF-8
```
#### Konfigurasi hostname
Ganti hostname disini kita akan menggunakan `i-use-arch-btw` sebagai hostname. \
`echo i-use-arch-btw > /etc/hostname`

#### Membuat initramfs & membuat kunci LUKS kedua
Buat kunci LUKS kedua di `/etc/lukskeys/root`.
```
# mkdir /etc/lukskeys
# chmod 700 /etc/lukskeys
# dd if=/dev/random of=/etc/lukskeys/root bs=1024 count=128
```

<br/><br/>
Tambahkan kunci kedua ke partisi LUKS. \
`# cryptsetup luksAddKey /dev/sda2 /etc/lukskeys/root`

<br/><br/>
Edit `/etc/mkinitcpio.conf` untuk menambahkan kunci kedua ke  initramfs & dan menambahkan hook encrypt, LVM2 & resume ke initramfs. 
* hook initramfs systemd (Arch): \
`HOOKS=(base systemd autodetect keyboard modconf block sd-encrypt lvm2 filesystems fsck)`
* hook initramfs udev (Arch & Artix): \
`HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems resume fsck)`

Dan tambahkan kunci LUKS kedua ke initramfs. \
`FILES=(/etc/lukskeys/root)`

<br/><br/>
(initramfs systemd) Edit `/etc/crypttab.initramfs` dan tambahkan partisi LUKS.

> Catatan: Anda dapat melihat UUID partisi/volume menggunakan `blkid <device>`.
> ```
> # blkid /dev/sda2
> /dev/sda2: UUID="cc1b673c-159d-4224-9af0-5465373b63a9" TYPE="crypto_LUKS" PARTUUID="408fb50d-bb3b-2344-b928-ebcee8d2cc61"
> # blkid /dev/btw/swap
> /dev/btw/swap: UUID="ed071d58-59cf-4c38-8fc4-3d775207c5c2" TYPE="swap"
> ```
<pre>
btw             UUID=<b>your-LUKS-partition-UUID</b> 	/etc/lukskeys/root
</pre>

Anda dapat menyalakan TRIM apabila melakukan instalasi di SSD atau VM.
<pre>
btw             UUID=<b>your-LUKS-partition-UUID</b> 	/etc/lukskeys/root 	discard
</pre>


<br/><br/>
(udev initramfs) Konfigurasi GRUB agar GRUB dapat membuka partisi LUKS dan ganti `GRUB_CMDLINE_LINUX_DEFAULT=`.
> Catatan: apabila menggunakan initramfs udev Install GRUB terlebih dahulu, ikuti [Instalasi repo Chaotic-AUR](#instalasi-repo-chaotic-aur) & [Menginstall & konfigurasi GRUB](#Menginstall--konfigurasi-grub) sebelum melakukan konfigurasi dibawah.

<pre>
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=<b>your-LUKS-partition-UUID</b>:btw cryptkey=rootfs:/etc/lukskeys/root resume=UUID=<b>your-swap-LV-UUID</b> loglevel=3 quiet"
</pre>
Anda dapat menyalakan TRIM apabila melakukan instalasi di SSD atau VM.
<pre>
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=<b>your-LUKS-partition-UUID</b>:btw:allow-discards cryptkey=rootfs:/etc/lukskeys/root resume=UUID=<b>your-swap-LV-UUID</b> loglevel=3 quiet"
</pre>


<br/><br/>
Buat config grub. \
`# grub-mkconfig -o /boot/grub/grub.cfg`

<br/><br/>
Buat initramfs. \
`# mkinitcpio -P`

#### Instalasi repo Chaotic-AUR
Karena kita menggunakan LUKS2 untuk melakukan enkripsi instalasi Arch Linux kita harus melakukan instalasi GRUB bootloader (`grub-improved-luks2-git`) yang telah dimodifikasi dari AUR, tapi kita dapat menginstallnya dari repo Chaotic-AUR yang membuat beberapa package dari repo AUR.

<br/><br/>
(Artix) Tambahkan repo universe dengan memasukan daftar repositori dibawah file `/etc/pacman.conf`: 
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
(Artix) Install repositori Arch.
```
# pacman -Syu artix-archlinux-support
```

<br/><br/>
(Artix) Tambahkan repositori Arch dibawah repositori Artix ke `/etc/pacman.conf`: 

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
Import key Chaotic-AUR & install repositori Chaotic-AUR.
```
pacman-key --recv-key FBA220DFC880C036 --keyserver keyserver.ubuntu.com
pacman-key --lsign-key FBA220DFC880C036
pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-keyring.pkg.tar.zst' 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-mirrorlist.pkg.tar.zst'
```

<br/><br/>
Tambahkan repositori Chaotic-AUR dibawah repositori Arch ke `/etc/pacman.conf`:
```
[chaotic-aur]
Include = /etc/pacman.d/chaotic-mirrorlist
```

#### Menginstall & konfigurasi GRUB
Install `grub-improved-luks2-git`.\
```# pacman -Syu grub-improved-luks2-git```

<br/><br/>
(Untuk UEFI) Install `efibootmgr`. \
`# pacman -S efibootmgr`

<br/><br/>
Edit `/etc/default/grub` & hapus tanda `#` `GRUB_ENABLE_CRYPTODISK=y` 

<br/><br/>
Install GRUB.
* Untuk UEFI: `# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Arch Linux"`
* Untuk BIOS: `# grub-install --target=i386-pc /dev/sda`

<br/><br/>
Buat config GRUB. \
`# grub-mkconfig -o /boot/grub/grub.cfg`

#### Membuat user baru & konfigurasi sudo
Edit file sudoers dengan `visudo` & hapus tanda `#` didepan `%wheel ALL=(ALL:ALL) ALL`.
> Catatan: Apabila menggunakan text editor nano, gunakan `EDITOR=nano visudo`.

<br/><br/>
Buat user baru, kita akan membuat user `cincinmasukmangkok`.
```
# useradd -mG wheel cincinmasukmangkok
# passwd cincinmasukmangkok
```

#### Konfigurasi Snapper
Buat config Snapper `root`.
```
# umount /.snapshots
# rm -rf /.snapshots
# snapper --no-dbus -c root create-config /
# btrfs subvol delete /.snapshots
# mkdir /.snapshots
# mount /.snapshots
```

<br/><br/>
Edit `/etc/snapper/configs/root` dan ganti `TIMELINE_CREATE="yes"` menjadi `TIMELINE_CREATE="no"` agar snapshot hanya dibuat secara otomatis setelah menginstall atau mengupdate package. 

<br/><br/>
Install `snap-pac` & `grub-btrfs`.
```
# pacman -S snap-pac grub-btrfs
```
<br/><br/>
(Arch) nyalakan service grub-btrfs.
```
# systemctl enable grub-btrfs.path
```
<br/>

atau gunakan hook grub-btrfs \
`# pacman -S snap-pac-grub`

### Menginstall snapper-rollback
`snapper-rollback` tersedia di AUR tetapi tidak tersedia di Chaotic-AUR saat tutorial ini dibuat, tetapi kita dapat menggunakan `paru` untuk menginstall package dari AUR.
```
# pacman -S base-devel paru
# sudo -u cincinmasukmangkok paru -S snapper-rollback
```

<br/><br/>
Edit `/etc/snapper-rollback.conf` and ganti `dev` menjadi `dev = /dev/<VG>/<LV>`. 

### Konfigurasi Jaringan
#### systemd-networkd (Arch Linux only)

Lihat dulu nama network interface biasanya `enpXsY` atau `ethX` untuk ethernet. 
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
Buat file `/etc/systemd/network/20-wired.network`, sebagai contoh nama interfacenya `enp1s0` :
```
[Match]
Name=enp1s0

[Network]
DHCP=yes
```

<br/><br/>
Link `/run/systemd/resolve/stub-resolv.conf` ke `/etc/resolv.conf`. \
`ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`

<br/><br/>
Nyalakan service systemd-networkd & systemd-resolved
```
# systemctl enable systemd-networkd
# systemctl enable systemd-resolved
```

#### dhcpcd
Install dhcpcd
> Catatan: Untuk Artix dapat menginstall service dhcpcd tergantung init yang digunakan (`dhcpcd-openrc`, `dhcpcd-runit`, `dhcpcd-s6` atau `dhcpcd-dinit`).

`# pacman -S dhcpcd`

<br/><br/>
Nyalakan service dhcpcd

* systemd (Arch) : `systemctl enable dhcpcd`
* OpenRC (Artix) : `rc-update add dhcpcd`
* runit (Artix) : `ln -s /etc/runit/sv/dhcpcd /etc/runit/runsvdir/default`
* s6 (Artix) : `touch /etc/s6/adminsv/default/contents.d/dhcpcd && s6-db-reload`
* dinit (Artix) : `ln -s ../dhcpcd /etc/dinit.d/boot.d/`

#### Reboot
Unmount volume instalasi.
```
# exit
# umount -R /mnt
```

<br/><br/>
Reboot. \
`# reboot`