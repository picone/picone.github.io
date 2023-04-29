---
layout: post
title: 基于 KVM 安装 Win11
description: 本文介绍如果在 Ubuntu 20.04 上安装 KVM，并在 KVM 上安装 Win11。整个过程使用 VNC 不需要显示器。
---

# 具体过程

## 1. 安装 KVM

安装 KVM 很简单，网上也有很多教程。直接 apt-get 安装依赖即可：

```shell
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
# 查看 libvirtd 状态，跑起来了即成功
sudo systemctl status libvirtd
```

## 2. 创建虚拟机

先下载 [Win 11 的 ISO](https://next.itellyou.cn/Original/Index#cbp=Product?ID=42e87ac8-9cd6-eb11-bdf8-e0d4e850c9c6) 和
[virtio 驱动 ISO](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md)。

使用 virt-install 可以创建虚拟机镜像。

```shell
virt-install --connect qemu:///system \ 
--name=win11 \
--ram 4096 \
--vcpus=8 \
--accelerate \
--cdrom=zh-cn_windows_11_consumer_editions_version_22h2_updated_april_2023_x64_dvd_73bd9a5e.iso \ # 下载的 ISO 路径
--disk path=win11.raw,size=50,format=raw,bus=virtio \ # 虚拟磁盘路径
--disk /home/work/virtio-win.iso,device=cdrom,bus=sata \ # virtio 驱动路径，不能使用 --cdrom，会覆盖前面的，坑
--graphics vnc,listen=0.0.0.0,password=${你的密码} \ # 视频直接导到 VNC 上
--video virtio \
--noautoconsole \
--network network=default,model=virtio \
--features kvm_hidden=on,smm=on \
--tpm backend.type=emulator,backend.version=2.0,model=tpm-tis # 开启 TPM，不过貌似没作用
```

virsh 命令使用：

```shell
list --all # 查看所有虚拟机
start win11 # 启动虚拟机
shutdown win11 # 关闭虚拟机
destroy win11 # 强制关闭虚拟机
undefine win11 # 删除虚拟机
define win11.xml # 从 xml 文件定义虚拟机
```

重启时虚拟机会关闭，目前不知道怎么解决，可以使用 `virsh start win11` 启动回来就好了。

## 3. 安装 Windows 11

- 因为磁盘我使用了 bus=virtio，所以需要在安装界面手动加载驱动，浏览 virtio.iso 里面的 /amd/win11 文件夹，并选择后点击下一步即可找到磁盘。

- 安装过程会检查 TPM 和磁盘、内存等配置，需要手动禁用一下：

```shell
# 在安装界面按 Shift + F10 打开命令行，MAC 的话是 Fn + Shift + F10
reg add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassTPMCheck" /t REG_DWORD /d "1" /f
reg add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassSecureBootCheck" /t REG_DWORD /d "1" /f
reg add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassRAMCheck" /t REG_DWORD /d "1" /f
reg add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassStorageCheck" /t REG_DWORD /d "1" /f
reg add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassCPUCheck" /t REG_DWORD /d "1" /f
reg add "HKLM\SYSTEM\Setup\MoSetup" /v "AllowUpgradesWithUnsupportedTPMOrCPU" /t REG_DWORD /d "1" /f
```

- 初始化过程会检查网络，创建 Microsoft 账号，可以用这个方法跳过：

```shell
# 在安装界面按 Shift + F10 打开命令行，MAC 的话是 Fn + Shift + F10
OOBE\BYPASSNRO
```

- 安装完后没有网络，因为网卡使用了 `model=virtio`，需要手动安装驱动。在设备管理器里浏览 virtio.iso 的驱动即可。

- 只有一个核心，是因为使用命令创建时没有指定 CPU 拓扑，需要修改一下，否则所有不能所有核都打到：

```shell
virsh shutdown win11
virsh edit win11
# 在 <cpu> 标签下添加 <topology sockets='1' cores='4' threads='2'/>，意思为 4 核超线程。
virsh start win11
```

![Modify Cores]({{ "/assets/images/2023-04-29-after-modify-cores.png" | relative_url }})
