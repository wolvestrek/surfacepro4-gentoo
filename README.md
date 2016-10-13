This How To is inspired by the well-done [Installing Debian on the Microsoft Surface Pro 4](https://github.com/jimdigriz/debian-mssp4) instructions.

# What is working?

 * NVMe solid state drive including ext4 encryption

# Installation steps

 * [Disable secure boot](https://github.com/jimdigriz/debian-mssp4#prepping-windows-10)
 * Boot a [SystemRescueCd](http://www.system-rescue-cd.org/SystemRescueCd_Homepage)
 * [Preparing your NVMe disk](#preparing-your-nvme-disk)
 * [Downloading the latest stage3](#downloading-the-latest-stage3)
 * [Entering your new system](#entering-your-new-system)
 * [Configuring the kernel](#configuring-the-kernel)
 * [Installing the base system](#installing-the-base-system)
 * [Setup the bootloader](#setup-the-bootloader)
 * [Finalizing the base installation](#finalizing-the-base-installation)

# Preparing your NVMe disk

To use the whole disk, like I do, replace everything on the disk with random bytes.

**Warning: This command will remove all your data from your HDD including your Windows recovery. If you do not want this I recommend you to [shrink your Windows partition](https://github.com/jimdigriz/debian-mssp4#shrinking-the-windows-partition).**

    shred -n 1 -v /dev/nvme0n1

Now create a new parition scheme.

Partition | Filesystem | Size
--- | --- | ---
/boot | vfat | 512M
swap | swap | 8G or more if you want suspend to disk support
/ | ext4 | Rest of the disk

To create the filesystems use the following commands

    mkfs.fat -F 32 /dev/nvme0n1p1
    mkswap /dev/nvme0n1p2
    mkfs.ext4 /dev/nvme0n1p3
    
and to mount them use

    swapon /dev/nvme0n1p2
    mount /dev/nvme0n1p3 /mnt/gentoo
    mkdir /mnt/gentoo/boot
    mount /dev/nvme0n1p1 /mnt/gentoo/boot
    
# Downloading the latest stage3

Before you download the latest stage3 archive double check your system's datetime

    date
    
and connect to the internet

    nmtui
    
then download the archive

    cd /mnt/gentoo
    wget http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-20161006.tar.bz2
    tar xvjpf stage3-amd64-20161006.tar.bz2

# Entering your new system

Mouting some needed filesystems, if you want to know more about it see [Mounting the necessary filesystems](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Base#Mounting_the_necessary_filesystems)

    mount -t proc none /mnt/gentoo/proc
    mount --rbind /sys /mnt/gentoo/sys
    mount --rbind /dev /mnt/gentoo/dev

Copy DNS information

    cp /etc/resolv.conf /mnt/gentoo/etc

Entering the new system

    chroot /mnt/gentoo /bin/bash
  
Initialising the portage tree  
  
    emerge-webrsync
    emerge --sync

# Configuring the kernel

I recommend to use the [pf-sources](https://pf.natalenko.name) for later performance optimization

    emerge -a sys-kernel/pf-sources
    
For now only add some required modules

    cd /usr/src/linux
    make menuconfig
    
    Processor type and features --->
      [*] EFI runtime service support
      [*]   EFI stub support
      
    Device Drivers --->
      <*> NVM Express block device
      [*] Network device support --->
        [*] Wireless LAN --->
          [*] Marvell devices
            [*] Marvell WiFi-Ex Driver
              [*] Marvell WiFi-Ex Driver for PCIE 8766/8897/8997
      [*] USB support --->
        <*> xHCI HCD (USB 3.0) support
        
compiling and installing

    make -j4
    make install
    make modules_install
    
# Installing the base system

In the first step install just some basic packages

    emerge -a efibootmgr networkmanager terminus-font
    
Adjust /etc/fstab, /etc/conf.d/keymaps, /etc/conf.d/hostname and /etc/conf.d/consolefont and make sure that the appropriate services will be started next reboot
    
    rc-update add consolefont boot
    rc-update add NetworkManager default
    
    passwd
    
# Setup the bootloader

You have to exit the chroot envirmonment first because efibootmgr fails to determine the right disk UUID

    efibootmgr -c -d /dev/nvme0n1p1 -l /vmlinuz-4.5.0-pf4 -L "Gentoo Linux" -u "root=/dev/nvme0n1p3"
    
# Finalizing the base installation

    exit
    cd ~
    rm /mnt/gentoo/stage3-*.tar.bz2
    umount -lR /mnt/gentoo
    reboot
    
**If everything goes right you now should have a working base systen**
