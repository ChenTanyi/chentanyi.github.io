---
title: 通过 Serverless 服务简易地搭建一个 Proxy
date: 2021-05-02 20:13:04
tags: [Proxy, Serverless]
categories: Proxy
---

> **_！！！警告_** ：本文的 demo 仅为原理展示，没有任何安全认证，不建议长时间直接暴露在外网使用。（直接使用后果自负）

[Proxy](https://en.wikipedia.org/wiki/Proxy_server) 是一种特殊的网络服务，服务器不提供内容服务，而是进行流量转发。Proxy 可以用于：网络加速、访问限制与突破、隐藏IP、负载均衡等场景

[Serverless](https://en.wikipedia.org/wiki/Serverless_computing) 是一个云服务概念，通过对后端计算资源的隐藏，可以方便地部署特定的 HTTP 服务，并且大部分的 serverless 服务提供入口的 HTTPS 服务保证通信安全，使得用户不需要自行申请与安装 HTTPS 证书

所有能通过 HTTP/ws (Websockt 可以兼容 HTTP) 协议提供服务的 Proxy 都可以使用该方式搭建（不过，由于标准的 HTTP Proxy 协议不与负载均衡层如 nginx 兼容，故不可直接使用）

本文会介绍如何简易地搭建一个 Websocket + HTTP Proxy 的服务，我们使用的 proxy 服务是 [proxy.py](https://github.com/abhinavsingh/proxy.py)，websocket 服务是 [websockify](https://github.com/novnc/websockify) —— 它会以 websocket 协议监听一个 TCP 端口，并将收到的所有消息转发到另一个 TCP 端口上，从而实现了在 HTTP Proxy 的外面套上 Websocket 协议

# Docker

[Docker](https://www.docker.com/) 是一个 Linux 容器相关的工具，具有强大的可移植性，不受宿主机的运行环境影响，可以方便地在不同云服务商之间切换，大部分大型云服务商都会提供基于容器的 Serverless 服务。推荐大家使用 docker 运行 serverless 服务

我已经构建好一个 demo 用的 docker 镜像 [chentanyi/simple-proxy](https://hub.docker.com/r/chentanyi/simple-proxy)，当然你也可以选择用下面的 Dockerfile 构建自己的 docker 镜像:

```Dockerfile
from python:alpine
run apk add --no-cache --virtual .build-deps g++ && \
    pip install --no-cache-dir proxy.py websockify && \
    apk del .build-deps && \
    printf "proxy --hostname 0.0.0.0 --port 8899 &\nwebsockify 80 :8899" > /start.sh

cmd ["sh", "-c", "chmod +x /start.sh && /start.sh"]
```

这个 docker 会在启动的时候运行两个服务：`proxy` 运行在后台，监听 8899 端口，提供 HTTP Proxy 服务；`websockify` 运行在前台，监听 80 端口，提供 websocket 服务，并将消息传到 8899 的 HTTP Proxy 上。两者通过 `start.sh` 启动

docker 具体使用不做过多展开，可以参考其他文章，我们的具体操作并不依赖 docker 相关知识（如果大家有需求我也可以另写一篇介绍介绍）

# Azure App Service

具体的 Serverless 服务使用 Azure App Service 中的 Web App 作为例子（因为我只使用过这一个），Web App 要求 docker 监听标准 HTTP 端口 80 或者通过配置 `WEBSITES_PORT` 指定对应的 HTTP 端口

要使用 Azure 服务之前，必须要有一个 Azure 账号和可以创建资源的 Subscription。如果没有并想尝试的话，可以注册一个账号，使用 Azure 提供的 [1年+$200](https://azure.microsoft.com/free/) 的试用服务

## Java SDK
> 非开发者请跳到下面 [Portal](#Portal)

对于开发者，可以使用 SDK / Cli 进行 Web App 的创建，这里以 [Java SDK](https://github.com/Azure/azure-sdk-for-java/tree/master/sdk/resourcemanager) 为例，在做完认证之后，可以非常简单地使用以下代码创建对应的 Web App:

> webappName 与最终的网址相关，必须为全局唯一（如果与别人重名的话，创建时会报错）

```java
azure.webApps().define(webappName)
    .withRegion(region)
    .withNewResourceGroup(resourceGroupName)
    .withNewLinuxPlan(appServicePlanName, pricingTier)
    .withPublicDockerHubImage("chentanyi/simple-proxy")
    .create();
```

当然我也写好了 [demo project](https://github.com/tanyi-samples/azure-webapp-simple-proxy)，在 JDK 8 以上和 maven 的环境下，修改变量后就可以直接 `mvn exec:java` 跑起来了

## Portal

对于普通用户，可以选择使用 [Azure Portal](https://portal.azure.com) 创建 Web App（截图使用的是英文版的 portal，中文版界面应该类似）

1. 在 portal 页面，点击创建，选择 Web App
    <img src="/img/Azure/Proxy/create_resource_1.png" />
    <img src="/img/Azure/Proxy/create_resource_2.png" />

2. 依次填写必要的信息，如果然后选择 `create/创建`
    <img src="/img/Azure/Proxy/create_resource_3.png" />
    <img src="/img/Azure/Proxy/create_resource_4.png" />

3. 等大概几分钟 Web App 创建完成后，我们的服务就搭建好了，需要点进 Web App 页面，复制对应的网址以便后面连接使用
    <img src="/img/Azure/Proxy/webapp.png" />

# Client

由于我们建的不是一个标准的 HTTP Proxy 服务，因此我们无法用大部分工具自带的 HTTP Proxy 进行连接，因此我们需要在本地搭建一个服务，将标准 HTTP Proxy 放到 websocket 里进行传输。

最简单的操作是类似 websockify 写一个把 TCP 发送到 websocket 的服务，但因为没找到著名的开源项目可以完成该操作，因此我们使用 [v2ray](https://github.com/v2fly/v2ray-core) 来完成类似的需求。

v2ray 是一个强大的网络工具平台，可以实现许多 Proxy 的解析与转换，我们可以通过它将发出的 HTTP Proxy 或 Socks Proxy 转换为 Websoket + HTTP Proxy 再传到我们搭建好的 Serverless 服务上，这样我们就可以打通整条链路以实现 Proxy 功能。

## Windows x64

由于 v2ray 的 [release](https://github.com/v2fly/v2ray-core/releases) 有几乎全平台的版本，使用方式基本一致，因此这里仅用 windows x64 平台作展示（移动端不能直接跑 golang 程序，请自行在商店查找对应的移动版）

直接下载 windows x64 对应的[下载包](https://github.com/v2fly/v2ray-core/releases/latest/download/v2ray-windows-64.zip)并解压到合适的位置即可，其中我们唯一需要修改的是 `config.json`

### Config

v2ray 有非常多的 config 可以配置，我们这个 demo 仅会用到它的 inbound 和 outbound 进行传入和传出协议的转换，再加上 log 显示出来就足够了，以下是 config 样式，直接粘贴到 `config.json` 内（要先把已有内容删除），并将 `"proxy-demo-webapp.azurewebsites.net"` 修改为上面 Portal 复制出来的网址即可（网址需要把前面的`https://`和后面的`/`都删掉）

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 8000,
      "protocol": "http"
    }
  ],
  "outbounds": [
    {
      "protocol": "http",
      "settings": {
        "servers": [
          {
            "address": "proxy-demo-webapp.azurewebsites.net",
            "port": 443
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls"
      }
    }
  ]
}
```

简单介绍下上面的配置详情：inbound 和 outbound 是链路最重要的部分，inbound 负责监听不同端口对应的 proxy 协议，比如 HTTP，然后解包出相应的内容，并将其包装成另外的 proxy 协议从 outbound 送出，outbound 的 proxy 需要有服务器的地址与端口，从而将 TCP 送到对应的服务器上。如果有多个 outbound，需要自行定义不同的路由规则以匹配不同的 outbound（inbound 由端口匹配，而 outbound 只能通过规则匹配）。在 proxy 的基础上，还可以为其套上不同的传输协议，如我们用到的 websocket，因为我们 webapp 的 docker 前面还有 LB 层的 TLS，因此我们的传出协议在 ws 的基础上还需要有加密协议 tls。

### Test

修改完 `config.json` 后，双击 `v2ray.exe` 便可以启动 v2ray。（如果闪退了，可以按住 shift + 右键，打开 powershell，并跑 `./v2ray.exe`，看对应的报错）

<img src="/img/Azure/Proxy/v2ray_click.png" />
<img src="/img/Azure/Proxy/v2ray_powershell.png" />

> 比如我上面的截图就是意味着某一个 inbound 监听的端口 8000 不可用了，可能需要换个端口

按照我们上面的配置，只需要设置 HTTP Proxy 为本地 8000 端口就可以了，我们可以简单地采用 curl 测试：

<img src="/img/Azure/Proxy/curl.png" />

当然我们也可以通过[修改系统代理](https://www.kuaidaili.com/doc/using/win10/)的方式用浏览器进行测试：

<img src="/img/Azure/Proxy/proxy.png" />
<img src="/img/Azure/Proxy/ipaddress.png" />

可以看到，使用代理后我们的出口 IP 变成了 Azure 大阪的 IP 了，就意味着我们的 Proxy 搭建成功了

不过由于 webapp 是第一次请求时才启动的（免费版请求后服务也不是常驻的），代理请求可能会有一定的延迟

> 这就是全部 demo 的内容，有什么相关问题欢迎在 [github issue](https://github.com/chentanyi/chentanyi.github.io/issues) 提出

# 后记

这个 demo 写起来有点长，不过正常操作起来应该是很快就可以完成的，得到的是可以使用的不带认证的代理，意味着这个代理的所有信息都暴露在外网，请不要用作测试以外的任何用途。如果想要更安全稳定的服务，请自行修改配置搭建 Proxy 服务。

其实本来想写写最近玩了哪些代理与自动化脚本，后来想想人在屋檐下，还是不写太傻瓜化的东西了，防止人没了（当然这个 demo 还是傻瓜化，只是直接用会有问题），而且网上已经有太多教程了，不如在介绍使用的过程中多聊聊一些基本的原理，想要布一个稳定的 Proxy 最好还是基于自己对于网络和协议的理解。

PS: v2ray 是个功能很强大的代理网络工具，配置特别复杂，可以实现很多代理功能，具体可以参考官网文档 https://www.v2fly.org/

PPS: v2ray 也可以一个工具充当 websokect + proxy 的角色运用到 webapp 的服务内，可以参考[别人的文档](https://ibcl.us/Heroku-V2Ray_20191014/)
