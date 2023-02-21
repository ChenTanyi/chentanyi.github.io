---
title: Hyper-V 配置内网固定 IP
date: 2023-02-21 18:11:30
tags: [VM, Network]
categories: Network
---

使用 Hyper-V 或者 KVM 的虚拟机的时候，我一般会通过 SSH 或者 RDP 进行访问，这样不同虚拟机的使用体验相对统一，不依赖具体 VM 管理软件的功能。只要配置好网络，就和正常使用服务器一样（不考虑有些公司内的虚机限制联网的情况）。因此，我们需要先来配置合适的网络满足我们的需求。

# 网络选择

在之前 [KVM 网络](/2022/05/04/2022-05-01-kvm-bridge/)里面介绍了两种网络的拓扑，Bridge 和 NAT，两者最大的区别在于是否多了一层子网，因为子网内的机器正常情况下无法被外面的其他机器访问到，因此选择的时候最主要考虑的就是这个点。

1. **Default NAT**: Hyper-V 和 KVM 都会在安装完成后自动创建一个带 NAT 的子网。KVM 创建的是固定的 IP 段，还可以再自行修改到想要的 IP 段；而 Hyper-V 创建的 IP 段会在每次重启时发生变化，如果要使用固定的 IP 段，需要每次都重新配置一下，相当麻烦。
1. **Bridge**: 在可以搭建 Bridge 网络的情况下，我倾向使用 Bridge 网络，这种情况下，VM 和宿主机有相同的连通性，可以配置成一个类似整个局域网的设备，如旁路由。但是，Bridge 网络有着麻烦的前置条件，就是需要破坏宿主机已有的网络，并且依赖外部的路由器，一般适合用在家里能物理访问到的机器上，不用担心网络出问题了连不上。普通家用路由器的情况下，一般路由器的 DHCP 都会根据 MAC 分配固定的 IP，当然自己也可以手动指定一个不冲突的 IP。
1. **Static NAT**: 这个是专门针对 Hyper-V 的，因为 KVM 默认就是 Static NAT，不需要在单独配置。Hyper-V 可以通过新建一个虚拟交换机来分配固定的 IP 段，使得虚拟机可以配置固定的 IP。

因此，我一般在本地主机选择 Bridge，而使用服务器是会选 Static NAT。其实我之前[配置 VM](/2021/05/19/2021-05-19-hyperv-kvm-create-vm/) 的时候，也会选择配置主机名，经过局域网广播后，可以通过主机名直接访问而不需要关心 IP，但这个只有在局域网直连时有用，做一些特殊配置（如 proxy）的时候就可能会比较麻烦。

# 网络配置

## Bridge

Hyper-V 配置 Bridge 网络很简单，只要在创建虚拟交换机的时候选择`外部网络(External)`，就会自动配置完成，然后在 VM 设置里面指定对应的交换机即可。

## Static NAT

接下来就是主要讲讲配置 Static NAT 的一些步骤

1. 在 Hyper-V 的虚拟交换机里面添加一个`内部网络(Internal)`来配置 NAT，我把它的名字设置为 `Static`

    <img src="/img/OS/VM/switch_manager_static.png" />

1. 打开适配器页面，选择当前联网的适配器，打开属性，选择分享网络给这个 NAT 交换机（这个步骤比较关键，做完才能让这个 NAT 访问外网）
    > 每个适配器只能分享网络给一个适配器，所以如果需要分享给多个适配器，需要选择这些适配器，右键桥接（Bridge），然后分享网络给这个网桥

    <img src="/img/OS/VM/share_network.png" />

1. 选择 `Static` 适配器，打开属性中的 `IPv4`，适配器的 IP 段默认会被改成 `192.168.137.0/24`，可以自己改成其他的 IP 段（比如我改成 `192.168.100.1`）。要记住这个适配器的 IP，是这个 NAT 网络的网关。

    <img src="/img/OS/VM/eth_static_config.png" />

1. 新建 VM 或打开已有的 VM 的设置，将网络适配器修改为 `Static`（如果没有网络适配器可以从添加硬件里面新建一个）。

    <img src="/img/OS/VM/vm_static_adapter_settings.png" />

1. 打开 VM，将网络手动配置成 `Static` 适配器所在的 IP 段，并且选择一个不冲突的内网 IP（下面还是展示 Windows 的配置）。

    <img src="/img/OS/VM/static_ip_in_vm.png" />

经过上述步骤，就完成了 VM 内网固定 IP 的配置，并且也可以连接外网服务。只要适配器没有被修改，VM 的 IP 就可以一直保持不变 。

# Reference

* https://stackoverflow.com/questions/63449007/how-to-prevent-the-ip-address-of-hyper-v-virtual-switch-from-being-changed
* http://jimmoldenhauer.blogspot.com/2016/05/hyper-v-internal-network-with-internet.html
* https://superuser.com/questions/1374913/how-to-use-a-windows-pc-to-share-multiple-network-connections
