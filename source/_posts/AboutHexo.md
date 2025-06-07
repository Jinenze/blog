---
title: 关于Hexo在Github Pages上的搭建
lang: zh-CN
date: "2025/04/07"
updated: "2025/04/27"
---
记录一下我在Github Page上搭建Hexo的过程

[Hexo](https://hexo.io/docs/index.html) 的官方文档似乎年久失修了

英文和中文的情况都差不多

但至少是能看的

>截止2025.4.7

这些是我自己试出来的配置，跟文档里写的不太一样

因为我的环境不方便直接安装，我是在Docker容器中安装的

<!-- more -->
OpenSSH
---
设置连接配置以及信任github公钥
```bash
mkdir .ssh
```

>`~/website/blog/.ssh/config`
```
Host github.com
  HostName github.com
  IdentityFile ~/.ssh/key_blog
  User git
  IdentitiesOnly yes
```
>`~/website/blog/.ssh/known_hosts`
>
>公钥可以在这里获取 [GitHub 的 SSH 密钥指纹](https://docs.github.com/zh/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
```
github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl
```
>本地生成ssh密钥

>写这篇文章时官方首选的ssh密钥格式是ed25519
```bash
cd ~/website/blog
ssh-keygen -t ed25519 -f ./key_blog
mv key_blog .ssh/
```
Docker
---
往 [node](https://hub.docker.com/_/node) 镜像里安装Hexo
>`~/website/blog/Dockerfile`
```Dockerfile
FROM node:23-alpine

WORKDIR /blog

RUN npm install -g hexo-cli \
    && hexo init \
    && npm install hexo-theme-next \
    && npm install hexo-server --save \
    && npm install hexo-deployer-git --save \
    && npm install hexo-generator-searchdb \
    && apk add --no-cache git openssh-client \
    && git config --global user.name "Jinenze" \
    && git config --global user.email "admin@jinenze.xyz"

COPY .ssh /root/.ssh

ENTRYPOINT ["hexo","server"]
```
>- [hexo-theme-next](https://theme-next.js.org/docs/getting-started/)
>- [hexo-server](https://hexo.io/docs/server)
>- [hexo-deployer-git](https://hexo.io/docs/one-command-deployment)
>- [hexo-generator-searchdb](https://theme-next.js.org/docs/third-party-services/search-services)
>
>记得把名字和邮箱地址换了
> 
>`COPY` 的目标路径记得写绝对路径，不然你会看到一个名叫 `~` 的文件夹
>
>关于 `CMD` 和 `ENTRYPOINT`
>- [ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint)
>- [Shell and exec form](https://docs.docker.com/reference/dockerfile/#shell-and-exec-form)
>- [init](https://docs.docker.com/reference/compose-file/services/#init)
>
>顺带一提，页面生成与生成后的上传可以放到Github Actions上完成
>
>那样就不需要 `hexo-deployer-git` 和 `git` 了
>
>详细看左边的Github Actions那栏

>`~/website/blog/compose.yaml`
```yaml
services:
  blog:
    container_name: blog
    environment:
      # - VIRTUAL_HOST=blog.jinenze.xyz
      - TZ=Asia/Shanghai
    # expose:
    #   - 4000
    build: .
    # networks:
    #   - nginx
    ports:
      - 4000:4000
    volumes:
      - ./data/_config.yml:/blog/_config.yml
      - ./data/_config.next.yml:/blog/_config.next.yml
      - ./data/source:/blog/source
```
>注释掉的是 [nginx-proxy](https://hub.docker.com/r/nginxproxy/nginx-proxy) 配置，一个自动写nginx配置的项目
>
>默认端口是4000，可以通过在compose文件末尾加上 `command: ["-p","端口"]` 来更改
>
>如果映射出来的都是空文件记得先注释掉 `volumes` 然后 `docker cp blog:/blog ./data` 一下

Hexo
---
设置仓库地址
>`~/website/blog/data/_config.yml`
```yaml
deploy:
  type: git
  repo: git@github.com:YourName/YourRepository.git
  branch: main
```
>一般在文件末尾
>
>`repo` 换成你仓库ssh地址

>`~/website/blog/deploy.sh`
```bash
docker exec blog sh -c "
  hexo clean \
  && hexo g -d
"
```
>`hexo clean` 会删除 `public` 文件夹和 `db.json`

Github
---
新建仓库，名称随意
```bash
cd ~/website/blog
cat key_blog.pub
```
`Settings > Deploy keys > Add deploy key` 把 `key_blog.pub` 的内容添加到deploy key里

`Settings > Pages` 改成 `Deploy from a branch`

Configuration
---
```bash
cd ~/website/blog
docker compose up
```
就可以启动容器了

需要注意alpine镜像默认没有bash，进入容器调试要用sh
```bash
docker exec -it blog sh
```
到这里就可以开始写配置文件和文章了

文章放到 `source/_posts` 文件夹里

配置文件就是 `volumes` 映射出来的那些

下面是提醒

`source` 文件夹下的文件夹会出现在网站根目录

在`source` 文件夹里放一个 `CNAME` 文件

在里面写上你的域名，最后把域名DNS解析到你的github域名就可以用自定义域名访问你的博客了
>不知道你的github域名的话可以在放上文件后在 `Settings > Pages` 里等他验证一下DNS
>
>没验证到就会提醒你改DNS解析记录

可以在 `source/images` 下面存图片，用 `/images/name` 调用

记得注意图片大小，Github Pages还是有限制流量的

`_posts` 目录是存放文章的默认路径

`_posts` 目录下的所有md文件开头必须拥有一个 [Front Matter](https://hexo.io/docs/front-matter) ，不然不会显示在网站里
```
---
title: Hello World
---
```
>Next有对Front Matter进行扩展，点 [这里](https://theme-next.js.org/docs/advanced-settings/front-matter)

下面的是个标签
```
<!-- more -->
```
如果用户在首页，在这个标签下面的东西会被隐藏并显示一个阅读全文的按钮

得记得这玩意上面一行不要写字，不然那行字和底下的按钮会变大

`_posts` 目录下的md文件更改时会直接同步到测试服务器上

`_config` 开头的文件不会，更改的时候直接 `docker compose restart blog`

`_config` 开头的文件看文档编写
- [Hexo](https://hexo.io/docs/configuration) `_config.yml`
- [Next](https://theme-next.js.org/docs/theme-settings/) `_config.next.yml`
- Next [Local Search](https://theme-next.js.org/docs/third-party-services/search-services)

>官方文档里对 `language` 项的描述。。。只能说写了，看 [Next的文档](https://theme-next.js.org/docs/theme-settings/internationalization)

最好不要把 `_config.yml` 里的 `permalink` 项改成 `:title/`

这样会把所有 `_posts` 里的文章全部丢进根目录

我写的是 `:lang/:year/:title/`

需要在每篇文章的 `Front Matter` 里指定 `lang` ，或者看官方文档 [i18n Path](https://hexo.io/docs/internationalization#Path) 配置默认设置

不然文章会出现在一个叫未定义的路径里

>之后可能会详细写一下怎么配置

>评论和统计数据需要第三方服务，之后再说

记得看别人是怎么配置的，也可以看看官方文档的配置

这个方案不会把网站的源配置挂在Github上，而且不会有历史提交记录

如果你不喜欢可以看下面的配置

Github Actions
---
> `~/website/blog/data/.github/workflows/deploy.yml`
```yaml
name: Deploy Hexo Blog

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-22.04
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
          git clone --branch main --single-branch --depth 1 https://github.com/Jinenze/blog.git tmp-repo
          rm -r source
          cp -r tmp-repo/* .
          rm -r tmp-repo

      - name: Build
        run: hexo generate

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```
这样配置时，当你推送代码到 `main` 分支的时候，会触发这个action

这样配置时就不需要在容器里安装 `git` 了，直接在宿主机把 `data` 文件夹上传到你仓库的 `main` 分支就行

部署时默认部署在 `gh-pages` 分支，记得把Github Page部署的 `branch` 换了
