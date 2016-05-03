# Install ZFS Root File System

Notes on installing Arch onto a ZFS root file system on a VM.

I made these notes for my own reference and I hope they can help you too.

This guide requires an existing Arch install.
An Arch OS is necessary for creating a custom Arch Live ISO that contain the ZFS packages.

## Install Details
Although I have successfully used both, ZFS is a bit easier to install on an EFI system 
using rEFInd than on a BIOS system that uses GRUB.
You can enable EFI for your VM in Virtual Box under 
`VM Settings > System > Motherboard > Enable EFI`.
- UEFI
- GPT
- A 256MB Fat32 EFI boot partition and a ZFS partition
- rEFInd
- Guide made using the 2016.04.01 Arch release
- Linux Kernel 4.5.1, x86_64
- ZFSonLinux 6.5.7 (bleeding edge for kernel 4.5.1 ACL compatibility)
- Virtual Box 5.0.20 Running on Mac OS X 10.10.5 Host

## Disclaimer
These notes are ***not*** meant to serve as a complete how to guide to ZFS or Arch Linux. 

These are notes and tips that are meant to supplement the 
[Arch Wiki](https://wiki.archlinux.org/) and 
[Forum](https://bbs.archlinux.org/). 
Do ***not*** blindly follow these instructions. Both for the safety of your system
and in accordance with 
[The Arch Way](https://wiki.archlinux.org/index.php/Arch_Linux#Principles), 
if you don't understand what a step does I highly recommend that you stop and go look it up.

## Useful Links
Highly recommended reading.
- [Arch Wiki: Installation Guide](https://wiki.archlinux.org/index.php/installation_guide)
- [Arch Wiki: Beginners Guide](https://wiki.archlinux.org/index.php/beginners'_guide)
- [Arch Wiki: Getting and Installing Arch](https://wiki.archlinux.org/index.php/Category:Getting_and_installing_Arch)
- [Arch Wiki: FAQ](https://wiki.archlinux.org/index.php/Frequently_asked_questions)
- [Arch Wiki: ZFS](https://wiki.archlinux.org/index.php/ZFS)
- [Arch Wiki: Archiso](https://wiki.archlinux.org/index.php/archiso)
- [Arch Wiki: Installing Arch Linux on ZFS](https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_ZFS)
- [ZFS on Linux Webpage](http://zfsonlinux.org/)
- [ZFS on Linux GitHub](https://github.com/zfsonlinux/zfs)
- [Arch ZFS GitHub](https://github.com/archzfs/archzfs)

## Build Custom Arch Live Iso
Using an existing Arch Linux install...
### Install archiso
You'll need netcat later too.

`# pacman -Syu archiso netcat`

You may need to run the following to generate a pgp key and populate the default keys.
```
# pacman-key --init
# pacman-key --populate
```
### Copy Profile
Copy the `releng` profile for adding custom packages to the archiso.
```
$ mkdir ~/archlive
$ cp -r /usr/share/archiso/configs/releng/* ~/archlive
```
### Add ZFS Repo Server to Pacman
Add the [archzfs](https://wiki.archlinux.org/index.php/Unofficial_user_repositories#archzfs) 
repo.
Append following to file: `~/archlive/pacman.conf`
```
[archzfs]
Server = http://archzfs.com/$repo/x86_64
```
Import and sign the key for the repo.
```
# pacman-key -r 5E1ABF240EE7A126
# pacman-key --lsign-key 5E1ABF240EE7A126
```
### Add ZFS to Installed Package List
NOTE: At the time of writing this guide the stable ZFS on Linux release was 6.5.6. 
This version works on Linux Kernel 4.5.1 but 
[without Posix ACL support](https://github.com/zfsonlinux/zfs/issues/4537) 
which is required at boot for 
[systemd-tmpfiles-setup.service](https://wiki.archlinux.org/index.php/Systemd#systemd-tmpfiles-setup.service_fails_to_start_at_boot). 
The [patch](https://github.com/zfsonlinux/zfs/pull/4549) 
is included on the bleeding edge git repo and will be included in ZFS version 6.5.7. To install
the patched bleeding edge version replace `archzfs-linux` with `archzfs-linux-git` in the line 
below.

`$ echo "archzfs-linux" >> ~/archlive/packages.x86_64`

Note that the ZFS Arch Wiki specifies that it should be added to packages.both but in my 
experience this causes an "architecture not found" error on the build step because ZFS on 
Linux is only for 64 bit. So add the repo to `packages.x86_64`.

### Add Any Additional Files
Additional files can be added to `~/archlive/airootfs/`.

### Build Archlive Iso
```
$ cd ~/archlive
$ mkdir out
$ ./build.sh -v
```
The build can take a little while but shouldn't require any input or intervention.

### Export ISO to Host Computer
There are many ways to move files between your guest and host machine: shared folders, 
mounting a USB drive, the Internet. Below I describe how to use Netcat.
#### Netcat
Find your host ip address for the vbox internal subnet. Use `ifconfig`.
Make sure you have an internal Virtual Box subnet and a host-only network adapter. 
Shutdown and enable these settings if you do not have them enabled already.
I found that I needed to disable my primary network adapter inside the Arch Linux guest
to be able to reliably ping my host computer. Use `# ip link set enp0s3 down`. Test your 
connection by pinging the host.

On Mac OS X (the OS X netcat version doesn't use `-p`) host machine: `$ nc -l 1234 > archlinux-2016.04.28-zfs.iso`

On guest machine: `$ nc 192.168.56.1 1234 < ~/archlive/out/archlinux-2016.04.28.iso`

Wait until the network traffic goes silent and then kill netcat. Verify that the .iso on the host machine is the same size as the one on the guest machine.

## Install Arch Linux on ZFS Root Filesystem
This is a great place to take a snapshot if you are installing over your existing Arch install!

### Virtual Box Setup
What I used, (you can definitely get away with fewer system resources):
- EFI mode enabled!
- 4096 MB memory
- 16 GB virtual HD, dynamically allocated
- 2 CPU
- 2nd Host Only Network Adapter (optional)

### Boot From ISO
Boot into an EFI enabled VM from the .iso we just created. Booting from the Arch Live ISO in 
EFI mode can take awhile. After seleting the first option on the initial boot screen, the 
screen will go black for a while. Be patient. This process will go faster if you have 
additional CPUs enabled.

### Partitioning
A variety of partitioning schemes can be used but ZFS makes using seperate partitions somewhat 
pointless since you can use multiple datasets in a zpool to achieve the same result without the 
limitations of fixed partitions. Datasets can grow to be whatever size and different attributes
can be applied to each dataset. Research what partitioning scheme is best for you. Regardless, 
UEFI generally requires a seperate boot partition. 

Create a 256MB partition for EFI booting and a single remaining partition.

Type of EFI partition must be `ef00`.
Type of main partition must be `bf00` -- Solaris Root!

- Use `# lsblk` to list block memory devices
- Partition `/dev/sda`. Your device's name may be different.
- Use **gdisk** to create an MBR partition table
    - `# gdisk /dev/sda`
    - Use `o` to start a new table.
    - Use `n` to start a new partition.
    - Use default for "part-type": e.g. `primary`.
    - Enter `1MiB` for start of partition.
    - Enter `257MiB` for end of partition.
    - Use `t` for setting the partition type. Use `ef00`.
    - Use `n` to start a new partition.
    - Use defaults for part-type, start, and end of partition
    - Use `t` for setting the partition type. Use `bf00`.
    - Use `p` to print the table and review it.
    - Use `w` to write the partition table. 

THIS WILL EFFECTIVELY DESTROY ANY EXISTING DATA ON THE DRIVE.
- `# lsblk` will now list `/dev/sda1` and `/dev/sda2` under `/dev/sda`

### Format EFI Boot Partition
Format the boot partition to Fat 32. `# mkfs.fat -F32 /dev/sda1`

### Create The Root Zpool
This is the effective equivalent of "formatting" the second partition to ZFS.
Check for ZFS with `# modprobe zfs`, there should be no output. Then use

`# zpool create -f zroot /dev/disk/by-id/ID-TO-PARTITION-partx`

Always use block id names when working with ZFS, **not** `/dev/sda2`.

### Create The Datasets
Create datasets for ROOT and for data. This allows for seperation of the system files and the 
user files. This scheme can be used with `beadm` to manage boot environments. This can make 
upgrading the system easier since you can easily revert back if something goes wrong. Now add 
the default dataset in ROOT and a home dataset in data. 
```
# zfs create -o mountpoint=none zroot/ROOT
# zfs create -o mountpoint=none zroot/data
# zfs create -o mountpoint=/ zroot/ROOT/default
# zfs create -o mountpoint=/home zroot/data/home
```
In the "Installing Arch on ZFS" Arch Wiki page it specifies the mount point of `zroot/data/home`
to be `legacy`. This is simply done as an example of legacy datasets. Legacy datasets must 
mounted explicitly in the `/etc/fstab` so I don't recommend it unless the dataset must be 
mounted earlier in the boot process. After this install I experimented with using a separate
dataset for `/var`. This required me to set the dataset `mountpoint` to `legacy` and add an 
entry to the `/etc/fstab`.

### Set bootfs
This specifies the root file system to boot into. 
Set bootfs. `# zpool set bootfs=zroot/ROOT/default zroot`

When we initially created the pool it was automatically mounted inside the live filesystem. So
we must unmount it. Run `# zfs umount -a`.


### Export and Reimport
Before a zpool can be used by a different system it must be exported. This is a protection
against two systems writing to the same pool at the same time. If a zpool was not properly
exported, it can be force imported with the `-f` flag.

Now we will export the zroot pool and import it to /mnt. The `-d` argument specifies where ZFS
can find the disks by id.

```
# zpool export zroot
# zpool import -d /dev/disk/by-id -R /mnt zroot
```

### Mount EFI Partition
```
# mkdir /mnt/boot
# mount /dev/sda1 /mnt/boot
```

### Install Arch Linux
Use `# pacstrap /mnt base base-devel` 

OR use the following for offline install.

#### Offline install
If you use the offline install, extra configuration and clean up is required.
[Read here.](https://wiki.archlinux.org/index.php/Archiso#Installation_without_Internet_access)

Copy files and kernel image. (Offline install only)
```
# time cp -ax / /mnt
# cp -vaT /run/archiso/bootmnt/arch/boot/$(uname -m)/vmlinuz /mnt/boot/vmlinuz-linux
```

### Generate and edit fstab
The only disks required to be listed in `/etc/fstab` are the bootfs specified in the previous 
step, the EFI boot partition, any other non-ZFS partitions if they exist, and any ZFS datasets
with their `mountpoint` set to `legacy`. Later we will create a swap volume and that will be
added to the fstab also.

Generate the fstab using 

`# genfstab -U -p /mnt >> /mnt/etc/fstab`

Now edit `/mnt/etc/fstab` and comment or remove any lines that are not the EFI boot partition
or `zroot/ROOT/default`

### Change root
`# arch-chroot /mnt /bin/bash`

Now you are working within your new Arch system.

#### Offline install only
If you did an offline install, don't skip the steps specified 
[here](https://wiki.archlinux.org/index.php/Archiso#Installation_without_Internet_access)!

Complete the extra offline install config steps now before proceeding.

### Set hostname
`# echo MY_NEW_HOSTNAME > /etc/hostname`

Also append the new hostname to the two lines ending in `localhost` in the file `/etc/hosts`.

e.g. For the first line,
```
127.0.0.1   localhost.localdomain   localhost MY_NEW_HOSTNAME
```

### Set time zone
```
# ln -s /usr/share/YOUR/TIMEZONE /etc/localtime
# hwclock --systohc --utc
```

### Locale
Uncomment locales in `/etc/locale.gen` then run `locale-gen`. 

Create `/etc/locale.conf` and add,
```
LANG=en_US.UTF-8
```

### Generate Ramdisk Environment
The proper hooks for ZFS need to be set in `/etc/mkinitcpio.conf`.
Find the following line and change it to match,
```
HOOKS="base udev autodetect modconf block keyboard zfs filesystems"
```
Now generate the ramdisk environment,

`# mkinitcpio -p linux`

### Set up pacman keys
You may need to run the following to generate a pgp key and populate the default keys, 
```
# pacman-key --init
# pacman-key --populate
```
If you need to sign a new key for a package from the AUR,
```
# pacman-key -r key-goes-here
# pacman-key --lsign-key key-goes-here
```

### Install rEFInd
```
# pacman -Syu refind-efi
# refind-install --usedefault /dev/sda1
```
Now we need to add edit the kernel parameters to let the kernel know about the root filesystem.
Edit `/boot/refind_linux.conf` to match the following

```
"Boot with standard options"    "ro quiet zfs=bootfs"
```

Really the only important things in that line are `ro` and `zfs=bootfs`. The string at the 
beginning is just a label for the boot option with this set of kernel parameters. 

You can remove the word `quiet` if you want to see all the status messages during boot.

### Set Zpool cache
This file tells ZFS what to automatically mount. Set it up with, 
```
# zpool set cachefile=/etc/zfs/zpool.cache zroot
```

### Enable ZFS Mounting At Boot
```
# systemctl enable zfs.target
```

### Enable POSIX ACL
```
# zfs set acltype=posixacl zroot/ROOT/default
# zfs set aclinherit=passthrough zroot/ROOT/default
```
From what I have read it is not necessary to edit the fstab to reflect this because the 
`acl/noacl` setting is ignored.

### Network config
Use `ip a` to get the interface names. Use 
`# systemctl enable dhcpcd@enp0s3.service` to enable DHCP for the specified interface, 
`enp0s3`, on startup.

### Set root password
`# passwd`

### Offline Install
If you did an offline install, you may also want to install `base` and `base-devel` since not 
all of these packages are included in the archiso.
```
# pacman -Syu base base-devel
```

### Unmount and reboot
Don't forget to unmount and export the zpool. That exporting step is important!
```
# exit
# umount /mnt/boot
# zfs umount -a
# zpool export zroot
# shutdown
```
Remove archlive iso. Restart.

### Post Installation
Verify that everything booted properly with `# systemctl --failed`.

- [Arch General Recommendations](https://wiki.archlinux.org/index.php/General_recommendations)
- [Install the Virtual Box Guest Additions](https://wiki.archlinux.org/index.php/VirtualBox#Install_the_Guest_Additions)

## Troubleshooting

### Not booting
If it won't boot you'll have to boot back into the arch live iso and remount the file system to
troubleshoot. See **Export and Reimport** for mounting the ZFS pool again. Once you do that you
can `arch-chroot` back into the filesystem.

I recommend checking the following,
- Is the zpool cachefile set?
- Is the fstab correct?
- Is rEFInd installed and are the kernel parameters set?

### ACL errors
[Issue with ACL operations](https://github.com/zfsonlinux/zfs/issues/4537)

Go back and read my description of this issue in **Add ZFS to Installed Package List** at the 
top. If you installed ZFS 6.5.6, you need 6.5.7 which might still be the bleeding edge version.
You can install a different version after the fact using,
```
# pacman -Syu --force archzfs-linux-git
```
If you do this **be sure to reset the zfs cachefile!** See **Set Zpool cache** above.



