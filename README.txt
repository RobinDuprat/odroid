================================================================================
== README
================================================================================
1] download sources
--------------------------------------------------------------------------------
$ cd ~ && mkdir odroid && cd odroid
$ git clone --depth 1 --single-branch -b odroidxu4-4.9.y https://github.com/hardkernel/linux
$ git clone https://github.com/raspberrypi/tools
$ export PATH=$PATH:$(pwd)/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian/bin/

In raspberrypo/tools you will find the gcc-linaro-arm-linux-gnueabihf toolchain
Also, can be install on system using APT :
$ sudo apt-get install gcc-arm-linux-gnueabihf

2] Build sources
--------------------------------------------------------------------------------
$ cd linux
$ export ARCH="arm" CROSS_COMPILE="arm-linux-gnueabihf-"
$ make odroidxu4_defconfig        # default configuration for odroid xu4
$ make menuconfig                 # active earlyprink
  > Kernel debugging : ON
  > Kernel low-level debugging functions (read help!) : ON
  > Early printk : ON
  > [SAVE]
$ make -j 8 zImage dtbs modules   # get zImage, device tree and kernel modules

3] Format SD card
--------------------------------------------------------------------------------
$ sudo umount /dev/mmcblk0p* # makes sure it's not mounted
$ sudo parted -s /dev/mmcblk0 \
  mklabel msdos \
  mkpart primary fat32 1M 100M \
  mkpart primary ext4 100M 250M
$ sudo mkfs.vfat /dev/mmcblk0p1 # /boot
$ sudo mkfs.ext4 /dev/mmcblk0p2 # /root
$ sync

4] Create & Install Busybox rootfs
--------------------------------------------------------------------------------
$ sudo mkdir /mnt/boot /mnt/root -p
$ sudo mount /dev/mmcblk0p2  /mnt/root
$ cd ~/odroid
$ git clone git://busybox.net/busybox.git && cd busybox
$ make defconfig
$ make menuconfig
  > Setting               > Build Options     > static binary (no shared libs)
  > Installation Options  > Destination path  > "/mnt/root"
  [EXIT & SAVE]
$ sudo make install       # use sudo for /mnt/root write permissions
$ cd /mnt/root && ls      # See mising some common kernel linux folders
=> bin  linuxrc  lost+found  sbin  usr
$ sudo mkdir proc sys dev etc etc/init.d
# Add bash script in order to perform userland operations after the kernel boot
$ edit etc/init.d/rcS
=>
  #!/bin/sh

  mount -t proc none /proc

  mount -t sysfs none /sys

  /sbin/mdev -s
<=
$ chmod +x etc/init.d/rcS

5] Install Kernel
--------------------------------------------------------------------------------
$ sudo mkdir /mnt/boot /mnt/root -p
$ sudo mount /dev/mmcblk0p1 /mnt/boot
$ cd ~/odroid/linux
$ sudo cp arch/arm/boot/zImage arch/arm/boot/dts/exynos5422-odroidxu4.dtb /mnt/boot
$ sudo make modules_install INSTALL_MOD_PATH=/mnt/root
$ sudo make headers_install INSTALL_HDR_PATH=/mnt/root/usr
$ krel=`make kernelrelease`
$ sudo cp .config /mnt/boot/config-${krel}
$ cd /mnt/boot
$ sudo mkinitramfs -k -oinitrd.img-${krel} ${krel}
# APT Install :  u-boot-tools
$ sudo mkimage -A arm -O linux -T ramdisk -a 0x0 -e 0x0 -n initrd.img-${kver} -d
initrd.img-${kver} uInitrd-${kver}


6.1] Install U-boot (Mainline) [FAIL]
--------------------------------------------------------------------------------
$ git clone git://git.denx.de/u-boot.git
$ cd u-boot
$ git checkout v2017.11 # become the current master need gcc6 and I have gcc5
$ make odroid-xu4_config
$ make -j4
$ dd if=u-boot.bin of=/dev/mmcblk0 seek=63
$ cd /mnt/boot
$ edit boot.txt
=>
  setenv initrd_high "0xffffffff"
  setenv fdt_high "0xffffffff"
  setenv fb_x_res "1920"
  setenv fb_y_res "1080"
  setenv hdmi_phy_res "1080"
  setenv bootcmd "fatload mmc 0:1 0x40008000 zImage; bootm 0x40008000"
  setenv bootargs "console=tty1 console=ttySAC1,115200n8 fb_x_res=${fb_x_res}
  fb_y_res=${fb_y_res} hdmi_phy_res=${hdmi_phy_res} root=/dev/mmcblk0p2 rootwait
  ro mem=2047M"
  boot
<=
$ mkimage -A arm -C none -T script -d boot.txt boot.scr

# See https://wiki.odroid.com/old_product/odroid-xu3/software/bootloader
# Boot sequencee : iROM->BL1->(BL2 + TrustZone)->U-BOOT

6.2] Install U-boot with iRom, BL1, BL2
--------------------------------------------------------------------------------
DOWNLOAD:
-http://dev.odroid.com/projects/4412boot/wiki/FrontPage?action=download&value=boot.tar.gz
-http://odroid.in/guides/ubuntu-lfs/boot.tar.gz
EXTRACT and FLASH:
$ cd ~/odroid
$ tar -xf boot.tar.gz . && cd boot
$ cp ~/odroid/u-boot/u-boot.bin .
$ ./sd_fusing.sh <device/path/of/your/card> # not partition /!\ warning disk op.




