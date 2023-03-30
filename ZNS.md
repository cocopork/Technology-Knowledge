# [ZNS](https://zonedstorage.io/docs/getting-started/zns-device)

[安装QEMU](#安装QEMU)

​	[手动卸载QEMU](#手动卸载QEMU)

​	[一直显现VNC接口的原因](#一直显现VNC接口)

[安装FEMU](#安装FEMU)

​	[配置openssh-server](#配置openssh-server)

[在虚拟机中安装Ubuntu](#在虚拟机中安装Ubuntu)

​	[配置共享文件夹](#配置共享文件夹)

​	[运行QEMU](#运行QEMU)

[构建ZNS虚拟设备(TODO)](#构建ZNS虚拟设备)

[支持ZNS的内核编译选项](#支持ZNS的内核编译选项)

[其他问题](#其他问题)

## 安装QEMU

[Download QEMU - QEMU](https://www.qemu.org/download/#source)

```shell
#一站式脚本
git clone https://gitlab.com/qemu-project/qemu.git
cd qemu
git submodule init
git submodule update --recursive
./configure --enable-kvm --disable-werror --prefix=/usr/local --target-list="x86_64-softmmu" --enable-libpmem
make
```

 如果没安装Ninja

```shell
sudo apt-get install ninja-build
```

### 手动卸载QEMU

```shell
rm -rf /usr/local/bin/qemu-*
rm -rf /usr/local/libexec/qemu-*
rm -rf /usr/local/etc/qemu
rm -rf /usr/local/share/qemu
```

### 一直显现VNC接口

```shell
#启动QEMU后只显示以下内容
QEMU 5.2.0 monitor - type 'help' for more information
(qemu) VNC server running on 127.0.0.1:5900
```

原因：

​	消息显示，QEMU正在使用VNC协议进行图形输出。您可以将VNC客户端连接到127.0.0.1:5900端口，它会告诉您即将看到图形输出。

​	如果您想要的是本机X11窗口（GTK），那么问题可能是您没有安装必要的库来构建GTK支持。QEMU的configure脚本的默认行为是“构建此主机安装了库的所有可选功能，并忽略没有库的功能”。因此，如果您在构建QEMU时没有任何GTK/SDL等库，那么在生成的QEMU二进制文件中唯一能得到的就是最低公分母VNC支持。如果您希望configure报告缺失功能的错误，则需要向其传递适当的--enable whatever选项以强制启用该功能（在本例中，--enable-gtk）。

```shell
#可以通过以下指令安装QEMU必要依赖
apt build-dep qemu
```

## 安装FEMU

​	FEMU是基于QEMU的、用于模拟NVMe SSD设备的虚拟机。安装FEMU的原因是，FEMU本身有自己模拟ZNS设备的方法，安装更加高效一些。

​	[vtess/FEMU: FEMU: Accurate, Scalable and Extensible NVMe SSD Emulator (FAST'18) (github.com)](https://github.com/vtess/FEMU)

<font color="red">安装之前先确保QEMU的依赖，可以用以下指令</font>

```shell
# 可以通过以下指令安装QEMU必要依赖
apt-get build-dep qemu
# 实在不行，自己找依赖包逐个安装
```

安装脚本

```shell
  git clone https://github.com/vtess/femu.git
  cd femu
  mkdir build-femu
  # Switch to the FEMU building directory
  cd build-femu
  # Copy femu script
  cp ../femu-scripts/femu-copy-scripts.sh .
  ./femu-copy-scripts.sh .
  # only Debian/Ubuntu based distributions supported
  sudo ./pkgdep.sh
```

> FEMU编译好后的二进制文件为 x86_64-softmmu/qemu-system-x86_64

### 操作femu

​	FEMU的镜像u20已经配置好了，直接在宿主机连接即可

​	如果在运行femu的终端直接操作，ctrl+c 终止快捷键会关闭整个虚拟机。

```shell
#SSH连接
ssh -p8080 femu@localhost
```



## 在虚拟机中安装Ubuntu

<font color="red">注意：以下qemu脚本中的注释只是方便阅读，实际运行时脚本中间不能插入注释</font>

```shell
# QEMU第一次启动内核.iso，需要安装操作系统
# 下载操作系统镜像
# wget ...
# 提取内核和根文件，先将.iso挂载到临时文件夹，将其中内容复制下来
mount ubuntu-20.04.5-live-server-amd64.iso /mnt/temp -o loop
cp -r /mnt/temp/ ./
# 开始安装
sudo qemu-system-x86_64 --enable-kvm \
-m 8G -smp 8 \
#直接指定镜像位置
../images/ubuntu.qcow2 \
-cdrom ubuntu-22.04.1-live-server-amd64.iso \
--nographic \
-append console=ttyS0 \
# 内核文件位置
-kernel temp/casper/vmlinuz \
# 根文件位置
-initrd temp/casper/initrd \
```

下面是使用scsi控制器，将镜像存储在模拟磁盘上

```shell
sudo qemu-system-x86_64 --enable-kvm \
-m 8G -smp 8 \
# 将磁盘镜像配置到硬盘上
# 配置一个 virtio 实现的 SCSI 控制器
-device virtio-scsi-pci,id=scsi0 \
# 在 SCSI 控制器上配置一个硬盘
-device scsi-hd,drive=hd0 \
-drive file=../images/ubuntu.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
# 指定.iso文件位置
-cdrom ubuntu-22.04.1-live-server-amd64.iso \
-kernel temp/casper/vmlinuz \
-initrd temp/casper/initrd \
# append指令只能跟随在 -kernel 之后
# 若需要在终端上显示命令行，后续启动不需要指定输出端口，使用 -nographic 即可
-append "console=ttyS0" 
```



### 配置共享文件夹

方法一：模拟共享磁盘，<font color="red">不能实时共享文件，不推荐！</font>

1. 宿主机

```shell
#使用dd创建一个4G大小的文件，作为虚拟机和宿主机之间传输桥梁
dd if=/dev/zero of=./share.img bs=4M count=1k
#格式化share.img文件
mkfs.ext4 ./share.img
#创建一个共享文件夹并挂载
mkdir /mnt/share
mount -o loop ./share.img /mnt/share
#之后把需要传输给虚拟机的文件放到/mnt/share即可
```

2. 虚拟机

```shell
#启动qemu时添加选项
-hdb ./share.img
#创建/root/share文件夹，作为虚拟机的挂载点
mkdir /root/share
#以同样的文件格式ext4挂载刚刚添加的/dev/sdb硬盘
mount -t ext4 /dev/sdb /root/share
#通过访问/root/share文件夹即可以获得宿主机上放在/tmp/share文件夹下的文件
```

方法二：使用virtfs网络文件系统

1. 在宿主机

​	在femu-compile.sh中加入配置  --enable-virtfs ，重新编译，实现共享目录。

​	如果使用的是QEMU，则在运行./configure时添加 --enable-virtfs 。

```shell
#!/bin/bash

NRCPUS="$(cat /proc/cpuinfo | grep "vendor_id" | wc -l)"

make clean
# --disable-werror --extra-cflags=-w --disable-git-update
../configure --enable-kvm  --enable-virtfs --target-list=x86_64-softmmu 
make -j $NRCPUS

echo ""
echo "===> FEMU compilation done ..."
echo ""
exit
```

在run-zns.sh中加入配置，或者在运行qemu_system_x86的脚本里加入配置。

```
-fsdev local,security_model=passthrough,id=fsdev0,path=/tmp/share \
-device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=hostshare \
```

2. 在虚拟机

在虚拟机中挂载共享目录

```shell
mount -t 9p -o trans=virtio,version=9p2000.L hostshare /home/femu/share
```



### 运行QEMU

```shell
sudo qemu-system-x86_64 --enable-kvm \
$IMGDIR/ubuntu.qcow2 \
-nographic \
-name "ggboy-Virtual" \
# 设置内存限制
-m 8G,slots=4,maxmem=600G \
-smp 16 \
#设置QEMU输出信息到控制台
-append "console=ttyS0 nokaslr" \
-machine pc,accel=kvm,nvdimm=on \
#降低性能损耗
-cpu host \
#设置系统镜像
# -device virtio-scsi-pci,id=scsi0 \
# -device scsi-hd,drive=hd0 \
# -drive file=$OSIMGF,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
#配置NVM设备
# -object memory-backend-file,id=mem1,share=on,mem-path=/dev/dax3.0,size=124G,align=2M \
# -device nvdimm,id=nvdimm1,memdev=mem1 \
#启动共享文件夹，使用virtfs网络文件系统实现共享
# -fsdev local,security_model=passthrough,id=fsdev0,path=/tmp/share \
# -device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=hostshare \
#配置网络接口
-net user,hostfwd=tcp::8097-:22 \
-net nic,model=virtio \
-serial stdio \
-qmp unix:./qmp-sock,server,nowait 
```

## 构建ZNS虚拟设备

### 创建设备虚拟文件

​	femu自带一块zns设备。

​	使用truncate命令创建指定大小的空文件，用于存储虚拟ZNS的数据。

```shell
# 在目标目录下创建一个虚拟文件
truncate -s 16G ./images/zns.raw
# 查看文件是否被成功创建
ls -l ./images/zns.raw
```



## 支持ZNS的内核编译选项

​	详见官方文档[Kernel Configuration | Zoned Storage](https://zonedstorage.io/docs/linux/config#device-drivers-configuration)。相关配置都是默认开启，不用专门设置

```
Enable the block layer
	Zone block device support
```

​	在宿主机编译内核`make -j24`，在虚拟机安装内核`make modules_install install`。



## 其他问题

### 关闭QEMU

```shell
# 只关闭系统，但不会主动关闭电源
# 关闭系统之后会停止在 boot:System Halted
shutdown -h now
# 要想完全关闭，需要下面的指令
poweroff
```

### 查看占用空间

```shell
#查看磁盘占用
df -h
#查看当前目录占用
du -h --max-depth=0
```

