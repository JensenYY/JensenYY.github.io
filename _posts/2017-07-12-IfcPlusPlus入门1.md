---
layout: post
title:  IfcPlusPlus入门（一）
date:   2017-07-12 15:08:00 +0800
tag: C++
---



### IfcPlusPlus入门（一）
Travis CI地址：[https://travis-ci.org/Supporting/ifcplusplus](https://travis-ci.org/Supporting/ifcplusplus)


  首先在网上下载IfcPlusPlus的代码：git clone https://github.com/Supporting/ifcplusplus.git

  然后执行以下命令进行相应环境的安装：

```bash
before_install:
  sudo add-apt-repository -y ppa:kubuntu-ppa/backports
  sudo add-apt-repository -y ppa:ubuntu-toolchain-r/test
  sudo apt-get update -qq
install:
  sudo apt-get install libboost-dev
  sudo apt-get install -qq gcc-4.8 g++-4.8
  sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 90
  sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 90
  sudo apt-get install cmake"
build the project:
  cd IfcPlusPlus
  mkdir build
  cd build
  cmake ..
  make -j4
```
