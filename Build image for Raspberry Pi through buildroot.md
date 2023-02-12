# Build image for Raspberry Pi through buildroot
---

clone buildroot repository 
```
mkdir -p ~/buildroot
cd ~/buildroot
git clone https://github.com/buildroot/buildroot.git 
cd buildroot
```
(option) check to other branch if you want
```
git checkout remotes/origin/2022.11.x
```

(option) list defconfigs, you can see raspberrypi3_defconfig for raspberry pi 3
```
make list-defconfigs |grep raspberry
# .. raspberrypi3_defconfig ..
```
set current config to raspberrypi3 default config
```
make raspberrypi3_defconfig
```

(option) config some useful config
```
make menuconfig 
```

(option) useful config:
```
system configuration ---> set root password "1"
Toolchain ---> Install glibc utilities
Development tools  ---> git makefile
Target packages  ---> Networking applications ---> openssh
Target packages ---> Compressors and decompressors ---> bzip2 xz-utils zip
Target packages ---> Interpreter languages and scripting ---> python3
Target packages ---> Text editors and viewers ---> nano
Target packages ---> Hardware handling ---> minicom picocom pigpio raspi-gpio
Target packages ---> Debugging, profiling and benchmark ---> gdb ltrace dt
Target packages ---> Libraries ---> Hardware handling ---> dtc
```
Start building the OS image
```
make -j$(nproc)
```

build pass log:
```
INFO: hdimage(sdcard.img): adding partition 'boot' (in MBR) from 'boot.vfat' ...
INFO: hdimage(sdcard.img): adding partition 'rootfs' (in MBR) from 'rootfs.ext4' ...
INFO: hdimage(sdcard.img): adding partition '[MBR]' ...
INFO: hdimage(sdcard.img): writing MBR
```

(option) check image size is about 153 MB
```
ls -hl output/images/sdcard.img 
# -rw-r--r-- 1 yuan yuan 153M Feb 11 10:41 output/images/sdcard.img
```

## flash image to sd card 

use `Pi Image` to erase sd card, If `Pi Image` show erase failed, replug sd card and erase agian\
`Pi Image` download link https://www.raspberrypi.com/software/ \
After erase, sd card show the partition as below:
```
lsblk
# sdd           8:48   1   7.7G  0 disk 
# └─sdd1        8:49   1   7.7G  0 part /media/yuan/16ED-E866
```
flash sdcard.img to sd card
```
sudo dd if=output/images/sdcard.img of=/dev/sdd bs=512 status=progress
# yuan@ax370m-gaming-3:~/buildroot/buildroot$ sudo dd if=output/images/sdcard.img of=/dev/sdd bs=512 status=progress
# [sudo] password for yuan: 
# 152728064 bytes (153 MB, 146 MiB) copied, 20 s, 7.6 MB/s
# 311297+0 records in
# 311297+0 records out
# 159384064 bytes (159 MB, 152 MiB) copied, 20.586 s, 7.7 MB/s
```

After flash, replug sd card, sdd show the partition as below:
```
lsblk
# sdd           8:48   1   7.7G  0 disk 
# ├─sdd1        8:49   1    32M  0 part /media/yuan/0636-B71A
# └─sdd2        8:50   1   120M  0 part /media/yuan/rootfs
```

### install terminal tool and login raspberry pi via uart

install minicom:
```
sudo apt install minicom
minicom -D /dev/ttyUSB0
```

install picocom:
```
sudo apt install picocom
picocom /dev/ttyUSB0 --baud 115200 --omap crlf --echo
```
exit minicom or picocom [ctrl] + [A] ->  [ctrl] + [x] 



### enable root loing ssh

```
vi /etc/ssh/sshd_config
# modify "#PermitRootLogin prohibit-password"                                          
# to
# "PermitRootLogin yes"
```
rerun sshd
```
/etc/init.d/S50sshd restart
```

## override Buildroot Linux Kernel by local kernel repository

clone kernel repository
```
cd ~/buildroot
git clone https://github.com/raspberrypi/linux.git --depth=1 --branch=rpi-6.2.y

```
create a local.mk file to point linux kernel source folder, config toolchain
```
cd ~/buildroot/buildroot
echo 'LINUX_OVERRIDE_SRCDIR = /home/yuan/buildroot/linux/' >> local.mk
make clean
make menuconfig # Toolchain ---> 6.1.x or later
```
Start building the OS image
```
# make linux-rebuild
make -j$(nproc)
```

(option) switch back to tarball 
```
rm ./local.mk
make clean
make menuconfig # Toolchain ---> 5.10
make -j$(nproc)
```

