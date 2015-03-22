
Prepare the "Live" environment
==========================

 * switch to root user for duration of using Live CD  
```
sudo -i
```

 * install Debian packages to Live environment  
```
aptitude install -y gdisk cryptsetup debootstrap
```

Set up ZFS support in the Live environment
---------------------------------------

 * Install the repository to apt sources, update package definitions, and install packages:

```
wget http://archive.zfsonlinux.org/debian/pool/main/z/zfsonlinux/zfsonlinux_4_all.deb
dpkg -i zfsonlinux_4_all.deb
aptitude update
aptitude install -y linux-image-amd64 debian-zfs
```

 * check that ZFS filesystem support got loaded properly
```
modprobe zfs
dmesg | grep ZFS
```

For the later command the following output is expected:
```
[    5.588948] ZFS: Loaded module v0.6.2-524_gd6385c9, ZFS pool
version 5000, ZFS filesystem version 5
```

__After initializing the live environment you won't need internet connection during the next few steps until told again__

Partition drives
================

 * create GPT partitions  
```
cgdisk /dev/sda
```

Partition the device the following way:

 * 500Mb partition for boot with default type `8300`
 * rest as partition for LUKS+ZFS with default type `8300`
 * select the remaining free space (<1M) before the boot partition and create new partition there
 * set the type as `ef02` (BIOS boot partition)
 * sort the partitions using `gdisk /dev/sda` with `s` option

Format boot partition to EXT4
-----------------------------

```
mke2fs -m 0 -j /dev/sda1
```

Set up encrypted device using LUKS
------------------

 * create random 4K key file  
 _Press random keys if the command blocks_
```
dd if=/dev/random of=/root/keyfile bs=1024 count=4
```
 * make the keyfile readonly to root  
 _Don't forget to back it up!_
```
chmod 0400 /root/keyfile
```
 * Format sda2 partition as LUKS device  
```
cryptsetup luksFormat /dev/sda2 /root/keyfile
```
 * Add an extra passphrase to the LUKS device  
 _This is optional but convenient_
```
cryptsetup luksAddKey /dev/sda2 --key-file /root/keyfile
```
 * Backup LUKS headers  
 _This is optional but highly recommended_
```
cryptsetup luksHeaderBackup /dev/sda2 --header-backup-file /mnt/external/luksHeaderBackup
```
 * Open LUKS device  
```
cryptsetup luksOpen /dev/sda2 crypt_zfs --key-file /root/keyfile
```

 * Random-erease the device  
_This technique uses the fact that encrypted data created through
dm-crypt is undistinguisable from random data. We can write the
entire open LUKS device with zeros and it will look as if it was
overwritten by random noise._
```
dd if=/dev/zero of=/dev/mapper/crypt_zfs
```

Set up ZFS pool, zvols, and filesystems
---------------------------------------

 * Create ZFS pool
```
zpool create -o ashift=12 -O mountpoint=none -O atime=off -O snapdir=visible rpool /dev/mapper/crypt_zfs
```
 * Create ZFS filesystems
```
zfs create -o compress=lz4 rpool/system
for i in var usr opt home; do zfs create -p rpool/system/root/$i; done
```
 * Configure and remount the pool
```
zfs umount -a
zfs set mountpoint=/ rpool/system/root
zpool set bootfs=rpool/system/root rpool
zpool export rpool
mkdir /mnt/debinst
zpool import -R /mnt/debinst rpool
```

### Create swap space as ZVOL ###

 * create ZVOL
```
zfs create -V 3G -b 4K rpool/swap
```
 * format as swap drive
```
mkswap -f /dev/zvol/rpool/swap
```
 * turn on swap device
```
swapon /dev/zvol/rpool/swap
```

Bootstrap minimal Debian system
===============================

__You will need to have internet connection for the following steps until told again__

 * bootstrap debian
```
debootstrap wheezy /mnt/debinst
```

 * Mount `dev` filesystem
```
mount --bind /dev /mnt/debinst/dev
```

 * Chroot into new Debian system
```
LANG=C chroot /mnt /bin/bash --login
```

 * mount boot filesystem
```
mount /dev/sda1 /mnt/debinst/boot
```

 * Mount `sys` and `proc` filesystems
```
mount --bind /proc /mnt/debinst/proc
mount --bind /sys /mnt/debinst/sys
```

Configure the new system
------------------------

 * Configure APT
```
vi /etc/apt/sources.list
```

 * Create FSTAB entries  
Add the following lines to `/etc/fstab`:

```
/dev/disk/by-id/scsi-SATA_disk1-part1 /boot/grub  auto  defaults  0 1
/dev/mapper/crypt_zfs / zfs defaults  0  0
/dev/zvol/rpool/swap  none  swap  defaults  0 0
```

 * Create CRYPTTAB entries  
Run the following commands to create an `/etc/crypttab` file:
```
CRYPT_ZFS_UUID=$(blkid | grep 'TYPE="crypto_LUKS"' | sed -e 's;.*UUID="\([^"]*\)".*;\1;')
echo crypt_zfs UUID=$CRYPT_ZFS_UUID none luks,discard >> /etc/crypttab
```

 * Update INITRAMFS config
```
echo target=crypt_zfs,source=UUID=$CRYPT_ZFS_UUID,key=none,rootdev,discard >> /etc/initramfs-tools/conf.d/cryptroot
```

 * Install and configure locales and keyboard layouts
```
aptitude install locales
locale-gen en_US.UTF-8
aptitude install console-setup
```

 * install gdisk and cryptsetup
```
aptitude install gdisk cryptsetup
```

 * install lsb-release  
_This is a dependency to install ZFS... must be a bug in the dpkg package definition_
```
aptitude install lsb-release
```

 * Install ZFS packages  
_The same steps as setting it up on live environment_
```
wget http://archive.zfsonlinux.org/debian/pool/main/z/zfsonlinux/zfsonlinux_4_all.deb
dpkg -i zfsonlinux_4_all.deb
aptitude update
aptitude install -y linux-image-amd64 debian-zfs
```

 * Install GRUB  
_When asked, select `/dev/sda` drive to install GRUB onto_
```
aptitude install -y grub2 zfs-initramfs
```

 * Upgrade the packages on the new system
```
aptitude dist-upgrade
```

 * Set root password
```
passwd
```

 * Reboot
```
umount /boot /proc /sys
exit   # from the chroot environment
umount /mnt/debinst/dev
zfs umount -a
zpool export rpool
cryptsetup luksClose crypt_zfs
reboot
```

A quick rescue environment
=========================

Boot into a Live disk and run the following commands

```
wget http://archive.zfsonlinux.org/debian/pool/main/z/zfsonlinux/zfsonlinux_4_all.deb
dpkg -i zfsonlinux_4_all.deb
aptitude update
aptitude install -y gdisk cryptsetup linux-image-amd64 debian-zfs
cryptsetup luksOpen /dev/sda2 crypt_zfs --keyfile /path/to/keyfile
zpool import -R /mnt rpool
zfs mount -a
mount --bind /dev /mnt/dev
LANG=C chroot /mnt /bin/bash --login
mount /dev/sda1 /boot
mount --bind /proc /proc
mount --bind /sys /sys
```

Debian WiFi drivers for ThinkPads
=================================

You need to download firmware from http://intellinuxwireless.org and put `*iwlwifi-5000-ucode*` in the `/lib64/firmware` directory

Install laptop packages
-----------------------

```
tasksel install laptop
```

Resources
=========
* https://help.ubuntu.com/community/encryptedZfs
* https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Debian-GNU-Linux-to-a-Native-ZFS-Root-Filesystem
* http://www.larsko.org/ZfsUbuntu
* http://www.firewing1.com/howtos/fedora-20/installing-zfs-and-setting-pool


