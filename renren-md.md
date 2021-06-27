---
title: 人人网好友关系爬取
date: 2019-03-11 22:07:36
tags: [python,爬虫]
categories: python
---


## 前言

一开始准备爬取人人网的数据，是因为想把自己的人人数据做一个备份，结果后来就跑偏了，成了爬取好友关系网。（p.s.再一次成功跑偏）。作为拖延症重度患者，本来这个爬虫是在人人网被转卖的时候写好的，但是仅仅是把数据爬了回来，让数据安静的躺在家里的树莓派里。

<!-- more -->

## 准备

要爬去一个网站的数据，首先是要对这个网站及这个网站中想要获取数据的数据接口进行分析。不同于上次大麦网（资讯类网站）数据的获取，这里解决需要用户的登录以及session保存的问题。
人人网作为曾经风靡一时的社交类网站，对于登录认证还是做了一些加固的，不像有些网站明文密码传输。而是对于密码信息进行了加密处理。正好前一段时间尝试爬取一些漫画类网站的时候，有看到使用selenium+phantomjs模拟浏览行为的方式，获取动态加载的漫画图片。也是出于偷懒的心思，就决定试试用selenium+phantomjs来绕过人人网的登录。

## 尝试

决定了使用seleium+phantomjs来作为这次爬虫使用的技术框架，就开始着手准备环境的搭建，总的来说环境搭建比较简单。

1.seleium安装
> pip install seleium

2.phantomjs下载

可以从[phantonjs官网](http://phantomjs.org/download.html)选择适合的操作系统下载最新版的程序，同时官网也提供源码下载。

完成了基础环境的准备，接下来就可以开始进行数据爬取了。

```python
from selenium import webdriver

driver = webdriver.PhantomJS('/Users/tian/Downloads/phantomjs-2.1.1-macosx/bin/phantomjs')
driver.set_window_size("800","600")
driver.get('http://renren.com')

print("---------------***************---------------------")
input1 = driver.find_element("id", "email")
input1.send_keys("****")
input2 = driver.find_element("id", "password")
input2.send_keys("*****")

driver.find_element("id", "login").click()
```
使用seleium+phantomjs进行人人网的模拟登陆的代码如上，通过seleium的webdriver.PhantomJS拉起phantomJS无界面浏览器，之后的操作类似于JQuery的选择器，找到需要使用的元素，然后设置相关的参数。

虽然很容易就可以绕过人人网的登录操作，但是并不是没有问题，phantomjs在并发情况下，表现不是很令人满意。最后还是放弃了使用seleium+phantomjs的方式进行数据的爬去。

## 分析和设计

由于seleium+phantomjs表现不能满足需求，不能使用偷懒的方式进行数据获取，只能按部就班的对登陆页面进行分析。最后在github上找到了前人的经验([github链接](https://github.com/XueSeason/renren-album)),作者使用js实现了人人网登陆的密码加密方法。于是发挥python“胶水语言”的特性以及强大的类库支持，也本着不重复造轮子的原则就厚颜无耻的拿过来使用了。

既然已经解决了登陆密码加密的问题，就可以进行下一步关系网的爬取了。在社交网络中有个六度理论，所以一开始想要从自己的人人账号出发，爬取六度好友，就可以获取所有人人网用户的关系网络。但是这个过程应该是一个先发散后收敛的过程.人人网有上亿用户，所以单任务爬取不是一个可行的方案。需要进行分布式爬取，而采用现有的分布式爬取框架，又觉得有点重，于是设计了如下架构进行这次爬取。

![爬虫架构设计](https://upload-images.jianshu.io/upload_images/7504708-0b3669cd964af72f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里redis有两个作用，一个是经常我们使用的共享缓存的作用，还有一个是借用redis中队列数据结构，将任务解耦供多个worker进行分布式爬取好友关系数据。同时为了避难重复爬去而导致的任务无法结束问题，借助redis的bitmap进行过滤已经获取过用户信息，从而避免无限爬取的情况。

login/worker是一个特殊的worker,该worker负责登录并将session信息缓存到redis中，供其他worker使用，然后获取数据源的好友信息，并将其放入redis队列中，之后和其他worker一样消费redis队列中的任务信息，进行分布式爬取。

worker的设计，为了进一步提升效率，使用线程池+线程的方式进行数据的爬取。由于树莓派的内存限制，导致最后只爬取了自己账号的三度好友的数据。

> p.s.爬虫代码上传到了github中（[传送门](https://github.com/skystmm/some-spider)）

## 总结

本次爬虫的编写，一开始走了很多弯路，也有了一些思维定式的原因。一开始主观臆断的选择了seleium+phantomjs方式进行，并没有考虑导数据量的问题和爬取效率的问题。同时也反映出初期准备的时候没有对目标网站连接仔细分析，导致选型上的偏差。从而做了大量的无用功，之后是需要注意的。
