---
title: 关于Dockerfile
lang: zh-CN
date: "2025/04/22"
updated: "2025/06/07"
---
主要记录学习过程

学到新东西了就来更新

Dockerfile有很多东西，这只是其中的一部分

<!-- more -->
Docker这东西说简单也不简单说难也不难

实际上不是一般不是Docker难而是你想在容器中跑的东西难配置

以下是一些实操

先来看一个别人写的Dockerfile
```Dockerfile
FROM python:3.11.11-slim-bookworm as build

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV SET_CONTAINER_TIMEZONE=true
ENV CONTAINER_TIMEZONE=Asia/Shanghai
ENV TZ=Asia/Shanghai 


ARG TARGETARCH
ARG VERSION
ENV VERSION=${VERSION}
ENV PYTHON_IN_DOCKER='PYTHON_IN_DOCKER'

COPY scripts/* /app/
WORKDIR /app

RUN apt-get --allow-releaseinfo-change update \
    && apt-get install -y --no-install-recommends jq chromium chromium-driver tzdata \
    && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo $TZ > /etc/timezone \
    && dpkg-reconfigure --frontend noninteractive tzdata \
    && rm -rf /var/lib/apt/lists/*  \
    && apt-get clean

COPY ./requirements.txt /tmp/requirements.txt

RUN mkdir /data \
    && cd /tmp \
    && python3 -m pip install --upgrade pip \
    && PIP_ROOT_USER_ACTION=ignore pip install \
    --disable-pip-version-check \
    --no-cache-dir \
    -r requirements.txt \
    && rm -rf /tmp/* \
    && pip cache purge \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /var/log/*

ENV LANG=C.UTF-8

CMD ["python3","main.py"]
```
槽点很多，多到数不过来

直接看修改后
```Dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    TZ=Asia/Shanghai

WORKDIR /app

COPY scripts/* .

RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    apt-get update && apt-get install -y --no-install-recommends \
    chromium \
    chromium-driver \
    && pip install --no-cache-dir -r /tmp/requirements.txt \
    && rm -rf /var/lib/apt/lists/*

ENTRYPOINT ["python3","main.py"]
```
Python3.7后不再需要 `PYTHONUNBUFFERED`
>https://docs.python.org/3/using/cmdline.html#cmdoption-u

Debian官方镜像只需要清理 `apt update` 时下载的目录即可

镜像中已经配置了自动清理deb包缓存
>在这个目录下 /etc/apt/apt.conf.d/docker-clean

对 `requirements.txt` 的调用换成了官方最佳实践的示例
>https://docs.docker.com/build/building/best-practices/#add-or-copy

将 `CMD` 换成了 `ENTRYPOINT`
>- [ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint)
>- [Shell and exec form](https://docs.docker.com/reference/dockerfile/#shell-and-exec-form)
>- [init](https://docs.docker.com/reference/compose-file/services/#init)

`jq` 这个包是用来在命令行中解析json的，很明显这里只有一个需要运行的python程序

我不知道为什么要加，反正删了没出问题

删除了一堆关于时区的配置，配置时区只需要使用 `TZ` 环境变量就行

`TZ` 是linux自带的环境变量而不是Docker添加的

我对时区了解的不是很多，反正跑起来了不是吗

他在python程序里读取了环境变量中的版本号

其实版本号不只这个地方有与其发布版本的时候打进去不如手动改

`FROM ... as ...` 这里的 `as` 记得写成 `AS`

大小写要规范

其他案例可以看我的其他blog

之后再继续更新
