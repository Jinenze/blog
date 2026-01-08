---
title: 在Github Pages上部署Hexo
lang: zh-CN
date: "2025/04/07"
updated: "2026/01/08"
---
记录一下我在 Github Page 上搭建 Hexo 的过程

[Hexo](https://hexo.io/docs/index.html) 的官方文档似乎年久失修了

英文和中文的情况都差不多

但至少是能看的

>截止2025.4.7

这些是我自己试出来的配置，跟文档里写的不太一样

<!-- more -->
本地部署及测试
---
往 [node](https://hub.docker.com/_/node) 镜像里安装 Hexo
>`~/blog/Dockerfile`
```Dockerfile
FROM node:23-alpine

WORKDIR /blog

RUN npm install -g hexo-cli \
    && hexo init \
    && npm install hexo-theme-next \
    && npm install hexo-server \
    && npm install hexo-generator-searchdb

ENTRYPOINT ["hexo","server"]
```
>-[hexo-theme-next](https://theme-next.js.org/docs/getting-started/)
>-[hexo-server](https://hexo.io/docs/server)
>-[hexo-generator-searchdb](https://theme-next.js.org/docs/third-party-services/search-services)
>
>关于 `CMD` 和 `ENTRYPOINT`
>- [ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint)
>- [Shell and exec form](https://docs.docker.com/reference/dockerfile/#shell-and-exec-form)
>- [init](https://docs.docker.com/reference/compose-file/services/#init)

>`~/blog/compose.yaml`
```yaml
services:
  blog:
    container_name: blog
    environment:
      - TZ=Asia/Shanghai
    build: .
    ports:
      - 4000:4000
    volumes:
      - ./data/_config.yml:/blog/_config.yml
      - ./data/_config.next.yml:/blog/_config.next.yml
      - ./data/source:/blog/source
```
默认端口是4000，可以通过在 compose 文件末尾加上 `command: ["-p","1234"]` 来更改
```bash
cd ~/blog
docker compose up
```
就可以启动容器了

需要注意 alpine 镜像默认没有 `bash`，进入容器调试要用 `sh`
```bash
docker exec -it blog sh
```

SSH
---
设置连接配置以及信任 GitHub 公钥

>`~/.ssh/config`
```
Host blog
  HostName github.com
  IdentityFile ~/.ssh/key_blog
  User git
  IdentitiesOnly yes
```
>`~/.ssh/known_hosts`
>
>公钥可以在这里获取 [GitHub 的 SSH 密钥指纹](https://docs.github.com/zh/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
>
>如果没有预先设置首次连接时会提示是否信任发送过来的公钥
```
github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl
```
>本地生成 ssh 密钥
>
>写这篇文章时官方首选的 ssh 密钥格式是 ed25519
```bash
ssh-keygen -t ed25519 -f ~/.ssh/key_blog
```
GitHub
---
新建 GitHub 仓库，名称随意
```bash
cat ~/.ssh/key_blog.pub
```
`Settings > Deploy keys > Add deploy key` 把 `key_blog.pub` 的内容添加到 deploy key 里

配置
---
到这里就可以开始写配置文件和文章了

`source` 路径下不以 `_` 开头的内容会出现在网站根目录下

比如存储在 `source/images/pic.webp` 的图片可以在 md 文件里用 `/images/pic.webp` 调用

在`source` 文件夹里放一个名称叫 `CNAME` 的文件，在里面写上你的域名，最后把域名 CNAME 解析到 GitHub 给你的域名上就可以用你的域名访问博客了

注意在 Hexo 的配置文件里的域名也需要更改

没有自己的域名的话就不需要这个文件，配置文件里的域名要改成 `用户名.github.io/仓库名` 的格式

>GitHub 给你的域名一般是 `用户名.github.io` 的格式
>
>不知道域名的话可以在部署完成之后在 `Settings > Pages` 里等他验证一下解析
>
>验证失败就会提示你改 DNS 解析记录

`_posts` 目录是存放文章的默认路径

路径下的所有 md 文件开头必须拥有一个 [Front Matter](https://hexo.io/docs/front-matter) ，不然不会显示在网站里
```
---
title: Hello World
---
```
>Next 有对 Front Matter 进行扩展
>
>https://theme-next.js.org/docs/advanced-settings/front-matter

下面的是个标签
```
<!-- more -->
```
如果用户在首页，在这个标签下面的东西会被隐藏并显示一个阅读全文的按钮

记得在这玩意上面空一行

`_posts` 目录下的md文件更改时会直接同步到测试服务器上

`_config` 开头的文件不会，更改的时候需要重启本地服务器

`_config` 开头的文件看文档编写
- [Hexo](https://hexo.io/docs/configuration) `_config.yml`
- [Next](https://theme-next.js.org/docs/theme-settings/) `_config.next.yml`

>官方文档里对 `language` 项的描述只能说确实有写，看 [Next的文档](https://theme-next.js.org/docs/theme-settings/internationalization)

最好不要把 `_config.yml` 里的 `permalink` 项改成 `:title/`，这样会把所有 `_posts` 里的文章全部丢进根目录

我写的是 `:lang/:year/:title/`

如果这样写需要在每篇文章的 `Front Matter` 里指定 `lang` ，或者看官方文档 [i18n Path](https://hexo.io/docs/internationalization#Path) 配置默认设置

>评论和统计数据比较麻烦，之后再说

记得多看看别人是怎么配置的

GitHub Actions
---
新建以下文件
> `~/blog/data/.github/workflows/deploy.yml`
```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '23'

      - name: Install Hexo
        run: |
          npm install -g hexo-cli
          hexo init
          npm install hexo-theme-next
          npm install hexo-generator-searchdb

      - name: Download Source
        run: |
          git clone --branch main --single-branch --depth 1 https://github.com/用户名/仓库名.git tmp-repo
          rm -r source
          cp -r tmp-repo/* .

      - name: Build
        run: hexo generate

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```
>记得把仓库的链接换了

这样配置时，当你推送到 `main` 分支的时候，会触发这个 action

部署时默认部署在 `gh-pages` 分支

Git
---
```bash
cd ~/blog/data
git init
git remote add github blog:用户名/仓库名.git
git push github main
```

最后
---
去仓库的 `Settings > Pages` 里改成 `Deploy from a branch` 并切换到 `gh-pages` 分支

如果一切正常你可以在仓库的 `Code` 页面的右侧看见部署完成的提示，点开会附上目前正在使用的域名
