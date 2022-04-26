---
title: 用 Hyper-V/KVM 搭建与测试虚拟机
date: 2021-05-19 21:29:28
tags: [OS, VM]
categories: OS
---

最近把 Ubuntu 从双系统中删掉了，不过因为经常需要折腾 Linux 相关的东西，所以最近没事在折腾 Linux 虚拟机相关，所以顺便也记录记录。

# 前言

虚拟机一般都是单系统，相对物理机双系统来说最大的优势是可以乱搞，不满意直接挂 iso 重新格式化就好。当然现在 UEFI 引导的系统理论上也可以随便格分区，只要还有有效的启动项就可以了，但如果还是 BIOS 引导的电脑乱格分区，很容易就会需要搜索如何修复引导了（主要还是虚拟机不会影响物理机上的重要文件）。

不过，由于虚拟机需要跑在宿主系统内，性能肯定是比不上直接装到物理机上的，所以虚拟机不适合跑性能要求高的东西，比如炼丹。

我测试的系统是：物理机 [Windows 10 x64](https://www.microsoft.com/software-download/windows10)，图形界面虚拟机 [Ubuntu 20.04](https://ubuntu.com/#download)，命令行虚拟机 [Debian 10](https://www.debian.org/)。虚拟机软件为：Windows 的 Hyper-V 和 Linux 的 KVM。之前用过 VMware 和 VirtualBox，但由于第三方的虚拟机软件或多或少会和系统的一些配置有冲突，因此我现在一般都采用这两个操作系统原生的虚拟机技术。

# Hyper-V

Hyper-V 是微软推出的虚拟化技术，通过图形界面一步一步操作基本问题不大，这里主要记录一些重要的点。

## 启用 Hyper-V

Hyper-V 需要在 windows 功能里面打开，在 win 10 里直接搜索 hyper-v，最匹配就会出现 `windows 功能`，如果是 `hyper-v 管理器` 则说明 hyper-v 已经启用了（如果搜不到的话，走 `控制面板` -> `程序` -> `启用 windows 功能`）。勾选 `Hyper-V` 后，重启电脑便可以生效。

> 根据[文档](https://docs.microsoft.com/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v)，win 10 家庭版没有 hyper-v，得自行升级或[安装](https://www.itechtics.com/enable-hyper-v-windows-10-home/)

<img src="/img/OS/VM/hyperv_search.png" />

<img src="/img/OS/VM/hyperv_enable.png" />

## 配置虚拟网络（交换机）

1. 打开 `hyper-v 管理器`（和上面一样搜索或者在 `开始` -> `Windows 管理工具`），点击右边的 `虚拟交换机管理器`，默认应该已经有一个 `default switch`，如果没有需要自己建一个外部网络的虚拟网卡。（或者选内部网络并配置 NAT 也可以）

    > 外部网络属于桥接模式，依赖路由器分配 IP，可以与局域网内其他设备直接通信；内部网络会建立一个新的局域网，只有宿主机可以访问到虚拟机

    <img src="/img/OS/VM/switch_manager.png" />

2. 打开 `网络设置` 里的 `适配器选项`，双击 `default switch` 或者刚刚建立的网卡，选择 `属性` -> `IPv4` -> `高级` -> `DNS`，添加一个 DNS 后缀（如 mshome.net 等公网不解析的域名或自己的域名）

    > 修改适配器 DNS 主要是为了用计算机名字访问对应虚拟机，如果不需要的话可以跳过

    <img src="/img/OS/VM/default_switch.png" />

## 创建虚拟机

1. 回到 `hyper-v 管理器`，选择 `新建` -> `虚拟机`，按引导一步步往下走配置内存、硬盘和网络即可。其中，第一代虚拟机为 BIOS 引导，第二代为 UEFI 引导，按自己喜欢的选就行。创建好后，右键虚拟机按需进行一些额外的设置。
    * 如果选择 UEFI，并且不装 windows，需要将安全启动(Secure Boot)关闭或按自己需求修改（如 ubuntu 或 debian 可以采用微软 UEFI 证书）
    * 如果没有建 DVD 光驱或没有挂载 iso，可以在 SCSI 里添加 DVD 与 iso (iso 在开头有下载链接，或者自己找官网下载)，并在 `固件(Firmware)` 中设置为 DVD 启动

2. 用管理员模式打开 powershell，输入以下命令打开嵌套虚拟化（为了在虚拟机内跑虚拟机，如果不需要可以跳过），其中 `<vmName>` 是上面创建的虚拟机名字:

    ```powershell
    Set-VMProcessor -VMName <vmName> -ExposeVirtualizationExtensions $true
    Get-VMProcessor -VMName <vmName> | fl
    ```

    如果输出里面 `ExposeVirtualizationExtensions` 是 `true` 就说明开启成功了。

    <img src="/img/OS/VM/powershell_virtualization.png" />

3. 启动虚拟机，按正常装系统的步骤走就可以，虚拟硬盘直接整个格式化就好（除非要在虚拟机装双系统）。
    * debian 安装中有一个填 domain name 的步骤，填上我们之前设置的 DNS 域名，这样我们就可以安装完成后通过宿主机用机器名 ssh 进去
    * ubuntu 界面大小修改时在 `/etc/default/grub` 里面的 `GRUB_CMDLINE_LINUX_DEFAULT` 加上 `video=hyperv_fb:1920x1080`，然后 `update-grub` 并重启就好
    * 通过 `echo "$USER ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER` 可以让所有的 `sudo` 命令都不用再输入密码

4. 顺便提一句，hyper-v 的虚拟机似乎不支持宿主机与虚拟机之间通过复制粘贴传输文件，因此我是采用自己写的一个 http 文件服务来传输的（也可以自行采用其他的网络传输协议）
    * 下载 [fileserver](https://github.com/ChenTanyi/fileserver/releases) 对应平台的可执行文件，放到对应文件夹执行，便可以在另一个的浏览器内通过 IP 访问到文件夹内的文件

# KVM

> 由于 KVM 是 Linux 内核的虚拟化方案，因此就算是 GUI 的操作，也会涉及很多 terminal 命令，默认读者已经了解，不会作过多解释（不了解的话，推荐先自己玩玩 Linux 了解下）。我用的两个 Linux 系统都是 debian 系的，所以安装命令都只会按 debian 系给出。

1. 执行下面的命令查看系统是否支持 kvm 虚拟化（如果有输出就是支持）:
    ```bash
    lsmod | grep kvm
    ```

2. 安装 common package:
    ```bash
    sudo apt install -y qemu-kvm libvirt-daemon-system bridge-utils
    sudo usermod -aG libvirt $USER
    sudo usermod -aG kvm $USER
    ```
    装完可以通过 `sudo systemctl status libvirtd` 查看服务状态；如果没启动，则通过 `sudo systemctl enable libvirtd --now` 启动，通过 log 看看是否有什么问题（有问题可以先重启试试）

## GUI

`virt-manager` 提供以图形界面(GUI)的方式管理 KVM，第一次用的时候推荐先用 GUI 测试，相对来说操作比较简易。

安装并启动 `virt-manager` 后，和 `hyper-v` 一样一步步跟着配置系统、CPU、内存、硬盘就 OK 了。

连接虚拟机时， `virt-manager` 也一样会打开一个图形界面操作虚拟机，因此没有什么需要详细说明的。

```bash
sudo apt install -y virt-manager
virt-manager
```

ubuntu 虚拟机嵌 ubuntu 虚拟机的效果如下：

<img src="/img/OS/VM/ubuntu_in_ubuntu.png" />

## CLI

### 安装与操作

`virt-install` 命令可以通过命令行(CLI)的方式安装 KVM 虚拟机，所有通过图形界面配置的参数都需要在命令参数里面指定。

对于虚拟系统的安装，其提供远程桌面协议（如 vnc, spice）打开图形界面进行安装，这种方式其实和 `virt-manager` 安装过程区别不大；当然我们也可以直接在命令行里面进行系统的安装，这个需要系统镜像支持命令行模式安装（一般是 server 版系统会支持），我们只需要使用 `--graphics none --extra-args console=ttyS0` 将安装界面重定向到当前 terminal 即可，退出时按下 `Ctrl + ]`。

下面只给出一个安装 debian 最简单的例子，其他高级技巧请参考 [docs](https://linux.die.net/man/1/virt-install)

```bash
sudo apt install -y virtinst

# 启动全局的网络配置
export LIBVIRT_DEFAULT_URI="qemu:///system"
virsh net-start default && virsh net-autostart default

# 安装 vm 并从 iso 启动
virt-install --name vm --memory 1024 --vcpus 1 --disk path=$HOME/vm.disk,size=15 --location $HOME/debian-10.9.0-amd64-netinst.iso --network bridge:virbr0 --graphics none --extra-args console=ttyS0

# 如果要开窗口安装如 ubuntu desktop 系统可以用下面命令
# virt-install --name vm --memory 1024 --vcpus 1 --disk path=$HOME/vm.disk,size=15 --cdrom $HOME/ubuntu-20.04.2.0-desktop-amd64.iso --network bridge:virbr0 --graphics vnc,listen=0.0.0.0
# sudo apt install -y xtightvncviewer && vncviewer localhost
```

其余的 VM 操作基本都通过 `virsh` 完成，这边举一些简单的例子，高级技巧请参考 [docs](https://linux.die.net/man/1/virsh)

> 当然用 CLI 创建的 VM 一般会使用 ssh 进行连接与操作，只有在非正常操作 VM 的时候（如强杀或删除）才会使用 virsh

```bash
virsh console vm # 进入 vm 的命令行，一样是 Ctrl + ] 退出
virsh start vm # 启动 vm
virsh shutdown vm # 关闭 vm
virsh destroy vm # 强杀 vm （相当于直接断电）
virsh undefine vm # 删除 vm
```

### 网络与 DNS

KVM 的网络配置主要与宿主机的访问相关，我们可以直接使用虚拟机 IP 进行访问，也可以通过 libvirt 的 DNS 服务直接使用虚拟机机器名访问。

1. 如果要获取 VM 的 IP，可以通过 `virsh domiflist vm` 获取 vm 的所有网卡以及对应的 MAC，然后通过 `arp -n | grep $MAC` 可以获得对应 MAC 的 IP 地址。当然，如果虚拟机的网络不是桥接的，可以直接通过 `virsh domifaddr vm --source arp` 得到所有网卡对应的 MAC 和 IP。
    * 如果需要 NAT 模式的静态 IP 的话，可以通过 `virsh net-edit default` 把 `<dhcp>` 改成下面这种样子，然后 `virsh net-destroy default && virsh net-start default`。
    ```xml
    <dhcp>
        <range start='192.168.122.100' end='192.168.122.254' />
        <host mac='52:54:00:50:65:74' name='vm' ip='192.168.122.11' />
    </dhcp>
    ```
    * 如果需要阻止 vm 联网，可以在 `virsh edit vm` 里面把网卡 `<interface>` 注释掉（最好通过 `virsh dumpxml vm > vm.xml` 备份与 `virsh define vm.xml` 还原）。
2. libvirt 的 DNS 服务默认地址为 192.168.122.1（可以通过 `virsh net-edit default` 进行修改），通过下面命令注册 DNS 服务器，就可以直接使用虚拟机的机器名访问到对应的虚拟机了。
    ```bash
    sudo apt install -y resolvconf
    echo "nameserver 192.168.122.1" | sudo tee -a /etc/resolvconf/resolv.conf.d/head
    sudo resolvconf -u
    ```

<img src="/img/OS/VM/kvm_ip.png" />

# 参考

* https://fm9t.github.io/2019/01/03/HyperV-DefaultSwitch/
* https://askubuntu.com/questions/702628/ubuntu-hyper-v-guest-display-resolution-win-10-15-04
* https://askubuntu.com/questions/147241/execute-sudo-without-password
* https://www.linuxtechi.com/install-kvm-on-ubuntu-20-04-lts-server/
* https://linuxize.com/post/how-to-install-kvm-on-ubuntu-20-04/
* https://www.cyberciti.biz/faq/howto-linux-delete-a-running-vm-guest-on-kvm/
* https://unix.stackexchange.com/questions/33191/how-to-find-the-ip-address-of-a-kvm-virtual-machine-that-i-can-ssh-into-it
* https://serverfault.com/questions/627238/kvm-libvirt-how-to-configure-static-guest-ip-addresses-on-the-virtualisation-ho
* https://help.ubuntu.com/community/KVM/Networking
