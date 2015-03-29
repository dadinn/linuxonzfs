
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

The drive will be referred to from now on as `${SATA_DRIVE}`:
```
SATA_DRIVE=/dev/sda
```

Create GPT partitions the following way:

 * using `cgdisk ${SATA_DRIVE}`:
  - create a 500M partition for boot with default type `8300`
  - create a partition on the rest of the disk for LUKS+ZFS with default type `8300`
  - go back to the remaining free space before the boot partition (<1M) and create new partition there with type `ef02` (BIOS boot partition)
  - press `w` to write out the changes
  - press `q` to quit the partitioner
 * sort the partitions using `gdisk ${SATA_DRIVE}`:  
 (due to the creating the BIOS boot partition the last it got numbered as the last, despite being the first on the drive by position)
  - press `s` to sort the partitions
  - press `w` to write out the changes
  - press `q` to quit the partitioner

Alternatively you can do all the above with one command using `sgdisk`:
```
sgdisk ${SATA_DRIVE} -o -n 1:0:+500M -N 2 -N 3 -s -t 1:ef02
```

With the following we can refer to the boot and LUKS+ZFS partitions using variables:
```
function partuuid { sgdisk -i $2 $1|grep "Partition unique GUID:"|sed -e "s;^.*: \(.*\)$;\L\1;" ;  }
SATA_BOOT_PART=/dev/disk/by-partuuid/$(partuuid ${SATA_DRIVE} 2)
SATA_LUKS_PART=/dev/disk/by-partuuid/$(partuuid ${SATA_DRIVE} 3)
```

Format boot partition to EXT4
-----------------------------

Format the partition to be journaling but with little metadata block:
```
mke2fs -m 0 -j ${SATA_BOOT_PART}
```

Set up encrypted device using LUKS
------------------

 * create random 4K key file  
 _Press random keys if the command blocks_
```
dd if=/dev/random of=/root/keyfile bs=1024 count=4
```
 * make the keyfile readonly to root  
 _Don't forget to back this file up!_
```
chmod 0400 /root/keyfile
```
 * Format root partition as LUKS device  
```
cryptsetup luksFormat ${SATA_LUKS_PART} /root/keyfile
```
 * Add an extra passphrase to the LUKS device  
 _This is optional but convenient_
```
cryptsetup luksAddKey ${SATA_LUKS_PART} --key-file /root/keyfile
```
 * Backup LUKS headers  
 _This is optional but highly recommended_
```
cryptsetup luksHeaderBackup ${SATA_LUKS_PART} --header-backup-file /root/luksHeaderBackup
```
 * Open LUKS device  
```
LUKS_NAME=crypt_zfs
cryptsetup luksOpen ${SATA_LUKS_PART} ${LUKS_NAME} --key-file /root/keyfile
```

 * Random-erease the device  
_This technique uses the fact that encrypted data created through
dm-crypt is undistinguisable from random data. We can write the
entire open LUKS device with zeros and it will look as if it was
overwritten by random noise._
```
LUKS_DEVICE=/dev/mapper/${LUKS_NAME}
dd if=/dev/zero of=${LUKS_DEVICE}
```

Set up ZFS pool, zvols, and filesystems
---------------------------------------

In the rest of the guide the root ZFS pool will be refered to using the following definition:
```
RPOOL=rpool
```
 * Create install root directory
```
INSTOOT=/mnt/inst
mkdir ${INSTROOT}
```
 * Create ZFS pool
```
zpool create -o ashift=12 -O atime=off -O mountpoint=none -O snapdir=visible \
  -R ${INSTROOT} ${RPOOL} ${LUKS_DEVICE}
```
 * Create ZFS pool and root filesystem
```
# DUE TO A BUG WITH THE INITRAMFS MODULE THE LEGACY MODE HAS TO BE USED
# SEE https://github.com/zfsonlinux/zfs/issues/2498
zfs create -o compress=lz4 -o mountpoint=legacy ${RPOOL}/system

zfs create ${RPOOL}/system/root
zpool set bootfs=${RPOOL}/system/root ${RPOOL}
```
 * Create additional file systems
```
# This is the definition of ZFS file systems and their mount points
# Entries are spearated by semicolon(;), while mount points are
# separated by colon(:) from the file system paths
ZFSTAB="root/var:/var;home:/home;opt:/opt"

# DUE TO A BUG WITH THE INITRAMFS MODULE MOUNT POINTS CANNOT BE SPECIFIED HERE
# SEE https://github.com/zfsonlinux/zfs/issues/2498
for i in ${ZFSTAB//;/ }; do zfs create -p ${RPOOL}/system/${i/:*}; done
```
 * export and reimport the pool
```
zpool export ${RPOOL}
zpool import -R ${INSTROOT} ${RPOOL}
mount -t zfs rpool/system/root ${INSTROOT}
```

### Create swap space as ZVOL ###

 * create ZVOL
```
zfs create -V 3G -b 4K ${RPOOL}/swap
```
 * format as swap drive
```
SWAP_DEVICE=/dev/zvol/${RPOOL}/swap
mkswap -f ${SWAP_DEVICE}
```

Bootstrap minimal Debian system
===============================

__You will need to have internet connection for the following steps until told again__

 * bootstrap debian
```
debootstrap wheezy ${INSTROOT}
```

 * Grab UUIDs for creating tab entries
```
function fsuuid { blkid -s UUID -o value $1; }
BOOT_UUID=$(fsuuid ${SATA_BOOT_PART} )
LUKS_UUID=$(fsuuid ${SATA_LUKS_PART})
SWAP_UUID=$(fsuuid ${SWAP_DEVICE})
```

 * Create FSTAB entries  
Add the following lines to `fstab`:
```
cat > ${INSTROOT}/etc/fstab <<EOF
# <file system> <mount point> <type>  <options> <dump>  <pass>
${LUKS_DEVICE}  / zfs defaults  0 0
$(for i in ${ZFSTAB//;/ }; do echo -e "${RPOOL}/system/${i/:*}\t${i/*:}\tzfs\tdefaults\t0\t0"; done)
UUID=${SWAP_UUID} none  swap  defaults  0 0
UUID=${BOOT_UUID} /boot ext4  defaults  0 1
EOF
```

 * Create CRYPTTAB entries  
Run the following commands to create an `crypttab` file:
```
cat > ${INSTROOT}/etc/crypttab <<EOF
# <name>  <device>  <key> <options>
${LUKS_NAME}  UUID=${LUKS_UUID} none  luks,discard
EOF
```

 * Mount `dev` filesystems
```
mount --bind /dev ${INSTROOT}/dev
mount --bind /sys ${INSTROOT}/sys
mount --bind /proc ${INSTROOT}/proc
```

 * Chroot into new Debian system
```
LANG=C chroot ${INSTROOT} /bin/bash --login
```

 * Automount using FSTAB entries
```
mount -a
```

Configure the new system
------------------------

 * Create hostname file
```
HOSTNAME=debian-laptop
cat > /etc/hostname <<EOF
${HOSTNAME}
EOF
```

 * Configure APT
```
cat > /etc/apt/sources.list <<EOF
# See https://www.debian.org/mirror/list
deb     http://ftp.uk.debian.org/debian wheezy main contrib non-free
deb-src http://ftp.uk.debian.org/debian wheezy main contrib non-free

# See https://wiki.debian.org/StableUpdates
deb     http://http.debian.net/debian wheezy-updates main contrib non-free
deb-src http://http.debian.net/debian wheezy-updates main contrib non-free

# See https://www.debian.org/security/
deb     http://security.debian.org/ wheezy/updates main contrib non-free
deb-src http://security.debian.org/ wheezy/updates main contrib non-free
EOF
```

 * Tell APT not to install "recommended" packages
```
cat > /etc/apt/apt.conf.d/10norecommends <<EOF
APT::Install-Recommends "false";
APT::Install-Suggests "false";
EOF
```
 
 * Update APT package list and install upgrades
```
aptitude update
aptitude full-upgrade
```

 * Install and configure locales
```
aptitude install locales
locale-gen en_US.UTF-8
```
 
 * Install keyboard layouts for console
```
aptitude install console-setup
```

 * Install usefull command line tools
```
aptitude install less bash-completion
source /etc/bash_completion
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
aptitude install make gcc linux-image-amd64 debian-zfs
```

 * Install GRUB  
_When asked, select `/dev/sda` drive to install GRUB onto_
```
aptitude install -y zfs-initramfs grub2
```

 * Upgrade the packages on the new system
```
aptitude dist-upgrade
```

 * Install `sudo`
```
aptitude install sudo
```
 * install root user
```
useradd -m -G sudo pista
passwd pista
```
 * disable root
```
passwd -l root
```

 * Reboot
```
umount /boot /proc /sys
exit   # from the chroot environment
umount /mnt/debinst/dev
zfs umount -a
swapoff $SWAP_DEV
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


