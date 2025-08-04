---
title: 关于使用 ZeroTier 进行异地组网
lang: zh-CN
date: "2025/04/05"
updated: "2025/08/04"
---
# 前言
>以下内容仅适用于 ZeroTier 1.14

ZeroTier 可以将设备添加至一个由 ZeroTier 网络控制器控制的虚拟局域网

通过协调服务器的协调来进行节点间的直连

当你在你的家庭服务器上部署了个人网盘/密码托管/即时通讯/游戏服务器这样的服务时

想在楼下的餐馆里，其他城市的酒店里，你朋友的家里安全地访问到这些服务

这时候就可以使用 ZeroTier 来构建一个虚拟局域网

在国内环境下这种方案也可以降低一定的搭建个人信息平台时的法律风险

<!-- more -->
>关于国内ICP备案相关的信息我知道的不多
>
>这种方式只是避免直接打开公网端口，降低一定的法律风险的同时封锁公网扫描的可能性
>
>如果需要对公网部署还请购买云服务器，ICP备案针对的是服务器

开讲之前先来点免责声明

以下内容皆为个人经验绝对不是什么教程

主要是记录一些踩过的坑防止别人再踩一遍

以及我在编写前没有阅读过源码，如果有错误可以来仓库提个 issue

异地组网
---
异地组网的存在本身是为了解决异地设备之间的加密问题

异地的两台设备在互相访问时会有大量数据经过公网

不对敏感数据进行加密无异于裸奔

广泛应用的 SSL/TLS 加密需要一个被信任的第三方机构签发的证书

这在小型网络中昂贵麻烦且无必要

[WireGuard](https://www.wireguard.com/) 的出现就是为了解决这样的问题

让两台处在不同环境的设备获得在局域网中一样的安全通讯

但连接时打开的面向公网的端口也可能成为麻烦的根源

于是组网软件出现了

他们简化了流程，让连接过程不再需要设备允许公网流量进入

粗略地说，就是让两台需要连接的设备分别向可以直连的协调服务器建立连接

再让双方代替协调服务器的位置直接进行通讯

整个流程不需要终端设备接收任何的入站流量，只有对协调服务器发出的出站流量

软件选择
---
目前主流的组网软件有 [ZeroTier](https://www.zerotier.com/) [Tailscale](https://tailscale.com/) 和 [EasyTier](https://easytier.cn/)

关于 Tailscale 可以看看这个视频 [能与ZeroTier齐名的内网穿透工具Tailscale，比ZeroTier还好用？](https://www.bilibili.com/video/BV13P411q7Cy)

我个人不太喜欢 Tailscale 的管理模式，而且必须使用第三方账号注册使用，还得在每个设备上登录

EasyTier 看上去确实不错，但我有些 ZeroTier 的沉没成本在导致没有什么动力去尝试了

同时去中心化的设计也会造成一些问题，比如配置麻烦一类的

我只是抛出一点选项来让你做对比，请结合自身情况来找到最适合你的选项

# 理论

这些视频给了我很大帮助，但好像只有已经会了的人才看得懂，我做点补充

[内网穿透工具ZeroTier，从简单到复杂的玩法，无保留，一期全放送](https://www.bilibili.com/video/BV1Vh411F7Mr)

[这么良心的开源、内网穿透工具ZeroTier，为啥到你手就不好用了？](https://www.bilibili.com/video/BV1oL411Y7pB)

半自部署
---
半自部署简单来说就是部署 [网络控制器](https://docs.zerotier.com/controller)

半自部署最大的好处是没有官方网络控制器的设备数和网络数上限，并不会有安全性的提升

因为连接的planet服务器依旧是别人的，在连接网络时使用的是 **节点id+网络id** 的格式

而节点id是由planet服务器分发的，理论上planet可以把你导向另外的控制器

>个人理解，总之就是planet上运行的代码对用户来说不透明，在使用其他人的提供的服务前请确认你信任他

网络控制器控制着几乎所有的ZeroTier功能，比如IP分配，路由表和DNS配置下发

ZeroTier的客户端本身就拥有成为网络控制器的能力

你可以把他部署在一个始终在线的节点上，比如你家路由器

在连接到官方的网络管理界面 https://my.zerotier.com/ 时，实际上就是连接到了官方分配给你的节点

我在这个方案上花了很多时间，最后发现其实挺简单

只要部署 [ztncui](https://github.com/key-networks/ztncui) 就行

详细看下方实操章节

全自部署
---
全自部署和半自部署的区别就是直接部署了自己的planet服务器

这样你可以看到所有你正在运行的代码，并且所有的数据和权限都在自己手里

这是社区制作的一键部署包 [docker-zerotier-planet](https://github.com/xubiaolin/docker-zerotier-planet)

他会同时部署planet服务器，moon服务器，网络控制器管理界面 [ztncui](https://github.com/key-networks/ztncui)，以及官方客户端

但这个方案服务器ip变动时就需要进行重新部署，并且需要暴露端口

既然都暴露端口了，在我的情况下不如直接配置 [ddns-go](https://github.com/jeessy2/ddns-go) ，解析到公网ipv6从公网连接到服务器

这个方案没什么好讲的，下载下来一键启动就行

# 实操

ztncui
---
>注意，ztncui的GitHub仓库上次更新时间是 [2023.8.31](https://github.com/key-networks/ztncui/commit/1b2284864de48d2dcae22582fff122fe24909c3d)

我这里提供Docker的部署方法，因为我的环境不方便直接安装npm

```Dockerfile
FROM node:23-alpine AS builder

RUN apk add --no-cache git python3 build-base \
    && git clone --branch master --single-branch --depth 1 https://github.com/key-networks/ztncui.git

WORKDIR /ztncui/src

RUN npm -g install node-gyp \
    && npm install

FROM node:23-alpine

COPY --from=builder /ztncui /ztncui

WORKDIR /ztncui/src

ENTRYPOINT ["npm","start"]
```
```yaml
services:
  ztncui:
    build: .
    container_name: ztncui
    restart: unless-stopped
    environment:
      - HTTP_ALL_INTERFACES=yes
    network_mode: host
    volumes:
      - ./passwd:/ztncui/src/etc/passwd
      - /var/lib/zerotier-one/authtoken.secret:/var/lib/zerotier-one/authtoken.secret
```
>这两个文件一个是web面板的用户密码，一个是ZeroTier客户端的访问密钥
>
>默认端口 `3000`，能用 `HTTP_PORT=3456` 改
>
>`HTTP_ALL_INTERFACES` 意思是任何ip地址都能访问，方便nginx反向代理
>
>详细设置请看官方仓库 [ztncui](https://github.com/key-networks/ztncui)

所有功能都是官方客户端实现的，所以配置文件只有密码

启动后打开网页点左上角的 `Add network` 就可以添加网络

少量设备其实没必要使用这个功能，而且在我的情况下必须先连接到网络才能打开管理面板

DNS
---
ZeroTier虽然不需要去在意节点本身的ip地址，但连接其他节点时使用的还是虚拟内网中的ip地址

如果不想记忆ip地址，想使用域名访问服务器的话，在web面板里有一个DNS配置

![DNS](/images/ZTDNS.webp)

Search Domain里填写你想要解析的域名，再在Server Address里填写DNS服务器的ip就可以

如果你想访问 `123.zt` 或 `www.123.zt` ，那么你就可以填写 `zt` 或 `123.zt`

因为我有一台openwrt软路由，使用的默认的DNS服务器 `Dnsmasq`

所以我没有自己部署过DNS服务器，你需要自己部署一个

>官方控制器一个Server只能同时对应一个Search Domain

>其实还有很多方案，比如
>- 买个域名然后解析到ZeroTier内网地址
>- 分发hosts文件
>- 修改本地DNS配置

局域网联机
---
在使用ZeroTier进行局域网联机时有时不能发现其他设备，可以尝试调整 [Windows 网络连接优先级设置](https://www.techkoala.net/windows_10_network_priority/)

默认路由
---
如果想让网络内的设备像在家里的局域网里一样通过网络内的路由器上网

官方有一篇教程 [VPN Exit Node](https://docs.zerotier.com/exitnode)

简单来说就是把路由规则设置成以下这样

![Routes](/images/ZTRoutes.webp)

然后将终端设备的allowDefault设置为1

Windows的长这样

![Windows](/images/ZTWindowsConfig.webp)

官方安卓客户端上长这样

![Android](/images/ZTAndroidConfig.webp)

引用一下DeepSeek生成的
>ZeroTier中的`allowDefault`设置名称源于其功能目的，具体解析如下：
>
>1. **命名来源解析**：
>   - **"Default"（默认）**：指代计算机网络中的**默认路由（Default Route）**。默认路由是当目标地址不在其他路由规则中时，数据包选择的路径，通常用于连接互联网（如`0.0.0.0/0`）。
>   - **"Allow"（允许）**：表示该设置是一个权限开关，控制是否**允许ZeroTier修改系统的默认路由**> 。
>
>2. **功能作用**：
>   - 当`allowDefault`设置为`true`时，ZeroTier会将自身的虚拟网络接口（如`zt0`）添加为系统的默认路由。这意味着**所有未匹配到其他路由规则的流量**（包括互联网访问）都会被导向ZeroTier网络。
>   - 若设置为`false`，则ZeroTier不会干预系统的默认路由，设备通过原有网络接口（如Wi-Fi、以太网）访问互联网。
>
>3. **典型应用场景**：
>   - **VPN类似功能**：启用`allowDefault`后，所有流量经过ZeroTier网络，适合需要统一流量出口或加密的场景（如通过企业服务器访问外网）。
>   - **隔离网络通信**：禁用此设置时，ZeroTier仅处理与虚拟网络相关的流量（如`10.147.17.0/24`），其他流量保持本地直连，避免影响正常上网。
>
>4. **注意事项**：
>   - **潜在风险**：若ZeroTier网络未配置正确出口节点（如缺少公共IP转发），启用`allowDefault`可能导致设备断网。
>   - **配置建议**：通常仅在需要全局流量管控时启用，普通组网场景建议保持关闭，仅通过特定路由规则（如`route 0.0.0.0/0 via 网关IP`）定向部分流量。
>
>**示例配置**：
>```json
>// 在ZeroTier网络配置的routes部分：
>{
>  "routes": [
>    { "target": "10.147.17.0/24", "via": null },  // ZeroTier局域网直连
>    { "target": "0.0.0.0/0", "via": "10.147.17.1", "allowDefault": true }  // 默认路由经网关
>  ]
>}
>```
>此配置中，`allowDefault: true`允许将`10.147.17.1`设为默认网关，所有非本地流量通过该节点转发。
>
>总之，`allowDefault`的命名直观反映了其核心功能：**允许（Allow）ZeroTier接管系统的默认（Default）路由**，这一设计既符合技术术语，也遵循了软件配置的常见命名规范。

Docker 部署
---
我认为一般用户没必要看这个，这个东西本身其实是我在摸索的时候走歪的一条路

一般来说直接安装官方包就行

如果实在不行，可以继续看

在 Docker 下安装有三条路
- 部署 [官方镜像](https://hub.docker.com/r/zerotier/zerotier)
- 在 Debian 镜像下安装官方包
- 在 Alpine 镜像下自行编译源码
>Alpine 似乎本来是有官方包的，后来被撤了

第一条路我死活跑不起来，第二条路镜像体积太大

下面是我摸索很久走出来的第三条路

```Dockerfile
FROM alpine:latest AS builder

RUN apk add --no-cache curl git make linux-headers openssl-dev pkgconf gcc g++ \
    && curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
    && source "$HOME/.cargo/env" \
    && git clone --branch main --single-branch --depth 1 https://github.com/zerotier/ZeroTierOne.git \
    && cd ZeroTierOne \
    && make -j

FROM alpine:latest

RUN apk add --no-cache libstdc++ gcompat openssl libgcc

COPY --from=builder /ZeroTierOne/zerotier-one /usr/sbin/zerotier-one
COPY --from=builder /ZeroTierOne/zerotier-cli /usr/sbin/zerotier-cli

ENTRYPOINT ["zerotier-one"]
```
```yaml
services:
  zerotier:
    build: .
    container_name: zerotier
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./data:/var/lib/zerotier-one
      - /dev/net/tun:/dev/net/tun
    privileged: true
```
构建镜像时的CPU需求量很高，记得放在主力机上构建

这是我试出来的最小配置，不嫌烦的可以自己试试

以及我给了这个容器 privileged 权限，不喜欢的可以自己试试

顺带一提，我现在用的是 OpenWrt 仓库里的软件包

# 结语

目前这套解决方案对于我来说已经足够了，所以我没有仔细研究过其他的解决方案

虽然东西看上去不多，但是折磨了我很久，我没能找到一篇能从头到尾给我讲明白的文章

(其实现在也讲得不太明白，以后可能还会回来更新)

这篇文章也引用了很多别人的东西，我不觉得我能讲得比别人好

我很感谢他们，希望我也能帮助到你

总之我写了这篇文章，如果有帮到你或者你有什么需要补充的，可以给我发个邮件提个 issue 什么的
