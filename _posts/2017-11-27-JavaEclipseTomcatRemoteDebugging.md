---
layout: post
title:  "Eclipse远程调试tomcat中的java程序"
date:   2017-11-27 19:25:01 +0800
tag: Java
---

### 1、首先要配置服务器上的tomcat
#### Windows:
>tomcat在启动的时候会查看/bin目录下是否存在setenv.bat的文件，如果存在则加载其中的配置信息,所以在/bin目录下创建setenv.bat文件，然后输入以下配置信息：
```
set CATALINA_OPTS=-Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=n
```
>此时提供给远程调试访问的端口号是8000，然后重新启动tomcat，然后使用cmd查看8000端口是否打开：
```
netstat -an
```

#### Linux:
>linux服务器和windows类似，在tomcat下的/bin目录下创建文件setenv.sh，添加
```
CATALINA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=n"
```
>然后重新启动tomcat，并查看指定的端口是否被占用。
```
netstat –apn
```
### 2、然后配置eclipse
>打开调试选项：
![pic1]({{'/styles/images/eclipse1.jpg'}})
>然后进行相应的配置：
![pic2]({{'/styles/images/eclipse2.jpg'}})
>最后点击debug开始调试。