---
title: python环境搭建
date: 2017-12-06 22:38:02
tags: [python,树莓派,环境搭建]
categories: python
---

# Python 安装笔记

## 1.下载源码包

>wget https://www.python.org/ftp/python/3.6.3/Python-3.6.3.tgz

如果没有wget则可通过

>yum install wget

安装wget

<!-- more -->

## 2准备工作

检查当前linux系统中的ssl相关.so是否齐全：

>rpm -aq|grep openssl

齐全情况应该如下：

```
openssl-devel-1.0.2k-8.el7.armv7hl
openssl-libs-1.0.2k-8.el7.armv7hl
```

如果缺少则可通过

> yum install openssl openssl-devel

进行补全安装

## 3.编译安装

> ./configure --prefix=/usr/local/python3

> make && make install

就完成了基本的python源码安装，当下版本的python源码安装已经将pip集成在源码中，不需要在单独安装pip，就可以直接使用pip进行模块安装了

这里没有进行环境变量的配置而是直接在/usr/bin下建立了python3和pip的软连接，命令如下：

> ln -s /usr/local/python3/bin/python /usr/bin/python3
> ln -s /usr/local/python3/bin/pip3 /usr/bin/pip
