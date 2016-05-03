# Basic Arch Linux Install

Notes on a basic install of Arch on a VM. 

I made these notes for my own reference and I hope they can help you too.

## Install Details
- BIOS
- MBR
- Single ext4 partition
- GRUB
- Guide made using release 2016.04.01
- Linux Kernel 4.5.1, x86_64
- Virtual Box 5.0.20 Running on Mac OS X 10.10.5 Host

## Disclaimer
These notes are ***not*** meant to serve as a complete how to guide to Arch Linux. 

These are notes and tips that are meant to supplement the 
[Arch Wiki](https://wiki.archlinux.org/) and 
[Forum](https://bbs.archlinux.org/). 
Do ***not*** blindly follow these instructions. Both for the safety of your system
and in accordance with 
[The Arch Way](https://wiki.archlinux.org/index.php/Arch_Linux#Principles), 
if you don't understand what a step does I highly recommend that you stop and go look it up.

## Useful Links
Highly recommended reading.
- [Arch Installation Guide](https://wiki.archlinux.org/index.php/installation_guide)
- [Arch Beginners Guide](https://wiki.archlinux.org/index.php/beginners'_guide)
- [Getting and Installing Arch](https://wiki.archlinux.org/index.php/Category:Getting_and_installing_Arch)
- [Arch FAQ](https://wiki.archlinux.org/index.php/Frequently_asked_questions)

## Installation Steps
### Get Arch
1. [Download](https://www.archlinux.org/download/) the latest official Arch ISO.
2. Verify checksum. Download signature and run 
    `gpg --verify archlinux-<version>-dual.iso.sig`

### Virtual Box Setup
What I used, (you can definitely get away with fewer system resources):
- 4096 MB memory
- 16 GB virtual HD, dynamically allocated
- 2 CPU
- 2nd Host Only Network Adapter (optional)

Attach Arch Live Iso image to a disk drive.

### Initial Steps on Booting Into Archiso
1. Test internet. `ping 8.8.8.8`
    - `ip a` `ip link` `iw dev`
2. Update the [system clock](https://wiki.archlinux.org/index.php/Systemd-timesyncd).
`timedatectl set-ntp true` `timedatectl status`

### Partitioning
Research what partitioning scheme is best for you. 
This is a simple one that works with MBR/BIOS. 
UEFI generally requires a seperate boot partition.
If you are using Virtual Box then you are using BIOS by default. 
(It is possible to set Virtual Box to use UEFI firmware instead.)

- Use `lsblk` to list block memory devices
- Partition `/dev/sda`. Your device's name may be different.
- Use fdisk to create an MBR partition table
    - `fdisk /dev/sda`
    - Use `o` to start a new table.
    - Use `n` to start a new partition.
    - Use all defaults: e.g. `primary`, `2048-33554431`
    - Use `p` to print the table and review it.
    - Use `w` to write the partition table. THIS WILL DESTROY ANY EXISTING DATA ON THE DRIVE.
- `lsblk` will now list `/dev/sda1` under `/dev/sda`
- Format the partition. `mkfs.ext4 /dev/sda1`
- Mount the partition. `mount /dev/sda1 /mnt`

### Install Arch
`pacstrap -i /mnt base base-devel`
#### Offline Install
This requires extra clean up. **Be sure to read [this](https://wiki.archlinux.org/index.php/Archiso#Installation_without_Internet_access)!**
```
time cp -ax / /mnt
cp -vaT /run/archiso/bootmnt/arch/boot/$(uname -m)/vmlinuz /mnt/boot/vmlinuz-linux
```

### Generate fstab
`genfstab -U /mnt >> /mnt/etc/fstab`

### Change root
`arch-chroot /mnt /bin/bash`

### Set hostname
`echo MY_NEW_HOSTNAME > /etc/hostname`

Also append the new hostname to the two lines ending in `localhost` in the file `/etc/hosts`.

e.g. For the first line,
```
127.0.0.1   localhost.localdomain   localhost MY_NEW_HOSTNAME
```

### Set time zone
```
ln -s /usr/share/YOUR/TIMEZONE /etc/localtime
hwclock --systohc --utc
```

### Locale
Uncomment locales in `/etc/locale.gen` then run `locale-gen`. 

Create `/etc/locale.conf` and add,
```
LANG=en_US.UTF-8
```

### Generate Ramdisk Environment
Set hooks in `/etc/mkinitcpio.conf` and regenerate initramfs image
`mkinitcpio -p linux`

### Set up pacman keys
You may need to run the following to generate a pgp key and populate the default keys, 
```
pacman-key --init
pacman-key --populate
```
If you need to sign a new key for a package from the AUR,
```
pacman-key -r key-goes-here
pacman-key --lsign-key key-goes-here
```

### Install GRUB boot loader
Update system and download and install GRUB.
```
pacman -Syu grub
grub-install --target=i386-pc /dev/sda
```
Generate `grub.cfg` using `grub-mkconfig -o /boot/grub/grub.cfg`

### Network config
Use `ip a` to get the interface names. Use 
`systemctl enable dhcpcd@enp0s3.service` to enable DHCP for the specified interface, 
`enp0s3`, on startup.

### Set root password
`passwd`

### Clean Up
If installed offline by copying from the archlive iso, be sure to follow the clean up 
mentioned above. You may also want to install `base` and `base-devel` since not all of these 
packages are included in the archiso.

### Unmount and reboot
```
exit
umount -R /mnt
shutdown
```
Remove archlive iso. Restart.

### Post Installation
[Arch General Recommendations](https://wiki.archlinux.org/index.php/General_recommendations)













