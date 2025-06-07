---
title: 关于Dockerfile
lang: zh-CN
date: "2025/04/22"
updated: "2025/04/27"
---
主要记录学习过程

学到新东西了就来更新

Dockerfile有很多东西，这只是其中的一部分

<!-- more -->
先来看一个Dockerfile
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

似乎有很多人安装了 `jq` 这个包，说实话我看不懂为什么要装，明明根本没有使用到啊？

删除了一堆关于时区的配置，配置时区只需要使用 `TZ` 环境变量就行，这是linux本身的环境变量

我对时区了解的不是很多，反正跑起来了不是吗

之后再继续更新
