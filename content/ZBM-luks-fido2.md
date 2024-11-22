---
title: Linux PC running ZFSBootMenu with encryption key on LUKS partition unlocked with Yubikey (fido2)
draft: true
tags:
  - setup
date: 2024-10-31
description: Setup with rEFInd + ZFSBootMenu with native ZFS encryption key located on a LUKS partition that is unlocked via a Yubikey (fido2).
---
These notes document what I've learned while setting up a new laptop. Key features:

- BIOS is config to load [rEFInd](https://www.rodsbooks.com/refind/) - so I can also boot a Windows OS via USB;
- [ZFSBootMenu](https://github.com/zbm-dev/zfsbootmenu) with ability to unlock ZFS encryption from a LUKS partition;
- LUKS partition unlocked with a Yubikey (fido2) instead of the TPM chip;
- use of [Dracut](https://github.com/dracutdevs/dracut/blob/master/man/dracut.usage.asc) instead of initramfs (or mkinitcpio on Arch) to create the initial ramdisk;
- Secure boot enabled via Shim signed EFI and have everything else signed with locally generated key & certificate;
- install Debian testing from the weekly build [Live ISO](https://cdimage.debian.org/cdimage/weekly-live-builds/amd64/iso-hybrid/);

The plan of the guide (highly inspired by the [ZBM docs](https://docs.zfsbootmenu.org/en/v2.3.x/guides/debian/bookworm-uefi.html)) is to show how to:
- Configure the live ISO such that you can ssh into it & run the commands from another PC (unlike the ZBM docs, we'll do it from the testing Debian)
- Define disk variables - same Boot and OS device on a mirror ZFS pool
- Disk preparation - 4 partitions:
	- EFI boot partition
	- LUKS partition
	- zpool partition
- ZFS pool creation - everything encrypted & create 1 unencrypted dataset & HOME datasets are not mounted automatically so we can decide which home dataset we want to use given the distribution we boot into.
- Install Debian
- Basic Debian Configuration
- ZFS Configuration
- Install and configure ZFSBootMenu from source
- Setup LUKS partition & add unlock via FIDO2 device
- Configure EFI boot entries via latest [rEFInd](https://www.rodsbooks.com/refind/)
- Prepare for first boot

Future work:
- Boot into NixOS from the ZFSBootMenu
## Configure Live Environment

Flash a USB with [Ventoy](https://www.ventoy.net/en/index.html) & add the Debian ISO. Add the following bash script to the `CONFIG` partition:

Add you public key:
- Create the file:
```bash
cd INTO-THE-USB-CONFIG-DIRECTORY
cat <<EOF > id_ed25519.pub
ssh-ed25519 AAAAC3N*** YOUR-COMMENT-OR-EMAIL
EOF
```
- Copy the file from your home dir:
```bash
cd INTO-THE-USB-CONFIG-DIRECTORY
cp ~/.ssh/id_ed25519.pub ./ 
```
Add the following content into `setup_ssh.sh`:
```bash
#!/bin/bash
__dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

if [ $(id -u) -ne 0 ]
  then echo "Please run this script as root or using sudo!"
  exit
fi

apt update
apt install --yes openssh-server nano tmux

mkdir -p ~/.ssh && chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys

cat ${__dir}/id_ed25519.pub >> ~/.ssh/authorized_keys
```
You can now ssh into your machine.

Useful commands:
```bash
# Find the IP and etherface name
ip a
# Restart the DHCP to get an IP for the etherface eth0
dhclient eth0
```

### Source `/etc/os-release`
```bash
source /etc/os-release
export ID
```

### Configure and update APT
```bash
apt install lsb-release ca-certificates apt-transport-https
CODENAME=$(lsb_release --codename --short)

cat <<EOF > /etc/apt/sources.list
deb https://deb.debian.org/debian $CODENAME main contrib non-free-firmware
deb-src https://deb.debian.org/debian $CODENAME main contrib non-free-firmware
EOF

apt update
```

### Install helpers
```bash
apt install debootstrap gdisk dkms linux-headers-$(uname -r)
apt install zfsutils-linux
```

### Generate `/etc/hostid`
Set a unique ID within the Linux devices you use:
```bash
zgenhostid -f 0x00bab10c
# Generate the ID:
zgenhostid -f "0x$( date +%s | cut -c1-8 )"
```
Useful commands:
```bash
# Print the current hostid
hostid
```
## Define disk variables

The Boot and OS will run from the same devices in a mirror configuration.

```bash
export BOOT_DISK="/dev/nvme0n1"
export BOOT_PART="1"
export BOOT_DEVICE="${BOOT_DISK}p${BOOT_PART}"

export KEYS_PART="2"
export KEYS_DEVICE="${BOOT_DISK}p${KEYS_PART}"

export POOL_PART="3"
export POOL_DEVICE="${BOOT_DISK}p${POOL_PART}"
```

Helpful commands:
```bash
lsblk -f
df -H
```

## Disk preparation (4 partitions)

Wipe partitions:
```bash
zpool labelclear -f "$BOOT_DISK"

wipefs -a "$BOOT_DISK"
sgdisk --zap-all "$BOOT_DISK"
```
Helpful commands:
```bash
# List partition type codes: 
sgdisk --list-types
lsblk -o NAME,SIZE,FSTYPE,TYPE
```
You will see the code we will later use:
- `ef00` = EFI System
- `8309` = Linux LUKS
- `bf00` = Solaris root
### EFI boot partition
Create EFI boot partition:
```bash
sgdisk -n "${BOOT_PART}:1m:+1GiB" -t "${BOOT_PART}:ef00" "$BOOT_DISK"
```
### LUKS partition
Create LUKS2 keys partition:
```bash
sgdisk -n "${KEYS_PART}:0:+1GiB" -t "${KEYS_PART}:8309" -c "${KEYS_PART}:KEYSTORE" "$BOOT_DISK"
```
### zpool partition
Create zpool partition:
```bash
sgdisk -n "${POOL_PART}:0:-10m" -t "${POOL_PART}:bf00" "$POOL_DISK"
```

## ZFS pool creation
Everything encrypted & create 1 unencrypted dataset:
```bash
echo 'SomeKeyphrase' > /etc/zfs/zroot.key
chmod 000 /etc/zfs/zroot.key
```
Create the pool:
```bash
zpool create -f -o ashift=12 \
 -O compression=lz4 \
 -O acltype=posixacl \
 -O xattr=sa \
 -O relatime=on \
 -O encryption=aes-256-gcm \
 -O keylocation=file:///etc/zfs/zroot.key \
 -O keyformat=passphrase \
 -o autotrim=on \
 -o compatibility=openzfs-2.1-linux \
 -m none zroot "$POOL_DEVICE"
```

### Create initial file systems

HOME datasets are not mounted automatically so we can decide which home dataset we want to use given the distribution we boot into.

```bash
zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/${ID}

zfs create -o mountpoint=none -o overlay=off zroot/HOME
zfs create -o mountpoint=/home -o canmount=noauto zroot/HOME/work 
zfs create -o mountpoint=/home -o canmount=noauto -o encryption=off zroot/HOME/unencrypted

zpool set bootfs=zroot/ROOT/${ID} zroot
```


> [!IMPORTANT] Turn off overlay for `zroot/HOME`
> Turning off [the overlay](https://openzfs.github.io/openzfs-docs/man/master/7/zfsprops.7.html#overlay) for all children datasets will ensure that only 1 dataset is mounted at a time.


### Export, then re-import with a temporary mountpoint of `/mnt`
Export and import pool:
```bash
zpool export zroot
zpool import -N -R /mnt zroot
zfs load-key -L prompt zroot
```

Mount it:
```bash
zfs mount zroot/ROOT/${ID}
zfs mount zroot/HOME/work
```

Verify that everything is mounted correctly:
```bash
mount | grep mnt

zroot/ROOT/debian on /mnt type zfs (rw,relatime,xattr,posixacl)
zroot/HOME/work on /mnt/home type zfs (rw,relatime,xattr,posixacl)
```

## Install Debian
Update device symlinks:
```bash
udevadm trigger
```

Install Debian via [debootstrap](https://gist.github.com/varqox/42e213b6b2dde2b636ef):
```bash
# CODENAME=$(lsb_release --codename --short)
debootstrap $CODENAME /mnt
```

### Copy files into the new install
```bash
cp /etc/hostid /mnt/etc/hostid
cp /etc/resolv.conf /mnt/etc/
mkdir /mnt/etc/zfs
cp /etc/zfs/zroot.key /mnt/etc/zfs
```

### Chroot into the new OS
```bash
mount -t proc proc /mnt/proc
mount -t sysfs sys /mnt/sys
mount -B /dev /mnt/dev
mount -t devpts pts /mnt/dev/pts
chroot /mnt /bin/bash
```

## Basic Debian Configuration

### Set a hostname
```bash
echo 'YOURHOSTNAME' > /etc/hostname
echo -e '127.0.1.1\tYOURHOSTNAME' >> /etc/hosts
```

### Set a root password & apt sources
```bash
passwd
```

Configure apt. Use other mirrors if you prefer:
```bash
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware

deb http://deb.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian-security/ bookworm-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian bookworm-updates main contrib
deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware

deb http://deb.debian.org/debian bookworm-backports main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm-backports main contrib non-free non-free-firmware
EOF
```

All backports are disabled by default. To use it, run:
```bash
apt -t bookworm-backports install PACKAGE
```

### Update & install additional base packages
```bash
apt update
apt install locales keyboard-configuration console-setup bash-completion
```

### Configure packages to customize local and console properties
```bash
dpkg-reconfigure locales tzdata keyboard-configuration console-setup
```

For my machine, I choose:
- enable the `en_US.UTF-8` locale because some programs require it
- [Generic 104-key](https://mechkeys.com/blogs/guide/understanding-different-physical-layouts-for-keyboards-ansi-vs-iso-vs-jis) keyboard (see [debian wiki](https://wiki.debian.org/Keyboard))
- font TerminusBold22x11 that has the size 11x22

## Setup LUKS partition & add unlock via FIDO2 device
```bash
apt install cryptsetup
```
Format it:
- if you want to see the progress via pv (`apt install pv`):
```bash
pv -tpreb /dev/zero | dd of=/dev/mapper/KEYSTORE bs=1024M
```
- without progress:
```bash
dd if=/dev/zero of=/dev/mapper/KEYSTORE bs=1024M
```
Use EXT4:
```bash
mkfs.ext4 /dev/mapper/KEYSTORE
```
Mount it:
```bash
mkdir -p /etc/zfs/keys
mount /dev/mapper/KEYSTORE /etc/zfs/keys
```

## ZFS Configuration

## Install and configure ZFSBootMenu from source


## Configure EFI boot entries via latest [rEFInd](https://www.rodsbooks.com/refind/)

## Prepare for first boot



## Put rEFInd first to boot into

Make use of [refind-mkdefault](https://manpages.debian.org/bookworm/refind/refind-mkdefault.8.en.html):

```bash
refind-mkdefault
# If you have renamed refind to BOOT, then run:
# refind-mkdefault -L BOOT
```

## LUKS partition

## Own Dracut config (ZFS encrypted)

## Build ZBM with LUKS support

### ZBM Dracut config (NOT encrypted)

## Key resources

- [ZFSBootMenu docs](https://docs.zfsbootmenu.org/)
- Wiki [Arch systemd-cryptenroll (fido2)](https://wiki.archlinux.org/title/Systemd-cryptenroll#FIDO2_tokens)
- [Unlock LUKS volume with a YubiKey](https://www.guyrutenberg.com/2022/02/17/unlock-luks-volume-with-a-yubikey/)
- Instructions [how to install Debian using debootstrap](https://gist.github.com/varqox/42e213b6b2dde2b636ef)
