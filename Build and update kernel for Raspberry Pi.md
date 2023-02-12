# Build and update kernel for Raspberry Pi
---

首先需要一個可以開機的SD卡，Raspberry Pi OS 與燒錄SD卡的Tool可以在官方網站下載:

Download Raspberry Pi OS Imager and falsh imager tool:
```
https://www.raspberrypi.org/software/
https://www.raspberrypi.com/software/operating-systems/
https://roboticsbackend.com/enable-ssh-on-raspberry-pi-raspbian/
```

Raspberry Pi Imager 是官方的燒錄tool, 可以支援一些預燒設定, 像是 enable SSH, password 以及 Wi-Fi config.

建議使用下載的 image 保持一致性, 若未選則local image, Raspberry Pi Imager 預設會去下載最新的OS.

---

## Enable uart kanel log and login: 

1. 修改 SD 卡 `/boot/config.txt`\
在檔案最後增加以下設定:
```
[all]
enable_uart=1
force_turbo=1
dtoverlay=pi3-disable-bt
arm_freq=600
arm_freq_min=600
```
`enable_uart=1`: 啟用UART Terminal\
`force_turbo=1`: 強制保持CPU在最大頻率，固定頻率可以讓 Adapter的要求降低, 因為CPU瞬間拉高頻率需要大電流。\
`dtoverlay=pi3-disable-bt`: 禁用藍芽，因為藍芽默認使用UART0\
`arm_freq=600`: 設定CPU最大頻率為 600MHz, 降低此頻率可幾減少CPU發熱\
`arm_freq_min=600`: 設定CPU最小頻率為 600MHz

2. 修改 SD 卡 `/boot/cmdline.txt`, 刪除kanel參數中的 `quiet`, 可以使kanel log也出現在UART Terminal, Pi3 B+ 修改後如下:

```
console=serial0,115200 console=tty1 root=PARTUUID=f0b9305e-02 rootfstype=ext4 fsck.repair=yes rootwait splash plymouth.ignore-serial-consoles
```


3. 安裝 serial tool `minicom`,windows 推薦用 `MobaXterm`, 確認 baud rate 設為預設的 115200 (預設的baud rate 可以在SD 卡裡的 /boot/cmdline.txt 更改)

minicom安裝與使用:
```
sudo apt install minicom
sudo minicom -D /dev/ttyUSB0
```
接著上電後後就可以看到 kanel log 與登入提示了:


設定完成後, 上電時 UART 會輸出 kanel log, 開進OS後會顯示 Terminal:
```
[    8.359125] systemd[1]: Started Journal Service.

Raspbian GNU/Linux 11 raspberrypi ttyAMA0

raspberrypi login:
```

常用的`raspi-config`設定:
```
`Advanced Options` 執行 `Expand Filesystem`
`Interface Options` 啟用 `Serial Port`
`Interface Options` 啟用 `ssh`
```
---

## Cross-Compiling the Kernel
Use Ubuntu 22.04 LTS

## 事前準備
First install Git and the build dependencies:
```
sudo apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev
```

Download toolchain:

Install the 32-bit Toolchain for a 32-bit Kernel
```
sudo apt install crossbuild-essential-armhf
```
Install the 64-bit Toolchain for a 64-bit Kernel
```
sudo apt install crossbuild-essential-arm64
```


Prepare a folder to put the source code:
```
mkdir -p ~/raspberry_pi
cd ~/raspberry_pi
```

Download kernel source code:
```
git clone --depth=1 https://github.com/raspberrypi/linux
cd linux
```
---

## Setup kernel config
### Setup defconfig for 32-bit kernel
For Raspberry Pi 2, 3, 3+ and Zero 2 W, and Raspberry Pi Compute Modules 3 and 3+:
```
KERNEL=kernel7
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig
```
For Raspberry Pi 4 and 400, and Raspberry Pi Compute Module 4:
```
KERNEL=kernel7l
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
```

### Setup defconfig for 64-bit kernel
For Raspberry Pi 3, 3+, 4, 400 and Zero 2 W, and Raspberry Pi Compute Modules 3, 3+ and 4:
```
KERNEL=kernel8
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
```

載入預設的設定之後，可以再使用 menuconfig 來微調，可以在 General Setup 中的 Local Version 改一個自己的版本號，方便區分是否安裝成功：
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```
預設為`-v7` 我把它改成 `-v7-test`做示範.

修改可以在`.config`中看到:
```
CONFIG_LOCALVERSION="-v7-test"
```
---
## Start to build Kernel ,DeviceTree and Modules 
Building 32-bit Kernel:
```
make -j6 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs
```
Building 64-bit Kernel:
```
make -j6 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
```
編譯完成後，會產生 `arch/arm/boot/zImage` 檔案，這個就是kernel檔，64-bit Kernel檔名為Image.


---
## Install new kernel/modules/dtbs to Raspberry Pi via scp and ssh
建立build資料夾, 複製剛編譯完成的 kernel ,DeviceTree和 modules：
```
rm -r ../build/
mkdir -p ../build/
mkdir -p ../build/modules
cp arch/arm/boot/zImage ../build/kernel_1.img
cp arch/arm/boot/dts/bcm2710-rpi-3-b-plus.dtb ../build/bcm2710-rpi-3-b-plus_1.dtb
make -j6 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=../build/modules modules_install
```
壓縮 Linux modules，方便等一下複製的動作：
```
cd ../build
tar jcvf modules.tar.bz2 modules/
```
接著把 `kernel_1.img` ,`bcm2710-rpi-3-b-plus_1.dtb` 與 `modules.tar.bz2` 複製到 Raspberry Pi 中的 /tmp：
```
scp kernel_1.img pi@10.1.1.14:/tmp
scp modules.tar.bz2 pi@10.1.1.14:/tmp
scp bcm2710-rpi-3-b-plus_1.dtb pi@10.1.1.14:/tmp
```
在 Raspberry Pi ssh 上確認檔案已經成功傳輸:
```
pi@raspberrypi:~ $ ls /tmp/
bcm2710-rpi-3-b-plus_1.dtb  ssh-sr8F9OnvBBUk
dhcpcd-pi                   systemd-private-dfd00c9422aa459ebd96b3d4ab7dcb4d-ModemManager.service-aXIrLh
kernel_1.img                systemd-private-dfd00c9422aa459ebd96b3d4ab7dcb4d-systemd-logind.service-NhXMih
modules.tar.bz2             systemd-private-dfd00c9422aa459ebd96b3d4ab7dcb4d-systemd-timesyncd.service-Zw3yYg
ssh-QEEeej4KFia7
```
在 Raspberry Pi ssh 上解壓縮 modules ,並將 kernel ,dtbs 與 modules 放到正確的位置:
```
cd /tmp
tar jxf modules.tar.bz2
sudo cp -r modules/lib/modules/* /lib/modules/
sudo cp kernel_1.img /boot/kernel_1.img
sudo cp bcm2710-rpi-3-b-plus_1.dtb /boot/bcm2710-rpi-3-b-plus_1.dtb
```

編輯 `/boot/config.txt`:
```
sudo nano /boot/config.txt
```
指定pi3+要開啟的kernel檔名為kernel_1.img, device_tree為bcm2710-rpi-3-b-plus_1.dtb：
```
[pi3+]
kernel=kernel_1.img
device_tree=bcm2710-rpi-3-b-plus_1.dtb
```

接著在 UART Terminal 上重新啟動
```
sudo reboot
```
可以在 UART Terminal 上看到kernel log 版本已經替換成剛才建構的版本:
```
pi@raspberrypi:~$ sudo reboot
[ 4024.464461] reboot: Restarting system
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.15.61-v7-test+ (yuan@ax370m-gaming-3) (arm-linux-gnueabihf-gcc (Ubuntu 11.2.0-1
7ubuntu1) 11.2.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #4 SMP Sun Sep 4 14:15:25 CST 2022
[    0.000000] CPU: ARMv7 Processor [410fd034] revision 4 (ARMv7), cr=10c5383d
```

---
## Install new kernel/modules/dtbs to Raspberry Pi via SD card reader
一個灌好 Raspberry Pi OS 的 SD卡會有兩個分區，第一個分區是 FAT（boot），第二個分區是 ext4 文件系統（root）分區，準備要掛載SD卡的兩個分區如下:
```
mkdir -p /media/yuan/
mkdir -p /media/yuan/boot
mkdir -p /media/yuan/rootfs
```

接著插入SD卡，使用`mount`分別掛載兩個分區如下:
（boot可能會自動掛載，可以使用`umount`先卸載）

```
mount /dev/sda1 /media/yuan/boot
mount /dev/sda2 /media/yuan/rootfs
```
這裡的sda指的是第一個硬碟，如果電腦上有多個硬碟(or 隨身碟)可能會顯示在sdb或sdc等等，掛載兩個分區後可以用lsblk檢查，我這邊用16G的SD卡示範，格式應該要如下:
```
sda           8:0    1  14.5G  0 disk 
├─sda1        8:1    1   256M  0 part /media/yuan/boot
└─sda2        8:2    1  14.2G  0 part /media/yuan/rootfs
```

### 安裝剛 build 好的 modules 到 rootfs:
使用 `modules_install` 將 module 安裝到 rootfs:
```
sudo env PATH=$PATH make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/media/yuan/rootfs modules_install
```

### 複製剛 build 好的 32-Bit kernel 到 boot:
把剛 build 好的 kernel 複製到 boot ，並命名為`kernel_2.img`:
```
sudo cp arch/arm/boot/zImage /media/yuan/boot/kernel_2.img
```

### 複製剛 build 好的 dts 到 boot:
```
sudo cp arch/arm/boot/dts/*.dtb /media/yuan/boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /media/yuan/boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /media/yuan/boot/overlays/
```


使用不同的kernel文件名，例如`kernel_2.img`時，要在 config.txt 最後加入一行`kernel=`來註明要使用的 kernel image:
```
sudo gedit /media/yuan/boot/config.txt
# add kernel=kernel_2.img
```

最後卸載兩個分區:
```
sudo umount /media/yuan/boot
sudo umount /media/yuan/rootfs
```

替換後可以使用 dmesg 看 kernel log，可以看到 log 內顯示剛剛修改 Local Version 的版號 :

```
pi@raspberrypi:~ $ dmesg
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.15.61-v7-test+ (yuan@ax370m-gaming-3) (arm-linux-gnueabihf-gcc (Ubuntu 11.2.0-17ubuntu1) 11.2.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #3 SMP Sat Sep 3 17:37:25 CST 2022
```
也可以使用`uname -a`顯示版號 :
```
pi@raspberrypi:~$ uname -a
Linux raspberrypi 5.10.44-v7-yuan+ #6 SMP Sun Jun 27 14:11:52 CST 2021 armv7l GNU/Linux
```

如果要繼續修改source code，在Build前可以手動清除上次的make命令所產生的object文件:
```
make clean
```


---

## How to backup raspberry pi image in ubuntu
將SD卡放在讀卡器中, lsblk 確認 
```
sdd           8:48   1   7.2G  0 disk 
├─sdd1        8:49   1   256M  0 part /media/yuan/boot
└─sdd2        8:50   1     7G  0 part /media/yuan/rootfs
```

```
sudo dd bs=512 if=/dev/sdd of=raspberrypi_32bit_uaut_ssh_pi_1.img
```

```
sudo dd bs=512 if=raspberrypi_32bit_uaut_ssh_pi_1.img of=/dev/sdd
```




