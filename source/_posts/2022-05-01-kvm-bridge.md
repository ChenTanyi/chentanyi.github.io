---
title: 搭建 KVM Bridge 网络
date: 2022-05-04 09:33:32
tags: [Network, VM]
categories: Network
---

# 前言

由于本地环境都是 windows，所以基本上所有虚拟机相关的使用和测试都是在 hyper-v 里面的，就连之前[文章](../../../../2021/05/19/2021-05-19-hyperv-kvm-create-vm/)里面的 KVM 相关测试也是在 hyper-v 里面的 Linux VM 做的。很多时候虚拟机是需要向整个局域网提供服务的，所以一般情况使用的网络拓扑都是桥接（Bridge）模式。在 hyper-v 里面，搭建桥接网络很简单，只要虚拟交换机里面创建一个外部网络就可以；对于 KVM 而言，Linux 没有内置的指令一键完成，因此需要自行配置网络。

# 网络结构介绍

> 本文不会详细解释各个网络名词，如网关（Gateway），子网（Subnet）等，还不了解的话需要自行搜索

一般设备间的网络拓扑有两种（针对 IPv4 的局域网情况，IPv6 暂时不需要 NAT）：
1. 交换机（Switch）模式，多台设备通过交换机（或直连）形成 Bridge 网络，所有设备在同一个子网下面，可以互相访问，由外部的网关统一管理。
2. 路由器（Router）模式，一个设备作为网关（一般是路由器）构建的 NAT 网络，网关作为内外网交互的渠道，内网可以访问外网设备，反之不行。

## 虚拟机网络结构

从网络拓扑上看，虚拟机和宿主机是两台独立的设备，其中虚拟机使用的一般是虚拟网卡，通过一根虚拟网线连接到宿主机上，因此虚拟机只能通过宿主机的物理网卡与外网通信。这两者间的网络结构实际上完全取决于宿主机的网络模式：
1. NAT 模式：宿主机网络不变，创建一个虚拟网卡作为网关（以及 DHCP Server），所有的虚拟机都是隶属于该网关的子网设备。
2. Bridge 模式：宿主机创建一个虚拟交换机连接物理网卡，物理网卡以桥接模式连接，不分配 IP，再创建一个虚拟网卡连接到交换机供宿主机系统使用，所有虚拟机的网卡同样连接到交换机上，形成 Bridge 网络。

### 网络结构图示

下面用图具体展示两种网络对应的结构：
* 路由器子网为 10.0.0.0/24，KVM NAT 子网为 192.168.122.0/24
* 每个设备网卡为 eth0
* 宿主机的虚拟网关为 virbr0
* 宿主机的虚拟交换机和虚拟网卡为 br0（在 Linux 内，一个 Bridge 设备既是虚拟交换机也是虚拟网卡）

<img src="/img/OS/VM/vm_network.svg" />

# 搭建 Bridge 网络

> 由于网络配置需要的权限比较大，下方基本所有命令都要在 root 用户下完成，不再添加 sudo 了

## 修改宿主机网络

> **_警告_** ：由于操作会影响宿主机对应网卡的连通性，最好不要在远程主机的主网卡上进行测试。

### 手动配置

手动配置主要用于实践中一步一步操作的介绍，这些配置一般都需要配置成开机自启动

1. 配置 IP 转发。无论是什么网络，Linux 用作网络设备的第一步就是配置 IP Forwarding

    ```bash
    sysctl -w net.ipv4.ip_forward=1
    ```

2. 创建 Bridge 设备并将它连接到物理网卡

    ```bash
    # ip link set eth0 up # 如果网卡还未启用，需要先将其状态设置为启用
    ip link add br0 type bridge
    ip link set br0 up
    ip link set eth0 master br0
    ```

3. 清除网卡上的 IP 配置，并将它转移到 Bridge 设备上（运行命令前确保没有 DHCP 相关的 client 在跑，否则无法删除网卡的 IP）

    ```bash
    ip addr del 10.0.0.2/24 dev eth0
    ip addr add 10.0.0.2/24 broadcast + dev br0
    ip route add default via 10.0.0.1 dev br0
    ```

经过第二步配置后，网卡会失去上网功能，而第三步配完后，操作系统会通过虚拟网卡 br0 恢复网络连通性。因此，如果明确自己的输入无误的话，可以把它们合成一个脚本去执行，就可以保证网络基本不中断（和 hyper-v 创建一个新外部网络一样）。

### 开机自启动

手动配置会在系统重启后失效，因此在测试 OK 之后，我们需要把它配置成开机自启动。

1. 依然还是 IP 转发，可以直接添加到 `/etc/sysctl.conf` 也可以先看看这个文件里面的配置是否已经启用了

    ```bash
    echo 'net.ipv4.ip_forward=1` >> /etc/sysctl.conf
    ```

2. 配置网卡与 IP，通过修改 `/etc/network/interfaces` 这个文件达到效果，文件修改成下面的状态，具体 Bridge 相关参数可以自行抉择

    ```bash
    auto lo
    iface lo inet loopback

    auto eth0
    # iface eth0 inet dhcp
    iface eth0 inet manual  # 将 eth0 的 IP 改成手动配置，但不配置它

    auto br0
    iface br0 inet dhcp     # 让 br0 通过 DHCP 分配 IP，也可以自己手动配置想要的 IP
        bridge_ports eth0   # Bridge 到网卡 eth0
        bridge_stp off      # 禁用 Spanning Tree Protocol（判断 Bridge Routing 成环）
        bridge_fd 0         # 无转发延迟
        bridge_maxwait 0    # 不等待 Bridge 转发生效
    ```

3. 重启，然后通过 `ip a` 可以看配置是否生效（如果配置的是远程主机并且配置有问题，那么你就再也访问不了了，只能重置主机）

### 禁用 netfilter

在 Bridge 设备禁用 netfilter 不是搭建网络的必要步骤，需要自行斟酌。它可以使桥接网络不受宿主机各种转发规则的影响，并提高性能表现；但它也同时会和其他一些通过 bridge + netfilter 实现的系统有冲突，如 Kubernetes。

1. 添加 sysctl 的配置

    ```bash
    cat > /etc/sysctl.d/bridge.conf << EOF
    net.bridge.bridge-nf-call-ip6tables=0
    net.bridge.bridge-nf-call-iptables=0
    net.bridge.bridge-nf-call-arptables=0
    EOF
    ```

2. 开机加载 `br_netfilter` 模块，并加载上面的配置

    ```bash
    cat > /etc/udev/rules.d/99-bridge.rules << EOF
    ACTION=="add", SUBSYSTEM=="module", KERNEL=="br_netfilter", RUN+="/sbin/sysctl -p /etc/sysctl.d/bridge.conf"
    EOF
    ```

## KVM 网络配置

1. 添加一个 KVM 网络连接到 Bridge 设备

    ```bash
    cat > bridge-netowrk.xml << EOF
    <network>
        <name>bridge-network</name>
        <forward mode="bridge" />
        <bridge name="br0" />
    </network>
    EOF
    virsh net-define bridge-netowrk.xml
    virsh net-start bridge-netowrk
    virsh net-autostart bridge-netowrk
    ```

2. 使用这个网络创建新的虚拟机，或者把已有虚拟机的网络修改成这个网络
    * 创建：在 `virt-install` 添加参数 `--network network=bridge-network`
    * 修改：用 `virsh edit vm`，修改 `<interface>` 里面的 `type`, `source` 和 `target` 字段，推荐随便创建一个新的虚机并对比着修改

通过这些配置，启动后的虚拟机就和宿主机处于同一个子网下面，可以被子网下其他设备访问到。

# 参考

* https://linuxconfig.org/how-to-use-bridged-networking-with-libvirt-and-kvm
* https://askubuntu.com/questions/179508/kvm-bridged-network-not-working
* https://wiki.archlinux.org/title/Network_bridge
* https://wiki.libvirt.org/page/Networking
* https://wiki.debian.org/BridgeNetworkConnections
* https://manpages.debian.org/jessie/bridge-utils/bridge-utils-interfaces.5.en.html
* https://levelup.gitconnected.com/how-to-setup-bridge-networking-with-kvm-on-ubuntu-20-04-9c560b3e3991
* https://serverfault.com/questions/963759/docker-breaks-libvirt-bridge-network
* https://segmentfault.com/a/1190000009491002
* https://imroc.cc/post/202105/why-enable-bridge-nf-call-iptables/
