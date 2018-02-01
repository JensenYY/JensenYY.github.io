---
layout: post
title:  "NSQ的安装和使用"
date:   2018-02-01 12:50:01 +0800
tag: NSQ
---

## NSQ的安装

针对不同的系统下载NSQ的安装包，下载地址http://nsq.io/deployment/installing.html

这里我是用的是Ubuntu所以下载的对应版本```nsq-1.0.0-compat.linux-amd64.go1.8.tar.gz```
解压程序包```tar -zxvf nsq-1.0.0-compat.linux-amd64.go1.8.tar.gz```
然后将解压出的文件夹下的```bin/```目录下的程序全部拷贝到```/bin/```下即可。

## 验证NSQ

### 首先启动nsqlookup
>在终端输入:```nsqlookup```,然后显示如下界面，他会打印出监听的信息，如TCP监听本地的4160端口、HTTP监听本地的4161端口。
>![nsqlookup]({{'/styles/images/nsqlookup.png'}})

### 然后启动nsq
>在终端输入：```nsqd --lookupd-tcp-address=127.0.0.1:4160```,nsqd [参数] IP:PORT。启动后会显示nsqd所监听的信息。
>![nsqd]({{'/styles/images/nsqd.png'}})

### 启动nsqadmin
>在终端中输入:``` nsqadmin --lookupd-http-address 127.0.0.1:4161```,nsqadmin [选择监听lookup还是nsqd] IP:PORT。启动后会显示nsqadmin所监听的信息。
>![nsqadmin]({{'/styles/images/nsqadmin.png'}})
