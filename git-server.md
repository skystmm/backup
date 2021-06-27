---
title: 腾讯云搭建hexo博客采坑记录
date: 2018-11-11 17:48:09
comments: true
tags: [git,hexo]
categories: other
---

## git环境搭建

linux下的git环境搭建可以参考：[传送门](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137583770360579bc4b458f044ce7afed3df579123eca000)

<!-- more -->

## ssh免密登录注意事项

### 1.修改sshd读取免密公钥路径
如果使用git在的.ssh目录，需要在root用户下
对shhd_config进行修改，

> vim /etc/ssh/sshd_config

在文件内找到AuthorizedKeysFile配置，配置为自己配置的authorized_keys文件路径

### 2. .ssh路径的权限问题
需要对.ssh路径权限设置为700，authorized_keys设置为600

>chmod 700 .ssh
>chmod 600 authorized_keys

## git初始化以及提交部署问题

### 1. 初始化空仓库远程提交失败问题
在仓库路径下使用命令，命令行执行
> git config receive.denyCurrentBranch ignore

### 2.提交到仓库后自动部署
在仓库.git目录下的hooks目录下创建post-recieve文件，来设定提交后部署操作，脚本可参考下面的脚本内容’
```shell
#!/bin/sh
BASE_HOME=<发布路径>
git --work-tree=${BASE_HOME} --git-dir=/home/git/blog.git checkout -f
```

最后在hexo的根目录下的_config.yml中添加新建仓库即可。
