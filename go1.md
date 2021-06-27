---
title: go学习笔记(一)——环境搭建
date: 2020-02-07 22:34:17
tags: [go,环境搭建]
categories: go
---

![Go语言.jpg](http://image.stxl117.top/Go语言.jpg)
<!-- more -->
# go安装和配置
[国内源下载](https://studygolang.com/dl)

windows环境下，下一步安装法，即可完成安装


>[go中文文档](https://go-zh.org/doc/)
  [go包信息检索](https://golang.google.cn/pkg/)



# go的常用命令

go包含的命令 如下图：
![go命令.png](http://image.stxl117.top/7504708-46eb42f7e559ecec.png)

 ## 常用命令
| 命令  | 可选参数| 作用  | 备注 |
| ------ | ------ | ------ |------ |
| go build  | -o 指定输出名称 |将go代码文件编译成可执行文件  | 默认输出名称为package名称，使用build时，会忽略已"_"和"."开头的文件 |
| go install |   | ||
| go clean |  | 清理当前项目中的 | |
| go get | | 从远程仓库获取代码包| 同python的pip |
| go fmt | -w 写入到源文件 |代码标准格式化 | |
| go test | | 执行单元测试(文件名为*_test.go)| |

# IDE选择

## 1.atom
[在atom上开发go](https://blog.csdn.net/sweetvvck/article/details/50333327)

## 2. vs code
安装请至[官网](https://code.visualstudio.com/Download)选择对应版本进行安装

> [vscode实用快捷键](https://www.jianshu.com/p/3171be60b736)

### 手动安装go需要的插件
> [vs code go插件安装失败问题解决](https://blog.csdn.net/dmt742055597/article/details/85865186)

#### 创建github.com下的插件
```bash
cd $GOPATH/src
mkdir github.com
cd $GOPATH/src/github.com
mkdir acroca cweill derekparker go-delve josharian karrick mdempsky pkg ramya-rao-a rogpeppe sqs uudashr
cd $GOPATH/src/github.com/acroca
git clone https://github.com/acroca/go-symbols.git
cd $GOPATH/src/github.com/cweill
git clone https://github.com/cweill/gotests.git
cd $GOPATH/src/github.com/derekparker
git clone https://github.com/derekparker/delve.git
cd $GOPATH/src/github.com/go-delve
git clone https://github.com/go-delve\delve.git
cd $GOPATH/src/github.com/josharian
git clone https://github.com/josharian/impl.git
cd $GOPATH/src/github.com/karrick
git clone https://github.com/karrick/godirwalk.git
cd $GOPATH/src/github.com/mdempsky
git clone https://github.com/mdempsky/gocode.git
cd $GOPATH/src/github.com/pkg
git clone https://github.com/pkg/errors.git
cd $GOPATH/src/github.com/ramya-rao-a
git clone https://github.com/ramya-rao-a/go-outline.git
cd $GOPATH/src/github.com/rogpeppe
git clone https://github.com/rogpeppe/godef.git
cd $GOPATH/src/github.com/sqs
git clone https://github.com/sqs/goreturns.git
cd $GOPATH/src/github.com/uudashr
git clone https://github.com/uudashr/gopkgs.git
```

#### 创建golang.org下的插件
```bash
cd $GOPATH/src
mkdir -p golang.org/x
cd golang.org/x
git clone https://github.com/golang/tools.git
git clone https://github.com/golang/lint.git
```
#### 手动安装插件
```bash
cd $GOPATH/src
go install github.com/mdempsky/gocode
go install github.com/uudashr/gopkgs/cmd/gopkgs
go install github.com/ramya-rao-a/go-outline
go install github.com/acroca/go-symbols
go install github.com/rogpeppe/godef
go install github.com/sqs/goreturns
go install github.com/derekparker/delve/cmd/dlv
go install github.com/cweill/gotests
go install github.com/josharian/impl
go install golang.org/x/tools/cmd/guru
go install golang.org/x/tools/cmd/gorename
go install golang.org/x/lint/golint


```
