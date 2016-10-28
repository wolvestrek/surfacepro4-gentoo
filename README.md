This How To is inspired by the well-done [Installing Debian on the Microsoft Surface Pro 4](https://github.com/jimdigriz/debian-mssp4) instructions.

# What is working?

 * NVMe solid state drive including Ext4 Encryption

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

# Optimizations

 * [Re-compiling the system with custom CFLAGS](#re-compiling-the-system-with-custom-cflags)
 * [Optimized CPU_FLAGS_X86](#optimized-cpu_flags_x86)
 * [Systemd](#systemd)

# Security

 * [Ext4 encrypted home directory](#ext4-encrypted-home-directory)

# Preparing your NVMe disk

To use the whole disk, like I do, replace everything on the disk with random bytes

**Warning: This command will remove all your data from your HDD including your Windows recovery. If you do not want this I recommend you to [shrink your Windows partition](https://github.com/jimdigriz/debian-mssp4#shrinking-the-windows-partition)**

    shred -n 1 -v /dev/nvme0n1

Now create a new parition scheme

Partition | Filesystem | Size
--- | --- | ---
/boot | vfat | 512M
swap | swap | 8G or more if you want suspend to disk support
/ | ext4 | Rest of the disk

To create the filesystems use the following commands

    mkfs.vfat -F 32 /dev/nvme0n1p1
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
            [M] Marvell WiFi-Ex Driver
              [M] Marvell WiFi-Ex Driver for PCIE 8766/8897/8997
      [*] USB support --->
        <*> xHCI HCD (USB 3.0) support
        
compiling and installing

    make -j5
    make install
    make modules_install
    
# Installing the base system

In the first step install just some basic packages

    emerge -a efibootmgr linux-firmware networkmanager terminus-font
    
Adjust /etc/fstab, /etc/conf.d/keymaps, /etc/conf.d/hostname and /etc/conf.d/consolefont and make sure that the appropriate services will be started next reboot
    
    rc-update add consolefont boot
    rc-update add NetworkManager default
    
    passwd
    
# Setup the bootloader

You have to exit the chroot envirmonment first because efibootmgr fails to determine the right disk UUID

    efibootmgr -c -d /dev/nvme0n1p1 -l /vmlinuz-4.5.0-pf4 -L "Gentoo Linux" -u "root=/dev/nvme0n1p3"

I recommend to create a boot entry for *old* kernel also

    efibootmgr -c -d /dev/nvme0n1p1 -l /vmlinuz-4.5.0-pf4.old -L "Gentoo Linux (recovery)" -u "root=/dev/nvme0n1p3"

# Finalizing the base installation

    exit
    cd ~
    rm /mnt/gentoo/stage3-*.tar.bz2
    umount -lR /mnt/gentoo
    reboot
    
**If everything goes right you now should have a working base systen**

# Re-compiling the system with custom CFLAGS

Edit your */etc/portage/make.conf*

    CFLAGS="-march=native -O2 -pipe"
    CXXFLAGS="${CFLAGS}"

after that update GCC to version 5.4, maybe you have to unmask the *~amd64* flag

    emerge -u gcc
    
To re-compile the system do

    cd /usr/portage/scripts
    ./bootstrap.sh
    
run it twice to ensure that everything in the toolchain have been rebuilt using the new compiler

    ./bootstrap.sh
    
Now you can rebuild the system

    emerge --emptytree --with-bdeps=y @world

# Optimized CPU_FLAGS_X86

You can determine them by emerging *app-portage/cpuid2cpuflags* and add them to your */etc/portage/make.conf*

    CPU_FLAGS_X86="aes avx avx2 fma3 mmx mmxext popcnt sse sse2 sse3 sse4_1 sse4_2 ssse3"

after that recompile all affected packages

    emerge -DN @world

# Systemd

There's a very good documentation how to install [Systemd on Gentoo](https://wiki.gentoo.org/wiki/Systemd). In short you have to reconfigure your kernel, append *systemd* to the USE variable and update packages

    emerge -uDN @world
    
After that create new boot entries

    efibootmgr -c -d /dev/nvme0n1p1 -l /vmlinuz-4.5.0-pf4 -L "Gentoo Linux (systemd)" -u "init=/usr/lib/systemd/systemd root=/dev/nvme0n1p3 quiet"
    efibootmgr -c -d /dev/nvme0n1p1 -l /vmlinuz-4.5.0-pf4 -L "Gentoo Linux (systemd, debug)" -u "init=/usr/lib/systemd/systemd root=/dev/nvme0n1p3"
    efibootmgr -c -d /dev/nvme0n1p1 -l /vmlinuz-4.5.0-pf4.old -L "Gentoo Linux (systemd, recovery)" -u "init=/usr/lib/systemd/systemd root=/dev/nvme0n1p3"

alternative you can use Systemd's own boot manager, recompile the systemd package with *gnuefi* USE flag and [configure the loader entries](https://wiki.archlinux.org/index.php/systemd-boot) in /boot/loader, then

    bootctl install
    reboot

Now Gentoo should be restarted with the new Systemd init system so you can

    systemd-machine-id-setup
    hostnamectl set-hostname <HOSTNAME>
    localectl set-keymap <KEYMAP>
    timedatectl set-timezone <TIMEZONE>
    echo "FONT=ter-132n" >> /etc/vconsole.conf

    systemctl enable NetworkManager.service
    systemctl start NetworkManager.service

and reboot to double-check that everything is working fine with Systemd

    systemctl reboot

# Ext4 encrypted home directory

Enable Ext4 Encryption in the kernel

    File Systems --->
    <*> The Extended 4 (ext4) filesystem
    <*>   Ext4 Encryption

reboot the new kernel and make sure that you have at least version 1.43 of e2fsprogs installed, maybe you have to unmask *~amd64* for e2fsprogs

    emerge -u e2fsprogs

Now you can active Ext4 Encryption

    tune2fs -O encrypt /dev/nvme0n1p3
    
