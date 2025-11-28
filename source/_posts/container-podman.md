---
title: Rootless Podman Quadlet 搭建记录
lang: zh-CN
date: "2025/11/26"
updated: "2025/11/28"
---
>注意本文可能具有很强的时效性，请自行查证

最近把容器后端从 Docker 切换到了 Podman ，记录一下受到的折磨。

<!-- more -->
# 前言

Zero trust 安全模型里面有个 Least privilege ，简单来说就是只给程序运行所需要的权限，Docker 的容器毕竟都是 root 用户运行的，不太好。

Podman 与 Docker 最大的不同就是没有一个中心的进程去管理容器们，这也导致 Podman 需要使用 Systemd 这种服务管理器来实现容器的开机自启，这也是 Quadlet 存在的原因中的一部分。

没了中心进程也让系统中的非 root 用户也可以直接创建自己的容器。

Docker 有 Docker compose 来简单地做一些容器的编排工作，我之前就写了十几个 compose.yaml 然后把他们 include 到一个文件里去，差不多像这样：

```yaml
include:
  - ./service1/compose.yaml
  - ./service2/compose.yaml
  - ./service3/compose.yaml
  # - ./service4/compose.yaml
  - ./service5/compose.yaml
```

这导致我在使用 Podman compose 的时候会突然在神秘的地方出现神秘的空文件夹，或者容器不停报错重启，非常折磨。

这是因为 Podman compose 跟 Docker compose 对 include 的相对路径的处理方式不同导致的，前者是完全按你在哪执行 compose up 指令来算，而后者是看 compose.yaml 文件的路径来决定的。

>最开始决定使用 Podman 是因为他对 nftables 的支持，虽然目前我似乎完全没看见他有写防火墙规则

我实在是对这种重复劳动没啥兴趣，在进行了几天的检索之后我决定直接全部切换到 Quadlet 配置文件来管理。

Rootless 和 Rootful 的区别确实大，主要是容器内对宿主机的权限映射问题和 Linux 里的权限管理问题。

# Podman

Podman 在设计时决定了用户与用户之间互不影响，从 Network 和 Pod 这种用来让容器们互相交流的东西居然不能互通这点就能看出来。

>这里我真查得昏天暗地，问 AI 也不行，就像完全没人在意这个问题一样，最后给我翻到了这个 https://github.com/containers/podman/discussions/20408
>
>> This is not possible,
>
>给我气傻了当时

不同用户的容器间通讯只能通过绑定宿主机的端口来进行，或者你也可以把需要相互通讯的容器放到同一个用户下运行。

Podman 中的 Network 和 Pod 的用处差得挺多的，区别在前者类似用来装节点的局域网，后者像局域网里的节点。

Network 可以给 Container 设置也可以给 Pod 设置， Pod 可以给 Container 设置，同一个 Pod 下的 Container 共用端口和 IP 。

Podman 会给所有用户创建一个默认 Network 叫 `podman` ，这个网络没有内置 DNS 不能解析容器名，具体可以用 `podman inspect podman` 来看到网络的配置，其他网络也可以用这个指令。

Podman 的许多配置都在 `/etc/contaiers/containers.conf` 底下，比如这个默认网络的网段和名字，我倒是没找到他的 DNS 的配置。

Quadlet 与直接运行 `podman run` 没有本质上的区别，只是一个持久化的配置文件，这也意味着你在配置 Quadlet 时需要去查 `podman run` 的文档。

# Permission

Rootless 的容器你总不能拿 root 用户跑对吧？需要新建一个用来跑容器的用户，来看指令：

```bash
useradd -b /opt -m -s /usr/bin/nologin -U nginx_container
```

`-b /opt` 用户 home 目录的基础路径，会在 `/opt/nginx_container` 生成用户的 home 目录。

`-m` 生成 home 目录。

`-s /usr/bin/nologin` 用户的登录 shell ，这个 shell 会阻止用户登录，sudo 什么的倒是能用就是了。

`-U` 生成同名用户组。

需要注意的是 Podman 默认会把容器内的 root 用户映射为宿主机运行容器的用户，你在把宿主机的文件映射进去的时候如果运行容器的用户权限不够是无法访问的。

最常见的比如像证书私钥这种需要多容器访问的文件会把所有需要权限的用户添加到这个文件的拥有组里，这时候用户会有两个及以上的组，而 Podman 默认不会处理这种情况。

如果需要访问这样的文件，需要根据你的容器运行时来做不同的处理。

容器运行时可以通过运行 `podman info` 查看，在 `ociRuntime:` 项下。

如果是 crun 的话可以在容器启动时加上 `--userns=keep-id --group-add=keep-groups` 来保证将所有宿主用户的组映射进去，没有 `--userns` 的话 `--group-add=keep-groups` 不会生效。

>这个指令的实现方式说是跳过了创建容器时的一个阶段，我看不太懂，可以自行检索一下
>
>也是因为这个原因 runc 容器运行时并不支持 `--group-add=keep-groups` ，目前只有 crun 支持这个选项
>
>https://github.com/opencontainers/runc/issues/4642
>
>runc 的 star 数要比 crun 高，但前者是 go 写的后者是 c 写的，前者速度没有后者快，但他的速度只体现在容器启动的时候，我个人推荐 runc ，原因请看他们的 github releases 页
>
>https://github.com/opencontainers/runc/releases

runc 就比较麻烦点，看下面的脚本：

```bash
# usermod --add-subgids 1234-1234 container_user
echo "container_user:1234:1" >> /etc/subgid \
&& sudo -u container_user podman system migrate \
&& sudo -u container_user podman unshare cat /proc/self/gid_map \
&& sudo -u container_user podman run --rm --group-add=1234 --gidmap=+g1234:@1234 alpine id
```

`echo "container_user:1234:1" >> /etc/subgid` 添加可以使用的 subgid ，意思是从1234开始数1个，也就是只有1234一个，注释那个是从1234到1234的意思。

`sudo -u container_user podman system migrate` 刷新。

`sudo -u container_user podman unshare cat /proc/self/gid_map` 查看可用 gid ，第二列是开始的地方，第三列是数几个。

`sudo -u container_user podman run --rm --group-add=1234 --gidmap=+g1234:@1234 alpine id` 先给容器内用户添加组 1234 ，然后将宿主机上的组映射进去。

`--gidmap=+g1234:@1234` `+` 是指附加在前一个 gidmap 上，`g` 是限制只添加组，当 `uidmap` 和 `gidmap` 只有一个时会默认两个都映射， `@` 的意思是不删除默认映射的组和用户。

**注意：**添加了这个映射的容器不需要宿主用户在这个 gid 的组里也可以访问这个组拥有的文件。

>这些文档里都有，但实在难找，刷新的那条指令还是我在试了十几遍在报错信息里找到的
>
>虽然麻烦但好像这才是标准操作

如果用了 `--userns=keep-id` 的话这时候容器内的用户就不再是 root 了，而是宿主机上运行容器的用户，这会导致很多没有考虑过 Rootless 情况的容器出现问题，可能需要一定的取舍。

>其实可以改成这样 `--userns=keep-id:uid=0 --group-add=keep-groups` ，这样容器里就是 root 用户带宿主权限组了，能用是能用但我不确定这是不是预期的用法，最好还是直接处理一下容器

容器里的 Linux 也是 Linux ，没什么太大区别，这也意味着如果你容器里用的不是 root 账户也会遇到容器里的权限问题，比如绑定不了1024以下的端口一类的。

将非 root 用户可以使用的最低端口设置为0：

```bash
sysctl -w net.ipv4.ip_unprivileged_port_start=0
```

宿主机上持久化设置需要写入 `/etc/sysctl.d/*.conf` ，具体以自己的系统为准。
```conf
net.ipv4.ip_unprivileged_port_start=0
```
Podman 中直接加上 `--sysctl` 就行，底下有写。

# Systemd

Quadlet 生成的是 Systemd 的配置文件，这也意味着你也得懂一点 Systemd 。

>我真不懂

主要就是用 `systemctl -M contaier_username@ --user start container_name` 启动容器然后 `systemctl -M contaier_username@ --user stop container_name` 停止容器。

Quadlet里也支持你去设置 `reload` 指令的具体实现。

`journalctl -b` 看容器的日志，具体的过滤什么的自己 help 看。

如果容器启动了之后自己停了一般是因为没有使用 `systemctl enable-linger user` 把 linger 打开，防止用户的最后一个 shell 消失之后 Systemd 自己停了。

可以 `ls /var/lib/systemd/linger` 看哪些人开了 linger 。

不能在 Systemd 这里把容器服务设置为开机自启， Quadlet 里可以设置。

# Quadlet

Qaudlet 是用于生成 Systemd unit 的一种配置文件，在 `systemctl daemon-reload` 时会读取特定路径下的特定后缀文件来生成 unit 配置，具体的可以直接 `man podman-systemd.unit` 来看，我这里给简单介绍下。

可以写一些和 Systemd 的 unit 一样的配置。

来看一个 rootless 的 nginx 容器配置：

>nginx.container
```toml
[Container]
ContainerName=nginx
Image=docker.io/nginx:1.29
Network=podman
PublishPort=80:80
PublishPort=443:443
Volume=/opt/nginx_container/data:/etc/nginx
Volume=/var/opt/certs:/etc/nginx/certs
PodmanArgs=--userns=keep-id --group-add=keep-groups
PodmanArgs=--sysctl net.ipv4.ip_unprivileged_port_start=0

[Install]
WantedBy=default.target
```

`ContainerName` 控制容器名与同一网络下的 Podman 提供的 DNS 解析名

`Network` 容器所属的网络，一般为 `host` `podman` 或者 `none`  ， `podman` 为 Podman 为每个用户单独创建的不带 DNS 解析的网络，详细的上面写了。

`PodmanArgs` 为 `podman run` 后面跟着的 flag，详细的上面写了。

这里保留权限主要是为了访问网站的证书，注意 nginx 在 rootless 环境下需要额外配置，可以看 docker 仓库中的 nginx 的简介。

以及这里拆成俩只是因为一行有点长。

`WantedBy=default.target` 设置开机自启，我不懂 Systemd 不知道什么意思。

network pod 啥的配置差不多的自己去查文档 `man podman-systemd.unit` 。

# 实操

>这里是 crun 作为容器运行时时的配置，不建议
```bash
useradd -b /opt -m -s /usr/bin/nologin -U nginx_container \
&& mkdir /opt/nginx_container/.config \
&& mkdir /opt/nginx_container/.config/containers \
&& mkdir /opt/nginx_container/.config/containers/systemd \
&& echo "[Container]
ContainerName=nginx
Image=docker.io/nginx:1.29
Network=podman
PublishPort=80:80
PublishPort=443:443
Volume=/opt/nginx_container/data:/etc/nginx
Volume=/var/opt/certs:/etc/nginx/certs
PodmanArgs=--userns=keep-id --group-add=keep-groups
PodmanArgs=--sysctl net.ipv4.ip_unprivileged_port_start=0

[Install]
WantedBy=default.target" > /opt/nginx_container/.config/containers/systemd/nginx.container \
&& chown -R nginx_container:nginx_container /opt/nginx_container \
&& systemctl -M nginx_container@ --user daemon-reload \
&& systemctl enable-linger nginx_container \
&& systemctl -M nginx_container@ --user start nginx \
&& journalctl -b -n 100
```

>这是 runc 作为容器运行时的配置
```bash
useradd -b /opt -m -s /usr/bin/nologin -U nginx_container \
&& groupadd certs \
&& mkdir /opt/nginx_container/.config \
&& mkdir /opt/nginx_container/.config/containers \
&& mkdir /opt/nginx_container/.config/containers/systemd \
&& echo "[Container]
ContainerName=nginx
Image=docker.io/nginx:1.29
Network=podman
PublishPort=80:80
PublishPort=443:443
Volume=/opt/nginx_container/data:/etc/nginx
Volume=/var/opt/certs:/etc/nginx/certs
PodmanArgs=--group-add=$(id -g certs) --gidmap=+g$(id -g certs):@$(id -g certs)

[Install]
WantedBy=default.target" > /opt/nginx_container/.config/containers/systemd/nginx.container \
&& chown -R nginx_container:nginx_container /opt/nginx_container \
&& echo "nginx_container:$(id -g certs):1" >> /etc/subgid \
&& sudo -u nginx_container podman system migrate \
&& sudo -u nginx_container podman unshare cat /proc/self/gid_map \
&& sudo -u nginx_container podman run --rm  alpine id \
&& systemctl -M nginx_container@ --user daemon-reload \
&& systemctl enable-linger nginx_container \
&& systemctl -M nginx_container@ --user start nginx \
&& journalctl -b -n 100
```

>反正不是复制下来一运行就能用的脚本就是了哈哈 主要是看看需要干什么

# 结尾

我也不知道我在干什么

还挺烦的这套流程， 怀念 docker compose 随便写两下就能跑的感觉

总之如果有发现错误欢迎来仓库发个 issue

>2025-11-28

不是哥们， crun 的依赖里面怎么能有 wayland pipewire mesa 这堆东西的？吓得我直接换 runc 了

所以说libkrun 为什么会依赖图形类的运行库啊？ issue 里也没人提吗？

回来更新一下，看看什么时候有人提这事
