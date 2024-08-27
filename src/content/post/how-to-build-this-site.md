---
title: "Github Pages+Github Action搭建一个Hugo站点"
publishDate: "19 Aug 2024"
description: "搭建Hugo站点"
tags: ["blog"]
draft: false
---


## 前言

之前使用typecho搭建了一个blog站点，不过没找到什么喜欢的typecho主题，久而久之也就荒废了，现在又重新捡起来，使用github pages + hugo搭建了一个Hugo静态站（本站前架构），想到站点搭建也不会太频繁，所以以这篇文章作为记录

此hugo搭建完后可以使用实现自动部署，最终可以通过git修改文章后进行add、commit、push等git操作，最后通过action自动同步到本站点

搭建使用的环境是macOS sonoma，使用ubuntu等别的类UNIX系统步骤应该也都差不多

## 准备

- 一个github账号
- 安装hugo和git，具体不做阐述
- 一个域名（可选）


## 搭建

### 本地环境搭建



`hugo new site blog`

这会在当前目录下生成一个hugo blog站点，以下称blog目录为根目录

``` hugo
blog
├── archetypes
│   └── default.md
├── assets
├── content
├── data
├── hugo.toml
├── i18n
├── layouts
├── static
└── themes
```

### 主题配置

本文使用的主题是[nostyleplease](https://github.com/g-hanwen/hugo-theme-nostyleplease)的hugo移植版本

```
cd blog

git init

git clone https://github.com/g-hanwen/hugo-theme-nostyleplease themes/nostyleplease

rm -rf themes/nostyleplease/.git
```

clone后初始化仓库，此仓库用于存储站点的配置以及后续拉取写文章，删去主题的.git文件，方便后续做修改

### 根目录私有仓库建立

接下来建根目录用的私密仓库。在 github，右上角新建 repository，Repository name 随意填，设置为 private，创建，然后仓库remote到本地

由于现在github默认分支从master换到了main，所以仓库remote后需切换分支

```
git remote add origin git@github.com:<github-username>/<repository-name>.git

git checkout -b main

git add .

git commit "init"

git push origin main

```


### 生成Token

右上角 Settings 里找到 Developer settings，再点 Personal access tokens，Generate new token 生成新的 token，有效期可选永久生效（永久生效的话，这个 Token 就不要用于其他用途，只用于部署这一个 hugo blog），否则要记得定时更换或称新生成 token。

Select scopes里勾选  repo 全部内容与 workflow。最后点击绿色按钮生成

生成后的token先作保留，后续会使用

### 为根目录仓库创建Secrets

进入刚创建好的根目录私有仓库，进入Setting，找到Secrets，New repository secret，创建名为PERSONAL_TOKEN，内容为刚备份的token



### 建立存放public的仓库

github中再建立一个仓库，此仓库用于存放hugo编译后的public文件，且此仓库命名需为 `<username>.github.io`，关于github pages的命名，可以参考[这篇文章](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#types-of-github-pages-sites)

### Github Action
以上，此时你已经有一个本地的站点，一个私有代码仓库（已经初始化），一个公有仓库（未初始化）


接下来需要在私有代码仓库中进行Action的配置，进入私有仓库。点击上方的Action，set up a workflow yourself，之后会在仓库的根目录下创建`.github/workflows/main.yml`
文件，下方是文件的配置，请将配置复制进入main.yml中，yml 文件的名称与配置第一行的 name 随意填，需注意添加注释的地方

```yml
name: hugo-blog

on:
  push:
    branches:
      - main  # 博客根目录的默认分支，这里是main，有时也是master
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
    steps:
      - uses: actions/checkout@v2
#         with:               # 如果你安装主题时用的是git submodule add
#           submodules: true  # 那么这三行不必注释掉，这一行填写 true
#           fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.133.0'  # 填写你的hugo版本，可用hugo version查看
          extended: true          # 如果你使用的不是extended版本的hugo，将true改为false

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: ${{ github.ref == 'refs/heads/main' }}  # 注意填写main或者master
        with:
          personal_token: ${{ secrets.PERSONAL_TOKEN }} # 如果secret取了其他名称，将PERSONAL_TOKEN替换掉
          external_repository: langchou/langchou.github.io # 填写远程仓库，不一定是这个格式，按照自己的情况写 
          publish_dir: ./public
          cname: blog.jontyding.com        # 填写你的自定义域名。如果没有用自定义域名，注释掉这行

```

此时，提交commit后进行push，你会发现私有仓库自动执行了一个action，同时另一个公有仓库中也会出现一个gh-pages的分支，里面就是hugo编译后的文件，此时去`<username>.github.io`，也就是你的站点，也可以正常访问了


### 域名配置（可选）

如有自己的域名，可以对域名进行配置，以下提供一个子域名的配置教程，顶级域名的配置请参考Github Docs

首先，你需要在你的私有仓库根目录下的`config.toml`中更改`baseURL`为你的自定义域名，如我的则为`http://blog.jontyding.com/`


#### 顶级域名

详细可参考Github Docs中[About custom domain configuration](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#about-custom-domain-configuration)


#### 子域名

如果需要配置`blog.example.com`作为网站的自定义域，需要在DNS服务商中为`blog`子域创建一条`CNAME`记录，并且指向Github Pages的默认域，也就是`<username>.github.io`。同样，如果组织网站位于`<organization>.github.io`，也应该创建一条`CNAME`，确保`blog.example.com`指向`<organization>.github.io`

当这些都配置好后，可以在共有仓库的`Settings`，`Pages`选项下的`Custom domain`下查看DNS Check是否成功

