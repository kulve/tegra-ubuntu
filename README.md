INSTALLING UBUNTU RARING TO OUYA
================================

This is tested on Debian Wheezy and mostly adapted from https://github.com/kulve/tegra-debian

*OUYA IS EASILY BRICKABLE. READ NO FURTHER*

That said, the goal is not to flash anything on Ouya. Kernel is booted from memory and Ubuntu from USB stick or SD card.

Distribution issues
-------------------
* Debian Wheezy: Segfault in OMX_UseEGLImage with XBMC.
* Debian Jessie: X.Org Video ABI 15, no driver.
* Ubuntu 12.04: Old.
* Ubuntu 12.10: Old.
* Ubuntu 13.04: Works.
* Ubuntu 13.10: X.Org Video ABI 14, rendering problems.
* Ubuntu 14.04: X.Org Video ABI 15, no driver.

Known issues
------------
* Not properly tested, so there is a bunch unknown issues.
* Low-power core doesn't work (kernel crash)
    * CPUfreq with ondemand governer works though.
* Gstreamer usually assumes xvimagesink as the video sink, but nvxvimagesink must be used.
    * Totem obeys gconf: gconftool-2  -s /system/gstreamer/0.10/default/videosink nvxvimagesink --type=string
* Wifi firmware binaries not included, they need to be copied from the Android rootfs.

Setting up the rootfs
---------------------

### Prepare a USB stick ###

Partition an USB stick (I used SD card in a small USB reader) for EXT4. I recommend using at least 8GB stick.

Use `mkfs.ext4` to initialise the partition. If your system is properly set you shouldn't need sudo for that while you would need sudo to format your actual root partition.

### Mount the USB stick: ###

It is possible to create chroot to a directory and copy the contents later, or inside a loopback device and use dd. These instructions creates the chroot directly to the USB stick.

Change the sdX1 below to match your setup.

    export TARGET=/mnt/rootfs
    sudo mkdir -p $TARGET
    sudo mount /dev/sdX1 $TARGET

### Extract base system packages to the USB stick: ###
    sudo debootstrap --verbose --arch armhf --foreign raring $TARGET http://ports.ubuntu.com/

### Prepare for chroot: ###
    sudo apt-get install qemu-user-static binfmt-support
    sudo cp /usr/bin/qemu-arm-static $TARGET/usr/bin
    sudo mkdir $TARGET/dev/pts
    sudo modprobe binfmt_misc
    sudo mount -t devpts devpts $TARGET/dev/pts
    sudo mount -t proc proc $TARGET/proc

### Finish the base system installation: ###
    sudo chroot $TARGET

You should see `I have no name!@hostname:/#`

    /debootstrap/debootstrap --second-stage

At the end, you should see `I: Base system installed successfully.`

Configuring rootfs while still in chroot
----------------------------------------

### Setup sources.list: ###
    cat <<END > /etc/apt/sources.list
    deb http://ports.ubuntu.com/ raring main multiverse universe restricted
    deb-src http://ports.ubuntu.com/ raring main multiverse universe restricted
    END

    apt-get update

### Configure language: ###
    export LANG=C
    apt-get install apt-utils dialog language-pack-en
    export LANG=en_GB.UTF-8

### Install some important stuff: ###
    apt-get install openssh-server wget curl ntp vim nano mc zip wireless-tools bluez rfkill

Hack around failing DBUS invoke:

    ln -s /bin/true /usr/local/sbin/invoke-rc.d
    apt-get install dbus
    rm /usr/local/sbin/invoke-rc.d

### Configure ethernet with dhcp and set hostname: ###
    cat <<END > /etc/network/interfaces
    auto lo eth0
    iface lo inet loopback
    iface eth0 inet dhcp
    END

    echo ouya > /etc/hostname

### Create filesystem mounts: ###

    cat <<END > /etc/fstab
    # /etc/fstab: static file system information.
    #
    # <file system> <mount point>   <type>  <options>       <dump>  <pass>
    /dev/root      /               ext4    noatime,errors=remount-ro 0 1
    tmpfs          /tmp            tmpfs   defaults          0       0
    /.swap      none            swap    sw                0       0
    END

Optionally create 512M sawp file:

    dd if=/dev/zero of=/.swap bs=1M count=512
    mkswap /.swap

### Activate remote console: ###
    echo 'T0:2345:respawn:/sbin/getty -L ttyS0 115200 linux' >> /etc/inittab

### Set root passwd: ###
    passwd

### Add normal user with sudo rights: ###
    adduser ouya
    adduser ouya video
    adduser ouya audio
    adduser ouya plugdev
    adduser ouya sudo

### Install Slim login manager, XFCE, and Totem: ###
    apt-get install xfce4 xfce4-goodies totem midori slim gstreamer0.10-plugins-good gstreamer0.10-alsa gstreamer0.10-plugins-good

### Install Tegra 3 proprietary binaries and Ouya config files: ###
    dpkg -i tegra30-r16_*_armhf.deb

### Disable core hotplugging

    vi /etc/init.d/ondemand

Change the options to match the following:

    echo 1 >/sys/module/cpu_tegra3/parameters/no_lp
    echo 0 >/sys/module/cpu_tegra3/parameters/auto_hotplug
    echo 0 >/sys/module/cpuidle/parameters/lp2_in_idle

### Finish up with the chroot: ###

Log out from the chroot and extract kernel modules to the target:

    sudo mkdir -p $TARGET/lib/modules/
    sudo tar zxf modules-3.1.10-tk*.tar.gz -C $TARGET/lib/modules/

Kill any process started in the chroot (`lsof $TARGET`) and finally unmount the target:

    sudo umount $TARGET

### Install adb and fastboot to the host Debian: ###
    sudo dpkg -i android-tools*deb

Booting Ouya
------------

### Reboot Ouya to fastboot: ###
    adb reboot-bootloader

### Boot Ouya with the kernel: ###
*WARNING: NEVER EVER FLASH THE KERNEL, JUST BOOT FROM RAM*

    fastboot boot zImage-3.1.10-tk*

Wifi
----

The BCM firmware binaries may not be redistributable so they need to be copied from the Android rootfs after booting to Debian:

    sudo mount -o ro /dev/mmcblk0p3 /mnt/
    sudo mkdir /lib/firmware/bcm4330/
    sudo cp /mnt/etc/nvram_4330.txt /lib/firmware/
    sudo cp /mnt/vendor/firmware/bcm4330/fw_bcmdhd.bin /lib/firmware/bcm4330/
    sudo cp /mnt/etc/firmware/bcm4330.hcd /lib/firmware/bcm4330/
    sudo umount /mnt

    sudo modprobe bcmdhd

Bluetooth
---------

The following command should set up BT but I have not got it working yet:
    sudo brcm_patchram_plus --enable_hci --use_baudrate_for_download --scopcm=0,2,0,0,0,0,0,0,0,0  --baudrate 3000000 --patchram /lib/firmware/bcm4330/bcm4330.hcd --no2bytes --enable_lpm --tosleep=50000 /dev/ttyHS2 &
