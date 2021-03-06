BSP Notes

### 编译U-boot, arm-trusted-firmware

U-boot相关文件共三个，分别是`trust.bin`, `uboot.img`, `idloader.bin`，具体可参考[wiki_Boot_option](http://opensource.rock-chips.com/wiki_Boot_option)。编译这三个文件需要ayufan-rock64的`rkbin`、`arm-trusted-firmware`、`linux-u-boot`三个池子

```bash
git clone https://github.com/ayufan-rock64/rkbin.git
cd rkbin
git checkout af17d09
cd ..

git clone https://github.com/ayufan-rock64/arm-trusted-firmware
cd arm-trusted-firmware
git checkout f947c7e
CROSS_COMPILE=aarch64-linux-gnu- sudo -E make PLAT=rk3328 all
# remove bl31.bin which can be 4+GiB in size and thus may fill the tmpfs mount
sudo rm build/rk3328/release/bl31.bin

# Build ATF: trust.bin
trust_merger trust.ini

cd ..
git clone https://github.com/ayufan-rock64/linux-u-boot.git
cd linux-u-boot
git checkout df02018479
# make uboot.img
./make.sh rock64-rk3328

# make idloader.bin
./rockdev/rk3328/out/tools/mkimage -n rk3328 -T rksd -d ../rkbin/rk33/rk3328_ddr_786MHz_v1.06.bin idbloader.bin
cat ../rkbin/rk33/rk3328_miniloader_v2.43.bin >> idbloader.bin

cd ..
mkdir u-boot
cp arm-trusted-firmware/trust.bin linux-u-boot/uboot.img linux-u-boot/idbloader.bin u-boot
```

三个文件需要写到镜像或SD卡特定位置

```bash
TARGET=/dev/sdc
sudo dd if=idbloader.bin of=$TARGET seek=64 conv=notrunc status=none
sudo dd if=uboot.img of=$TARGET seek=16384 conv=notrunc status=none
sudo dd if=trust.bin of=$TARGET seek=24576 conv=notrunc status=none
```

### 编译kernel

使用 ayufan-rock64/linux-kernel 池子的代码编译linux-kernel

```bash
git clone https://github.com/ayufan-rock64/linux-kernel.git
cd linux-kernel
git checkout ayufan-rock64/linux-build/0.6.45

# add drive WIFI_RTL88x2BU, see assets/add_config_for_rtl88x2bu.patch

# copy kernel-config
cp ./assets/linux-rk3328-default.config .config

# use .config to compile kernel
make -j4 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

# compile and make deb-pkg
make -j4 deb-pkg KBUILD_DEBARCH=arm64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

### create uInitrd

目前使用的uboot需要加载uInitrd，可以通过mkimage用`initrd.img`生成`uInitrd`

```bash
sudo mkimage -A arm64 -O linux -T ramdisk -C gzip -n uInitrd -d initrd.img-4.4.126+ uInitrd-4.4.126+
```

### 制作rootfs

需要已经准备的文件包括

+ ubuntu-base-16.04.5-base-arm64.tar.gz, [下载地址](http://cdimage.ubuntu.com/ubuntu-base/releases/16.04.5/release/)
+ linux-image-4.4.126-abel-arm64.deb, 自主编译的linux内核
+ uInitrd-4.4.126+, 事先生成的uInitrd
+ sources.list, apt源文件，目前使用的是中科大的ubuntu-ports的镜像
+ interfaces, 网络配置文件

具体制作过程包括，展开ubuntu-base、linux-kernel，chroot更新apt，安装需要的库，修改网络配置、添加用户，打包文件等

将文件拷贝至U盘，然后在ARM64平台的linux系统下运行脚本，即可得到`abel-rootfs-emmc-base.tar.gz`
```bash
sudo ./build-abel-rootfs-emmc-base.sh
```

### 分区与启动

设计上有三个分区，分别为P分区(2G)，A分区(6G)，B分区(6G)

P分区为保留分区，用于存储保留文件及配置启动选项。其中boot文件夹用于uuboot启动时判断进一步启动A分区还是B分区的系统。
boot文件夹下有5个文件，分别是

+ boot.scr: uboot启动时调用的命令
+ boot.cmd: boot.scr的源码，需要通过mkimage编译成boot.scr
```bash
mkimage -C none -A arm64 -T script -d /boot/boot.cmd /boot/boot.scr
```
+ envA.txt: 启动A分区的配置
+ envB.txt: 启动B分区的配置
+ armbianEnv.txt: envA.txt或envB.txt的link文件

### 制作SD卡
主要过程包括:

+ 分区和格式化
+ 写入U-boot相关文件
+ 拷贝boot文件夹至P分区
+ 展开rootfs至A分区，并更新`/etc/fstab`文件

以sd卡为/dev/sdc为例子，运行脚本

```bash
sudo ./prepare_sd.sh /dev/sdc
```

=============================

### minicom 直接连接

```bash
# Tx <---> Rx
# Rx <---> Tx
# GND <---> GND
# set baudrate=1500000

sudo apt install minicom
ls /dev/ttyUSB*
sudo minicom -D /dev/ttyUSB0 -c on
```

### 配置wifi驱动

```bash
# 查看已有驱动
find /lib -name *8188*

# 启动wifi驱动
modprobe -f 8188eu

# 查看驱动状态
dmesg
```

### 编译wifi驱动

参考[How to build a wireless driver](https://docs.armbian.com/User-Guide_Advanced-Features/#how-to-build-a-wireless-driver)

```bash
# build system with INSTALL_HEADERS="yes"

# make recordmcount, important !!
cd /usr/src/linux-headers-4.4.138-rk3328/scripts
make recordmcount

# build driver 88x2bu
cd
cd EW-7822ULC_Linux_Driver_1.0.1.6/
make

# Load driver for test
insmod 88x2bu.ko

# check driver
lsmod

# Plug the USB wireless adaptor and issue a command:
ifconfig
iwconfig wlx74da38d18b16

# Check which wireless stations / routers are in range
iwlist wlx74da38d18b16 scan | grep ESSID

# check saved Wireless config
cd /etc/NetworkManager/system-connections
ll

# install driver
make install
```

### enable network
```bash
ifconfig -a
ifconfig eth0 up
dhclient eth0
```

### 手动连接WIFI

```bash
# using nmcli

# show connection list
nmcli c show

# show device
nmcli d

# show wifi list
nmcli d wifi list

# connect to wifi
nmcli d wifi connect Xiaomi_123_5G password wisnuc123456

# disconnect wifi
nmcli device disconnect wlx74da38d18b16

# netwoking monitor
ip monitor

# wifi connected
# local 192.168.31.67 dev wlan0  table local  proto kernel  scope host  src 192.168.31.67

# wifi disconnected
# Deleted local 192.168.31.67 dev wlan0  table local  proto kernel  scope host  src 192.168.31.67

###############
# using ifconfig and iwconfig

# find wifi interface name
ifconfig -a
# turn on wifi
ifconfig wlx74da38d18b16 up
# scan wifi
iwlist wlx74da38d18b16 scan | grep ESSID
# connect to wifi
iwconfig wlx74da38d18b16 essid Xiaomi_123_5G key s:wisnuc123456
# enable dhcp
dhclient wlx74da38d18b16


# show saved wifi connection
ls -sail /etc/NetworkManager/system-connections/
```

### 设置自动连接WIFI

添加以下内容至 /etc/network/interfaces，其中wireless-ssid，wireless-key即所要连接的wifi和密码

```
auto wlx74da38d18b16
iface wlx74da38d18b16 inet dhcp
wireless-ssid SSID_Name
wireless-key XXXXX
```

### 网速测试

```bash
# check network info
ifconfig

# check network speed of enp1s0
ethtool enp1s0 | grep Speed

# install iperf in both client and station device
sudo apt install iperf

# station
iperf -s

# client
iperf -c station_IP
```

### 烧录工具

使用etcher工具烧录镜像至SD卡 [etcher](https://etcher.io/)

### 官方 Armbian 镜像

使用默认 Armbian Xenial 镜像，下载地址 [rock64](https://www.armbian.com/rock64/)

+ 该镜像默认root密码为1234，首次ssh登陆会要求重置密码，并添加用户，如winsuc(passwd: wisnuc)


### 编译Armbian

+ 推荐编译环境的配置：4G内存，SSD，四核CPU，20G以上硬盘空间

+ VirtualBox、Docker或者native环境的Ubuntu Bionic 18.04 x64系统

+ 管理员权限（sudo或root）

+ 执行代码

```bash
apt-get -y install git
git clone https://github.com/armbian/build
cd build
./compile.sh
```

注意编译所在环境不要有空格。首次编译过程需要下载大量内容，可能耗时数小时，之后的编译可在10min之内。

编译过程会提示选择编译类型、板子型号，内核分支及OS版本，具体可看[Build-Options](https://docs.armbian.com/Developer-Guide_Build-Options/)

目前分别选择

```
Full OS image for flashing,
Do not change the kernek configuration,
rock64,
xenial Ubuntu Xenial 16.04 LTS,
Image with console interface
```

亦可以通过配置文件config-default.conf自动设置，其中tag lock 对应commit `c63b212`

```
# Read build script documentation http://www.armbian.com/using-armbian-tools/
# for detailed explanation of these options and for additional options not listed here

KERNEL_ONLY="no"                  # leave empty to select each time, set to "yes" or "no" to skip dialog prompt
KERNEL_CONFIGURE="no"             # leave empty to select each time, set to "yes" or "no" to skip dialog prompt
CLEAN_LEVEL="make,debs,oldcache"  # comma-separated list of clean targets: "make" = make clean for selected kernel and u-boot,
                                  # "debs" = delete packages in "./output/debs" for current branch and family,
                                  # "alldebs" = delete all packages in "./output/debs", "images" = delete "./output/images",
                                  # "cache" = delete "./output/cache", "sources" = delete "./sources"
                                  # "oldcache" = remove old cached rootfs except for the newest 6 files

DEST_LANG="en_US.UTF-8"           # sl_SI.UTF-8, en_US.UTF-8

# advanced
KERNEL_KEEP_CONFIG="no"           # do not overwrite kernel config before compilation
EXTERNAL="yes"                    # build and install extra applications and drivers
EXTERNAL_NEW="prebuilt"           # compile and install or install prebuilt additional packages
CREATE_PATCHES="no"               # wait that you make changes to uboot and kernel source and creates patches
BUILD_ALL="no"                    # cycle through available boards and make images or kernel/u-boot packages.
                                  # set KERNEL_ONLY to "yes" or "no" to build all packages/all images

BSPFREEZE="no"                    # freeze armbian packages (u-boot, kernel, dtb)
INSTALL_HEADERS="yes"             # install kernel headers package
LIB_TAG="lock"                    # change to "branchname" to use any branch currently available.
CARD_DEVICE=""                    # device name /dev/sdx of your SD card to burn directly to the card when done
BOARD="rock64"                    # build for board rock64
RELEASE="xenial"                  # build xenial Ubuntu Xenial 16.04 LTS
BUILD_DESKTOP="no"                # Image with console interface
```

### 安装kexec-tools

```bash
# need enable Kernel Features -> kexec system call when compiling kernel
sudo apt install kexec-tools
kexec -l /boot/vmlinuz-4.4.138-rk3328 \
--initrd=/boot/initrd.img-4.4.138-rk3328 --reuse-cmdline
```

### Building a Root File System using BusyBox
```bash
# compile BusyBox
git clone https://github.com/mirror/busybox.git
cd busybox
git checkout 375951667
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
# Settings -> Build Options -> enable the option “Build BusyBox as a static binary (no shared libs)
# -> Installation Options (make install behavior) –> (./_install) : the root file system will be installed inside _install directory.
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- install
cd ./_install/
mkdir proc sys dev etc etc/init.d
touch etc/init.d/rcS
# edit etc/init.d/rcS
chmod +x etc/init.d/rcS

# mkdir boot, copy bootable boot/* to boot/

# create rootfs archives
find . | cpio -o --format=newc > ../rootfs.img
cd ..
gzip -c rootfs.img > rootfs.img.gz
```

+ Content of `etc/init.d/rcS`

```bash
#!bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```

### make full image

```bash
OFFSET=16                                       # MiB
BOOTSIZE=64                                     # MiB
bootstart=$(($OFFSET * 2048))                   # 32768
rootstart=$(($bootstart + ($BOOTSIZE * 2048)))  # 163840
bootend=$(($rootstart - 1))                     # 163839
ROOTSIZE=6 * 1024                               # MiB
rootend=$($rootstart + $ROOTSIZE - 1)           # 12746751

# parted sdcard, Sector size=512
sudo parted -s /dev/sdc -- mklabel msdo
sudo parted -s /dev/sdc -- mkpart primary ext4 32768s 163839s
sudo parted -s /dev/sdc -- mkpart primary ext4 163840s 12746751s
sudo parted -s /dev/sdc -- mkpart primary ext4 12746752s 25329663s
sudo parted -s /dev/sdc -- mkpart primary ext4 25329664s -1s
sudo dd if=idbloader.bin of=/dev/sdc seek=64 conv=notrunc status=none
sudo dd if=uboot.img of=/dev/sdc seek=16384 conv=notrunc status=none
sudo dd if=trust.bin of=/dev/sdc seek=24576 conv=notrunc status=none
```

### 配置Ubuntu base
```bash
# add fstab
sudo touch etc/fstab
echo -e \ 
"UUID=78facfa2-be63-4127-9386-c3ecd3921b28 / ext4 defaults,noatime,nodiratime,commit=600,errors=remount-ro 0 1\ntmpfs /tmp tmpfs defaults,nosuid 0 0" | sudo tee etc/fstab

# fix getty link
cd etc/systemd/system/getty.target.wants
sudo ln -s /lib/systemd/system/serial-getty@.service serial-getty@ttyS2.service
sudo ln -s /lib/systemd/system/getty@.service getty@ttyS2.service

# add kmod
sudo cp kmod bin
```



### 安装node

需要下载arm v8版本的[node](https://nodejs.org/dist/v8.11.3/node-v8.11.3-linux-arm64.tar.xz)

```bash
curl -O https://nodejs.org/dist/v8.11.3/node-v8.11.3-linux-arm64.tar.xz
```

### 其它相关命令

```bash
# 解压debs至特定文件夹
dpkg -x linux-image-rk3328_5.50_arm64.deb /tmp/image/

# 通过ARP协议获取到的网络上邻居主机的IP地址，用户获取板子的ip（也可以通过查看路由器连接设备的方式）
arp

# 查看本机器linux内核版本
cat /proc/version

# 查看linux版本
lsb_release -a

# 查看usb设备
lsusb

# 显示内核缓冲区系统控制信息
dmesg
```
