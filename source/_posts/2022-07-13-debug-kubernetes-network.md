---
title: Debug Kubernetes Network Timeout
date: 2022-07-13 08:33:32
tags: [Debug, Kubernetes, Network]
categories: Debug
---

事故现象：服务的 nginx 出现大规模的请求超时，从 log 上看 upstream 都是相同的 ip，从 metrics 上看有一个 Kubernetes Node 拿不到 usage 信息

# 猜想与疑惑

1. 看到这个现象的第一反应是 node 的网络出问题，也怀疑是不是 node 在重启
1. 疑惑的一点是 Kubernetes 为什么不会把 node 摘掉？
1. 除了 node 外，service pod 也配置了 readiness probe，请求失败的 pod 应该不会被 call 到

# Debug

由于生产环境的问题一段时间自动恢复，没有现场，只能自己尝试在本地复现

## 重启 Node

1. 查资料发现，kubernetes 有个启动参数为 [`pod-eviction-timeout`](https://kubernetes.io/docs/concepts/architecture/nodes/#condition)，node 只要在 timeout 前恢复就不会驱逐掉它上面的 pod
1. 经过重启测试，重启的时候 TCP 返回的 error 是 refused，和 timeout 不匹配，因此排除

## 网络测试

因为 Kubernetes 没有摘掉 node 和 pod，我倾向是 node inbound 出问题而 local 和 outbound 都是正常的，因此先尝试阻断 node 的 inbound tcp

1. 建 debug pod，并进入 host network:
    ```bash
    kubectl apply -f https://github.com/ChenTanyi/script/raw/master/kubernetes/debug.yml
    chroot /host bash
    ```
1. 阻断到特定 IP 的 TCP 包
    * 因为我们是 attach 到 node 上 debug，因此不能直接阻断 node 上的所有 TCP 包
    * `iptables` 有三条链 `INPUT`, `FORWARD`, `OUTPUT`，要注意区分 host 和 container 走的是不同的链
    ```bash
    iptables -I FORWARD -p tcp -j DROP -d x.x.x.x
    ```

测试后的结论：

1. kubelet 会收集该 node 上面所有 cluster 信息，如 readiness probe，所有网络请求只经过本机的网络，一般不会出现网络问题
1. kubelet 会定时向 api server 上报信息，因此只要 outbound 正常的情况下，kubernetes 会认为这个 node 是没有问题的
1. metrics server 会主动向每个 node 上的 kubelet 请求 metrics，因此如果 metrics server 不在这个 node 上，这个 node 的 metrics 就会受 inbound 的影响

## 修复 nginx

根据上面的测试，问题就定位到 node 的 inbound 处，但是按照 kubernetes 的设计，outbound 正常的情况下，node 和上面的 pod 就被认为是正常运行的。在找不到方法摘掉受影响的 node 的情况下，目标转向修改 nginx 配置避免 timeout。想起 nginx 有两个相关的 upstream 配置 `max_fails` 和 `fail_timeout`，查了一下，[nginx ingress 不支持配置这两个参数](https://github.com/kubernetes/ingress-nginx/issues/4773)，不过默认配置应该是完全足够的，应该是没有触发 timeout 这个失败条件

> 所以最终把原来的 `connect_timeout` 缩短，使得 nginx 在 timeout 之后可以发给正常的节点（理论上同个 cluster 内部的情况下，connect_timeout 不应该设太长）

# Reference

* https://kubernetes.io/
* https://arthurchiao.art/blog/deep-dive-into-iptables-and-netfilter-arch-zh/
* http://nginx.org/en/docs/http/ngx_http_upstream_module.html
