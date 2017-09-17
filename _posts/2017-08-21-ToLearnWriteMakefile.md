---
layout: post
title:  "如何写Makefile(一)"
---


##### 相关知识

* 链接（link）：无论是c还是c++，首先要把源文件编译成中间代码文件，unix下是.o文件，即ObjectFile,这个动作叫做编译（compile）。然后再把大量的ObjectFile合成执行文件，这个动作叫做链接（link）。

* 基本原则：
1、如果这个工程没有编译过，那么我们的所有的c文件都要编译并被链接。
2、如果这个工程的某几个c文件被修改，那么我们只编译被修改的c文件，并链接目标程序。
3、如果这个工程的头文件被改变了，那么我们需要编译引用了这几个文件的c文件，并链接目标程序。

##### makefile 规则

```Makefile
target ... :prerequisites ...
  command
  ...
  ...
```
* target : 可以是一个object file，也可以是一个执行文件，还可以是一个标签（label）。
* prerequisites : 生成该target所依赖的文件和\\或target。
* command : 该target要执行的命令。

example:
```Makefile
edit : main.o kbd.o \
    cc -o edit main.o kbd.o

main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h
    cc -c kbd.c
clean :
    rm *.o
```
