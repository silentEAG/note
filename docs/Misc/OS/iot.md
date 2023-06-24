---
tags:
  - TODO
---

# IOT 环境搭建

Cisco RV34x RCE

[固件下载](https://software.cisco.com/download/home/286287791/type/282465789/release/1.0.03.24?catid=268437899)

拿 rootfs: 

```sh
binwalk -Me ...img
# https://github.com/nlitsme/ubidump
python ubidump.py -s . 0.ubi
```

源码编译 qemu

```sh
TODO
```

系统级 qemu 启动

```sh
wget https://people.debian.org/~aurel32/qemu/armhf/debian_wheezy_armhf_standard.qcow2
wget https://people.debian.org/~aurel32/qemu/armhf/vmlinuz-3.2.0-4-vexpress
wget https://people.debian.org/~aurel32/qemu/armhf/initrd.img-3.2.0-4-vexpress
sudo apt-get install bridge-utils uml-utilities
sudo brctl addbr Virbr0
sudo ifconfig Virbr0 192.168.153.1/24 up
sudo tunctl -t tap0
sudo ifconfig tap0 192.168.153.11/24 up
sudo brctl addif Virbr0 tap0
#qemu启动
sudo qemu-system-arm -M vexpress-a9 -kernel vmlinuz-3.2.0-4-vexpress -initrd initrd.img-3.2.0-4-vexpress -drive if=sd,file=debian_wheezy_armhf_standard.qcow2 -append "root=/dev/mmcblk0p2" -net nic -net tap,ifname=tap0,script=no,downscript=no -nographic -s
#qemu网卡配置
ifconfig eth0 192.168.153.2/24 up
#上传根目录文件
scp -r -O rootfs/ root@192.168.153.2:~/
chmod -R 777 rootfs/
sudo mount --bind /proc proc
sudo mount --bind /dev dev

# 配置 chroot
cp /bin/bash ./bin
ldd /bin/bash

# 然后复制依赖库到 rootfs/lib/ 目录下

# 启动
chroot . /bin/sh
```


## 分析

`etc/init.d` 文件夹下分析相关启动脚本