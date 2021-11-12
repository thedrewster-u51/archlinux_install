# Archlinux with Root Disk Encryption on BTRF
Basic install that uses the following components and configuration
- Archlinux Install ISO : archlinux-2021.11.01-x86_64.iso
- Filesytem : BTFS (with snapper)
- Encryption : root only / Everything excluding boot
- Boot : Grub on EFI with GPT

The official Archlinux Install guide: https://wiki.archlinux.org/index.php/Installation_Guide

## Write ISO image to USB

Download the archiso image from https://www.archlinux.org/download/

**Linux**
```
dd bs=4M if=archlinux.iso of=/dev/sdx status=progress oflag=sync # on linux
```

**Windows & OSX**

https://www.balena.io/etcher/

## Pre-install Setup

Set your keymap (Optional)
```
loadkeys en
```

Connect to WLAN using wlan0 & iwctl:
```
station wlan0 connect SSID
# or
iwctl --passphrase passphrase station wlan0 connect SSID
```

Check connectivity
```
# Checks internet without DNS
ping 1.1.1.1

# Checks internet with DNS
ping archilinux.org
```

## Partitioning

[TODO] : Writeup on SWAP encryption

**It's worth noting at this point, that as of recently, BTFS supports swap files / subvolumes, so this might not even be required.**

### Layout with swap:
---
| Use                 | Device    | Filesystem |
|---------------------|-----------|------------|
| EFI System Parition | /dev/vda1 | FAT32      |
| Swap (Optional)     | /dev/vda2 | SWAP       |
| Root                | /dev/vda3 | BTRFS      |
---

### Layout without swap:
---
| Use                 | Device    | Filesystem |
|---------------------|-----------|------------|
| EFI System Parition | /dev/vda1 | FAT32      |
| Root                | /dev/vda2 | BTRFS      |
---

First, establish which disk we're installing on
```
# Check your disk name with lsblk -p
root@archiso ~ # lsblk -p
NAME       MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
/dev/loop0   7:0    0 673.7M  1 loop /run/archiso/airootfs
/dev/sr0    11:0    1 846.3M  0 rom  /run/archiso/bootmnt
/dev/vda   254:0    0    20G  0 disk 

# In this case, we're using /dev/vda
export install_disk=/dev/vda
```

Partion the disk
```
# For EFI you require a GPT parition 
parted -s ${install_disk} mktable gpt
parted -s ${install_disk} mkpart ESP fat32 0% 512Mib
parted -s ${install_disk} set 1 boot on
parted -s ${install_disk} mkpart root BTRFS 512Mib 100%
parted -s ${install_disk} print
```

Create luks container (luks1 for compatibility with grub)
```
cryptsetup --type luks1 --cipher aes-xts-plain64 --hash sha512 \
           --use-random --verify-passphrase luksFormat ${install_disk}2
```

Unlock the new crypt device
```
# Unlock the root volume
export crypt_name="arch_root"
cryptsetup open ${install_disk}2 ${crypt_name}
```

Format the paritions
```
mkfs.fat -F32 ${install_disk}1
mkfs.btrfs --force --label root -n 32k /dev/mapper/${crypt_name}
```

Create the subvolumes
```
mount -t btrfs -o compress=lzo /dev/mapper/${crypt_name} /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots

# Add any additional subvolumes of which you want to take separate snapshots
```

Remount BTRFS subvolumes
```
umount /mnt

#   [ BTRFS Mount Options ]
# - x-mount.mkdir helps us create the directory when we mount a volume
# - compress=lzo is the fastest
# - compress=zlib provides the best compression
# - nodatacow can further improve performance, but can potentially be
#   dangerous. Use with caution.
# - autodefrag can improve performance in the long run, but might
#   be slow to start with.
# - inode_cache can also improve performance, but can be slow in the
#   beginning
# - Remove "ssd" if you don't have solid state.
export btrfs_options=defaults,x-mount.mkdir,compress=lzo,ssd,noatime,nodiratime,space_cache,autodefrag

# Remount the partitions
mount -o subvol=@,${btrfs_options} /dev/mapper/${crypt_name} /mnt
mount -o subvol=@home,${btrfs_options} /dev/mapper/${crypt_name} /mnt/home
mount -o subvol=@snapshots,${btrfs_options} /dev/mapper/${crypt_name} /mnt/.snapshots
mkdir -p /mnt/boot/efi
mount ${install_disk}1 /mnt/boot/efi
```
## Install 

Get the fastest, closest, up-to-date mirrors with `reflector`

```
# Where "za" is your contry code.
reflector -c za -f 10 -l 10 | tee /etc/pacman.d/mirrorlist
pacman -Sy
```

Install the base system

```
# If you're testing this on a VM, linux-firmware is not required.
# It is important through to review your specific hardware requiredments
# in the event you need additional packages.

pacstrap /mnt base base-devel linux linux-firmware btrfs-progs snapper \
    net-tools wireless_tools wpa_supplicant neovim efibootmgr \
    grub sudo reflector
```

Generate fstab

```
genfstab -U /mnt | tee -a /mnt/etc/fstab
```

Copy the mirrorlist to the targe OS
```
cp /etc/pacman.d/mirrorlist /mnt/etc/pacman.d/mirrorlist
```

chroot into the base install
```
arch-chroot /mnt /bin/bash
```

Setup system clock
```
ln -s /usr/share/zoneinfo/Africa/Johannesburg /etc/localtime
hwclock --systohc --utc

# Review the date / time
timedatectl

# (Optional) Enable NTP
timedatectl set-ntp true
```

Set the hostname
```
export host_name=miniarch
echo ${host_name} > /etc/hostname
echo -e "127.0.0.1 \t localhost" >> /etc/hosts
echo -e "::1 \t\t localhost" >> /etc/hosts
echo -e "127.0.1.1 \t ${host_name}" >> /etc/hosts
```

Generate and set default locale
```
nvim /etc/locale.gen

# Uncomment en_US.UTF-8 and any other locale's you require
locale-gen
echo LANG=en_ZA.utf8 >> /etc/locale.conf
echo LANGUAGE=en_ZA >> /etc/locale.conf
echo LC_ALL=C >> /etc/locale.conf
```

(Optional) Set virtual console lang and font 
```
echo KEYMAP=en > /etc/vconsole.conf
echo FONT=Lat2-Terminus16 >> /etc/vconsole.conf
```

Set password for root
```
passwd
```

Create user
```
useradd -m -G wheel myuser
passwd myuser
```

Set up sudo
```
export EDITOR=nvim
visudo

## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL  
```

Before we generate the linux image, we need to add a keyfile to the image
in order to unlock the encrypted root volume. Since Grub / EFI takes care of decrypting the root volume to get to the boot partition, that means that the boot image is also effectively encrypted, therefore safe to add the keyfile there. If we dont' do this, we'll be asked for the password twice.

Create keyfile for passwordless login
```
dd bs=512 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
chmod 600 /crypto_keyfile.bin
chmod 600 /boot/initramfs-linux*
cryptsetup luksAddKey /dev/sdXmk# /crypto_keyfile.bin

# Remember to add it to mkinitcpio.conf (See below)
```

Configure mkinitcpio
```
nvim /etc/mkinitcpio.conf

# Early modules load
MODULES=()

# Remember to add your keyfile here.
FILES=(/crypto_keyfile.bin)

# Embed btrfs to initramfs
BINARIES=(/usr/sbin/btrfs /usr/bin/btrfsck)

# Add the encrypt hook, after block and before filesytem
HOOKS="base udev autodetect modconf block encrypt filesystems keyboard fsck"
```

Regenerate initrd image
```
mkinitcpio -p linux
```

## Grub Setup

First we need to get the UUID of the hardware block device, not the encrypted device.
```
blkid -s UUID -o value ${install_disk}2

# OR append to file for ease of editing:

UUID=$(blkid -s UUID -o value ${install_disk}2)
echo "# UUID : ${UUID}" | tee -a /etc/default/grub
```

Configure Grub
```
nvim /etc/default/grub

# add GRUB_ENABLE_CRYPTODISK=y
# add GRUB_DISABLE_SUBMENU=y
# modify GRUB_CMDLINE_LINUX="cryptdevice=UUID=YOURUUIDGOESHERE:archroot:allow-discards"

# Quick reminder on how to cut / paste in vi / neovim
# - Place your cursor at the beginning of what you want to copy.
# - Enter "visual" module by pressing "v"
# - Move the cursior to the end of the text you want to select
# - Then pres either "y" to copy or "d" to cut
# - Move your cursor now to where you want to pase
# - Press "p" to paste.
```

Install grub and generate config file
```
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

exit chroot, unmount and reboot
```
exit
umount -R /mnt
reboot
```
Hopefully at this point you should be prompted for a password, and boot into a base archlinux install.