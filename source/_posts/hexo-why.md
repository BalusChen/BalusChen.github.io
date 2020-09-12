---
title: hexo-why
date: 2020-09-12 12:06:45
tags:
    - hexo
---

hexo 搭建博客折腾过很多次，但是其中一些疑问点一直困扰着我，而且其中也遇到了很多的坑，所以把它们记录下来。

## GitHub 仓库

- 为什么 GitHub 仓库名一定要是 ${username}.github.io

一开始按照[这篇博客](https://juejin.im/post/6844904070533087245)提供的方法来使用 hexo 搭建博客，其中直接把仓库命名为 myblog，然后使 travis.ci 进行自动化部署 github-pages，travis 配置：

```yaml
sudo: false
language: node_js
node_js:
  - 12
cache: npm
branches:
  only:
    - master
script:
  - hexo generate
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GITHUB_TOKEN
  keep-history: true
  on:
    branch: master
  local-dir: public

```

大概的意思就是说使用 node 12 版本（本机其实已经 14.9 了），然后只在 master 分支更新时触发部署，触发时执行`hexo generate`这个命令，部署的类型为 github-pages；它新建一个 gh-pages 的分支，用来存储生成的数据。

然后在 _config.yml 配置文件中将 root 改为 /myblog/，这样就可以直接从 ${username}.github.io/myblog 访问（如果将仓库命名为`${username}.github.io`则不用修改 root，并且直接可以通过 ${username}.github.io 访问）。

但是后面在关联域名的时候就没法用了，在 DNSpod 上配置时需要配置：

![dsnpod](images/inpost/dnspod.png)

其中的记录值只能是一个简单的域名，不能带有 path，他表示当访问 baluschen.github.io 时，会跳转至 laputa.world，但是我没有配置域名之前访问 baluschen.github.io 就是 404，所以后面还是把它乖乖改回标准格式了。

## 主题的使用

主题放在 themes/ 目录下面，我一开始是直接 clone 进去的（因为想保持更新），但是在使用 travis 部署的时候却总是提示我 no layout，部署过后所有页面都是空的。而且在本地测试的时候，也会报错说有多个 git 仓库（博客本身一个，主体又又一个），所以我打算使用 git submodule 的方式来引用主题，而且其实 travis 本身也是会递归 clone 的：

```sh
git clone --depth=50 --branch=master https://github.com/BalusChen/laputa.git BalusChen/laputa
git submodule update --init --recursive
```

开始的 .gitmodules 文件是这样的：

```git
[submodule "themes/icarus"]
	path = themes/icarus
	url = git@github.com:ppoffice/hexo-theme-icarus.git
```

结果 travis 在拉取 theme repo 的时候报错：

```tet
Please make sure you have the correct access rights
and the repository exists.
fatal: clone of 'git@github.com:ppoffice/hexo-theme-icarus.git' into submodule path '/home/travis/build/BalusChen/laputa/themes/icarus' failed
Failed to clone 'themes/icarus'. Retry scheduled
Cloning into '/home/travis/build/BalusChen/laputa/themes/icarus'...
Warning: Permanently added the RSA host key for IP address '140.82.112.3' to the list of known hosts.
Permission denied (publickey).
fatal: Could not read from remote repository.
```

后面参考[这个](https://docs.travis-ci.com/user/common-build-problems/#git-cannot-clone-my-submodules)把 git 协议改成 https 协议就 OK 了：

```git
[submodule "themes/icarus"]
	path = themes/icarus
	url = https://github.com/ppoffice/hexo-theme-icarus.git
```



## 关联域名

域名是在[namesilo](https://www.namesilo.com/)上买的，域名的关联比较简单，主要是域名解析的工作，我使用的是[DNSPod](dnspod.cn)来解析，主要工作由：

- 在博客中新增 CNAME 文件指定自己的域名
- 在 DNSPod 中新增 A 记录和 CNAME 记录
- 在 namesilo 替换域名服务器为 DNSPod 提供的 name server

## 评论系统

评论系统使用[gittalk](https://github.com/gitalk/gitalk/blob/master/readme-cn.md)搭建，首先需要在[GitHub](https://github.com/settings/applications/new) 中注册一个新的 GitHub Application，其中有两项信息很重要：

- Homepage URL：博客首页地址
- Authorization callback URL：gittalk 需要 GitHub 登陆才可以评论，这里经过 GitHub 授权后跳转回去的地址，一般也是博客首页地址

这两个地址需要填入 _config 中，如果二者和博客地址不一样（比如 http 和 https），那么可能会出现点击 GitHub 登陆后直接就跳回主页的情况。

## 文章插入图片

在 _config.yml 中指定 logo 等图标时，都是像`/img/logo.png`这样指定 img 目录下面的文件，我搜索了下这是 public 目录下面的一个子目录，但是 public 目录在 gitignore 的，即不会上传至 github。可以看看`hexo generate`后生成的数据：

![gh-pages](images/inpost/gh-pages.png)

可以看到出 source 目录下除下划线开头的文件外其它文件都被放在了根目录，所以我们只需要在 source 目录下新建一个 images 目录就可以在 Markdown 文件中直接以`/images/xxx.png`的方式引用了。

> 至于`hexo generate`命令生成的数据为什么是这样的，还得 RTFM




## 参考

[Icarus用户指南 - 主题配置](https://blog.zhangruipeng.me/hexo-theme-icarus/Configuration/icarus%E7%94%A8%E6%88%B7%E6%8C%87%E5%8D%97-%E4%B8%BB%E9%A2%98%E9%85%8D%E7%BD%AE/)

[2020年，必须拥有自己的博客网站(上)](https://juejin.im/post/6844904070533087245)

[用Hexo + Github Pages搭建你的个人博客（2020版）](https://blog.yaozhihao.com/2020/03/09/2020-03-09-%E7%94%A8Hexo-Github-Pages%E6%90%AD%E5%BB%BA%E4%BD%A0%E7%9A%84%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2%EF%BC%882020%E7%89%88%EF%BC%89/)

[Hexo博客搭建之在文章中插入图片](https://yanyinhong.github.io/2017/05/02/How-to-insert-image-in-hexo-post/)

[Hexo基础教程(二)：个人域名绑定](https://indexmoon.com/articles/1318032967/)





