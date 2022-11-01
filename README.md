# Artix Linux (OpenRC) Full-Disk Encryption


* [Preface](#preface)
* [Getting Started](#getting-started)
* [Installation](#installation)
  * [Configure Wi-Fi](#configure-wi-fi)
  * [Disk Partitioning](#disk-partitioning)
    * [Erase the disk](#erase-the-disk)
    * [Create the partitions](#create-the-partitions)
    * [Setup the logical volumes](#setup-the-logical-volumes)
    * [Format the partitions](#format-the-partitions)
    * [Mount the partitions](#mount-the-partitions)
  * [Install base system](#install-base-system)
  * [Configure base system](#configure-base-system)
    * [Localization](#localization)
    * [Add user](#add-user)
    * [GRUB](#grub)
    * [Other packages](#other-packages)
  * [First Boot](#first-boot)
  * [Troubleshooting](#troubleshooting)
* [What's next?](#whats-next)

## Preface
This guide covers everything to get up and running with a full-disk encrypted install of Artix Linux (OpenRC) on x86-64 hardware, featuring LVM on LUKS and an encrypted boot partition (GRUB).

For a quick overview, this is the disk partitioning scheme we are going to create:


```
 /dev/sdX - physical disk with MBR partition table
 /dev/sdX1 - encrypted with LUKS and partitioned into a LVM container
 
 Logical volume 1 - /dev/mapper/lvm-volBoot - /boot encrypted partition of 1 GB size
 Logical volume 2 - /dev/mapper/lvm-volSwap - swap partition, the size of which is >= size of your RAM (i.e. 16 GB)
 Logical volume 3 - /dev/mapper/lvm-volRoot - / root partition, which gets 100% of remaining free space
```

For additional informations and troubleshooting steps, refer to the official [Artix FDE installation guide](https://wiki.artixlinux.org/Main/InstallationWithFullDiskEncryption).

## Getting Started
1. Download the Artix Base OpenRC ISO [`artix-base-openrc.iso`](https://artixlinux.org/download.php)
2. Create a bootable USB drive with [dd](https://wiki.archlinux.org/title/Dd) on Linux or [Rufus](https://rufus.ie/en/) on Windows
3. Disable secure boot and enable legacy boot within your BIOS/UEFI
4. Boot from the USB drive
5. Change keyboard layout as needed
5. Select `From CD/DVD/ISO: artix.x86_64`
6. Log in with username `artix` and password `artix`
## Installation
Switch to the sudo user for the installation procedure
```bash
sudo su
```
### Configure Wi-Fi 
If not already connected via ethernet.

Enter connmanctl:
```bash
connmanctl
```
Enable Wi-Fi and scan for available access points
```
connmanctl> agent on
connmanctl> enable wifi
connmanctl> scan wifi
connmanctl> services
```
Connect to your Wi-Fi and enter password
```
connmanctl> connect wifi_....._managed_psk
```
Exit connmanctl
```
connmanctl> quit
```

### Disk Partitioning
Install parted package
```
pacman -Sy parted
```

#### Erase the disk
Get the **X** letter of your desired target installation drive:
```
 parted -l
```
Print its' partition table with

``` 
parted -s /dev/sdX print
```
Ensure that there is nothing important on this drive, then erase its' partition table and overwrite all its' contents with
```
 dd bs=4096 if=/dev/urandom iflag=nocache of=/dev/sdX oflag=direct status=progress || true
```
For security, **WAIT** until this lengthy process completes: otherwise, if you interrupt it with Ctrl+C/Z, the not-overwritten data will be possible to restore. Then, run
```
 sync
```
to flush the disk operations. Also, it is recommended to reboot after doing this, and launch a terminal with `sudo su` again.

#### Create the partitions
Create a new MBR partition table:
```bash
 parted -s /dev/sdX mklabel msdos
```
Make a **`/dev/sdX1`** partition which will take the whole space, and set the **boot** and **lvm** flags:
```
 parted -s -a optimal /dev/sdX mkpart "primary" "ext4" "0%" "100%"
 parted -s /dev/sdX set 1 boot on
 parted -s /dev/sdX set 1 lvm on
```
Print the partition table of a drive and see if the alignment of your partition is optimal:
```
 parted -s /dev/sdX print
 parted -s /dev/sdX align-check optimal 1
```

#### Setup the logical volumes
The disk encryption will utilize the **Linux Unified Key Setup** (*LUKS*), which is now part of an enhanced version of [cryptsetup](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Cryptsetup_usage), using [dm-crypt](https://wiki.archlinux.org/index.php/Dm-crypt) as the disk encryption backend.

To force loading the Linux kernel modules related to Serpent and other strong encryptions from your LiveCD/LiveUSB, run

```
cryptsetup benchmark
```
and, after it completes, use 

```
 cryptsetup --verbose --type luks1 --cipher serpent-xts-plain64 --key-size 512 --hash whirlpool --iter-time 10000 --use-random --verify-passphrase luksFormat /dev/sdX1
```
to create and format the LUKS partition with your custom encryption flags. Open and mount it using the device mapper - into i.e. **lvm-system** :

```
 cryptsetup luksOpen /dev/sdX1 lvm-system
```
**Note**: later you may encounter the following warnings - they happen because /run is not available inside the chroot - so you can ignore them:
```
 WARNING: Failed to connect to lvmetad. Falling back to device scanning.
 /run/lvm/lvmetad.socket: connect failed: No such file or directory
 WARNING: failed to connect to lvmetad: No such file or directory. Falling back to internal scanning.
```
Now it is possible to create a **physical volume** using the [Logical Volume Manager (**LVM**)](https://wiki.archlinux.org/index.php/LVM) and the previously used id lvm-system as follows:
```
 pvcreate /dev/mapper/lvm-system
```
Having the physical volume, it is possible to create a **logical volume group** named **lvmSystem** as follows:
```
 vgcreate lvmSystem /dev/mapper/lvm-system
```
And having the logical volume group, the **logical volumes** can be created as follows:
```
 lvcreate --contiguous y --size 1G lvmSystem --name volBoot
 lvcreate --contiguous y --size 16G lvmSystem --name volSwap
 lvcreate --contiguous y --extents +100%FREE lvmSystem --name volRoot
```
#### Format the partitions
Having all physical and virtual disk partitions ready, now it is possible to format them.

Format a **boot** partition with
```
 mkfs.fat -n BOOT /dev/lvmSystem/volBoot
```
Format a **swap** partition with
```
 mkswap -L SWAP /dev/lvmSystem/volSwap
```
This command will print a message like
```
 # Setting up swapspace version 1, size = 16 GiB (17179865088 bytes)
 # no label, UUID=6955244c-c72a-4dec-8dee-079ec743a818
```
**Copy your swap UUID somewhere** - you will need it later.

Format a **root** partition with
```
 mkfs.ext4 -L ROOT /dev/lvmSystem/volRoot
```

#### Mount the partitions
Having each partition formatted, they can be mounted as follows:
```
 swapon /dev/lvmSystem/volSwap
 mount /dev/lvmSystem/volRoot /mnt
 mkdir /mnt/boot
 mount /dev/lvmSystem/volBoot /mnt/boot
```
### Install base system
```bash
basestrap /mnt base base-devel openrc linux linux-firmware vim intel-ucode elogind-openrc #amd-ucode if you have an amd cpu
fstabgen -U /mnt >> /mnt/etc/fstab
```
Now, it is time to change root (*chroot*) to the newly installed environment:

```
 artix-chroot /mnt /bin/bash
```
Set up a **root** password with
```
passwd
```
Update the database of packages by running:
```
pacman -Sy
```
### Configure base system
#### Localization
Set the time zone:
```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime # e.g. /usr/share/zoneinfo/Europe/Berlin
```
Run **hwclock** to generate /etc/adjtime:
```bash
hwclock --systohc
```
Configure **locale** with
```
 echo -e "en_US.UTF-8 UTF-8" >> /etc/locale.gen
 locale-gen
 echo "LANG=en_US.UTF-8" > /etc/locale.conf
 export LANG="en_US.UTF-8"
 echo "LC_COLLATE=C" >> /etc/locale.conf
 export LC_COLLATE="C"
```
and a **hostname** with
```
 vi /etc/conf.d/hostname
 hostname="art"
```
#### Add user
```bash
useradd -mG wheel <your-username>
```
Configure password for new user
```bash
passwd <your-username> 
```

```bash
EDITOR=vi visudo
```
Uncomment the line starting with `# %wheel ALL=(ALL) ALL` . Then save and exit.

#### GRUB
For security, the standard Linux kernel and its' headers - should be replaced by the **hardened version**:
```
 pacman -Rc linux linux-headers
 pacman -S linux-hardened linux-hardened-headers
```

Now, you should install these packages:
```
 pacman -S lvm2 cryptsetup nano glibc mkinitcpio
 ```
During that, **initramfs** should be re-generated automatically with the **encrypt/resume** hooks. If not, re-generate **initramfs** manually:
```
 mkinitcpio -p linux-hardened
 ```
After that, a **grub** package could be installed with
```
 pacman -S grub
```
 
In order for a GRUB to find the LUKS-encrypted partitions, you will need to configure it:
```
vi /etc/default/grub
```

1. Change the GRUB default command line from:
 ```
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
 ```
 to

 ``` 
 GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=xxx:lvm-system loglevel=3 quiet resume=UUID=yyy net.ifnames=0"
 ```
 or, if you are using a solid-state disk (SSD) and would like to be able to enable Continuous TRIM on a later step, - with an additional allow-discards parameter:
 ```
  GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=xxx:lvm-system:allow-discards loglevel=3 quiet resume=UUID=yyy net.ifnames=0"
 ```
 where xxx UUID could be found out with

 ```
  blkid -s UUID -o value /dev/sdX1
 ```
 and **yyy** UUID - **swap** UUID - is already known by you from the previous steps.

2. Uncomment the following line to enable booting from LUKS encrypted devices
 ```
  GRUB_ENABLE_CRYPTODISK="y"
 ```
Then save and exit.

Install these optional dependencies:
```
 pacman -S dosfstools freetype2 fuse2 gptfdisk libisoburn mtools os-prober
 pacman -S iw memtest86+ wpa_supplicant
 pacman -S device-mapper-openrc lvm2-openrc cryptsetup-openrc
```
Then, you can install GRUB to MBR and generate its' config:
```
 grub-install --target=i386-pc --boot-directory=/boot --bootloader-id=artix --recheck /dev/sdX
 grub-mkconfig -o /boot/grub/grub.cfg
```

#### Other packages
In order to decrypt and use the LUKS/LVM volumes, the following services need to be installed and activated:
``` 
 rc-update add device-mapper boot
 rc-update add lvm boot
 rc-update add dmcrypt boot
``` 
The udev service (eudev/eudev-openrc) should be started by default in the sysinit runlevel. Its activation can be confirmed as follows:
``` 
 rc-status sysinit | grep udev
``` 
**should print this output**:
``` 
 udev                                                              [  stopped  ]
 udev-trigger                                                      [  stopped  ]
``` 
The **dbus** service should be installed and activated. Should it not, it can be done as follows:
``` 
 rc-update add dbus default
 ``` 
The SystemD project’s [logind](https://freedesktop.org/wiki/Software/systemd/logind/) should be installed as part of the base meta package. Should it not be activated, it can be done as follows:
``` 
 rc-update add elogind boot
``` 
The [haveged](https://issihosts.com/haveged/) service is a simple entropy daemon useful for unpredictable random number generation, which can be installed and activated as follows:
``` 
 pacman -S haveged haveged-openrc
 rc-update add haveged default
``` 

If [Network Manager](https://wiki.archlinux.org/index.php/NetworkManager) GUI **is the desired choice** to manage network interfaces, the following needs to be run in order to install and activate the service:
``` 
 pacman -S networkmanager networkmanager-openrc networkmanager-openvpn network-manager-applet
 rc-update add NetworkManager default
``` 
NTP, ACPI, Syslog-NG daemons can be installed and activated as follows:
``` 
 pacman -S ntp ntp-openrc acpid acpid-openrc syslog-ng syslog-ng-openrc
 rc-update add ntpd default
 rc-update add acpid default
 rc-update add syslog-ng default
``` 

Exit the **chroot** and unmount the volumes:
```
 exit
 umount -R /mnt
 swapoff -a
```

Flush the disk operations:
```
 sync
```

Now the system can be rebooted:
```
 reboot
```

### First Boot
During the first boot, you might get the following error:
```
 A password is required to access the lvm-system volume:
 cryptsetup: /usr/lib/libjson-c.so.5: no version information available (required by /usr/lib/libcryptsetup.so.12)
 Enter passphrase for /dev/sdX2:
```
Ignore this error, enter your passphrase and boot. Then open a console, run
```
 sudo su
```
to get the root rights, **enable swap**
```
 swapon /dev/lvmSystem/volSwap
```
and fully upgrade your system with
```
 pacman -Suy
```
After the upgrade completes, instead of instantly rebooting, please open mkinitcpio.conf with
```
vi /etc/mkinitcpio.conf
```
and make sure that all the HOOKS - especially **encrypt** --- have been preserved:
```
 HOOKS="base udev autodetect modconf block encrypt keyboard keymap consolefont lvm2 resume filesystems fsck"
```
If not - manually insert it, then run
```
 mkinitcpio -p linux-hardened
```
to apply this change. Now you can reboot safely and enjoy your encrypted Artix Linux.


### Troubleshooting

If after a system update, instead of
```
 A password is required to access the lvm-system volume:
 Enter passphrase for /dev/sdX2:
```
you are getting
```
 ERROR: device '/dev/mapper/lvmSystem-volRoot' not found. Skipping fsck.
```
maybe **encrypt** has disappeared from HOOKS of **mkinitcpio.conf**. To fix this, boot from LiveCD/LiveUSB and do these commands:
```
 sudo su
```
get root rights,
```
 cryptsetup benchmark
```
to force loading the Serpent-related Linux kernel modules,
```
 cryptsetup luksOpen /dev/sdX1 lvm-system
 mount /dev/lvmSystem/volRoot /mnt
 mount /dev/lvmSystem/volBoot /mnt/boot
```
mount the partitions,
```
 artix-chroot /mnt /bin/bash
```
enter the chroot ,
```
 vi /etc/mkinitcpio.conf
```
insert the **encrypt** to HOOKS ,
```
 HOOKS="base udev autodetect modconf block encrypt keyboard keymap consolefont lvm2 resume filesystems fsck"
 ```
and, finally,
```
 mkinitcpio -p linux-hardened
```
to apply the change.

## What's next?
If all went well, you should boot into your new system. Log in as your root to complete the post-installation configuration. See [Archlinux's general recommendations](https://wiki.archlinux.org/index.php/General_recommendations) for system management directions and post-installation tutorials.

If you have suggestions on improving this guide, or need help with the
installation, feel free to contribute/open issues.
