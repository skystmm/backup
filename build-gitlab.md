---
title: Gitlab搭建和基本操作
date: 2020-01-14 22:34:20
tags: [git,环境搭建]
categories: other
---
由于近期工作的原因，项目中引入了gitlab作为规则引擎中规则版本进行管理，所以需要搭建gitlab，故记录下搭建过程已备以后查看。

## 1.安装

本文是在centOS7 64环境下进行安装，gitlab安装版本为CE-9.5.9，使用官网下载的rpm包进行安([官网下载链接](https://packages.gitlab.com/gitlab/gitlab-ce))

>注意：gitlab对硬件至少4GB内存

<!-- more -->
### 依赖安装
```
[root@your-server ~]# yum install -y curl policycoreutils-python openssh-server
[root@your-server ~]# systemctl enable sshd
[root@your-server ~]# systemctl start sshd
[root@your-server ~]# firewall-cmd --permanent --add-service=http
[root@your-server ~]# systemctl reload firewalld
##修改/etc/sysconfig/selinux 永久生效
setenforce 0
```

### gitlab安装
```
[root@your-server ~]# rpm -ivh gitlab-ce-9.5.9-ce.0.el7.x86_64.rpm
```

## 2.基础配置修改
配置文件尾 /etc/gitlab/gitlab.rb
```
[root@your-server ~]# vim  /etc/gitlab/gitlab.rb
#进入配置文件后

#对gitlab访问域名进行配置，可以为ip
external_url 'http://<gitlab访问的域名>:10000'
#修改nginx的端口，默认为80，这里修改后external_url需要进行同步修改
nginx['listen_port'] = 10000
```
完成修改后需要执行
```
[root@your-server ~]#  gitlab-ctl reconfigure
```
使修改生效

### gitlab常用命令

>p.s.启停等命令需要在root权限下执行，否则会出现access deny错误提示，导致无法启动问题
```
gitlab-ctl start #启动全部服务
gitlab-ctl restart #重启全部服务
gitlab-ctl stop #停止全部服务
gitlab-ctl restart nginx #重启单个服务
gitlab-ctl status #查看全部组件的状态
gitlab-ctl show-config #验证配置文件
gitlab-ctl uninstall #删除gitlab(保留数据）
gitlab-ctl cleanse #删除所有数据，重新开始
gitlab-ctl tail <svc_name>  #查看服务的日志
gitlab-rails console production #进入控制台 ，可以修改root 的密码
```

## 3.root密码修改
gitlab安装完毕后，使用浏览器第一次访问gitlab时，可以对root账号进行密码修改

对root账号密码进行修改，还可以使用命令行进行，具体命令如下

### 1）在系统root权限下执行
```
[root@your-server ~]#  gitlab-rails console production
```
### 2）进入rails console后进行如下操作

```
irb(main):001:0> user = User.where(id: 1).first
=> #<User id:1 @root>

irb(main):002:0> user.password="12345678"
=> "12345678"

irb(main):003:0> user.password_confirmation="12345678"
=> "12345678"

irb(main):004:0> user.save!
=> true

irb(main):005:0> quit
```
即可对gitlab的root账户进行密码修改



## 3.gitlab数据备份和恢复


### 1）对gitlab进行定时备份任务配置
```
[root@your-server ~]# vim  /etc/gitlab/gitlab.rb
#进入配置文件后

gitlab_rails['manage_backup_path'] = true
gitlab_rails['backup_path'] = "/data/gitlab/backups"    //gitlab备份目录，不配置时，默认为/var/opt/gitlab/backup目录
gitlab_rails['backup_archive_permissions'] = 0644       //生成的备份文件权限
gitlab_rails['backup_keep_time'] = 7776000              //备份保留天数为3个月（即90天，这里是7776000秒）

#如果修改了backup_path，需要确保配置的路径存在和权限正确，进行如下配置
[root@your-server ~]# mkdir -p /data/gitlab/backups
[root@your-server ~]# chown -R git.git /data/gitlab/backups
[root@your-server ~]# chmod -R 777 /data/gitlab/backups

#使配置修改生效
[root@your-server ~]# gitlab-ctl reconfigure
```
### 2）手动备份gitlab
```
[root@code-server backups]# gitlab-rake gitlab:backup:create
```
执行完成后，会在gitlab的backup_path下生成备份的tar文件，这里也可以利用linux的crontab来配置定时任务进行备份。



### 3）gitlab恢复
首先关闭gitlab数据相关服务
```
[root@your-server backups]# gitlab-ctl stop unicorn
[root@your-server backups]# gitlab-ctl stop sidekiq

#查看gitlab运行状态，确保unicorn，sidekiq两个服务已经停止
[root@your-server backups]# gitlab-ctl status
 ```

然后将备份文件拷贝到gitlab配置的backup_path路径下（默认为/var/opt/gitlab/backups目录）

然后执行gitlab恢复命令
```
#例如导出的备份文件为1578453712_2020_01_08_9.5.9_gitlab_backup.tar，则备份信息版本号为1578453712_2020_01_08_9.5.9
gitlab-rake gitlab:backup:restore BACKUP=<备份信息版本号>
根据恢复提示信息进行yes操作，即可完成gitlab的恢复工作
```


最后重新启动gitlab即可完成恢复工作
```
[root@your-server backups]# gitlab-ctl start
```
