---
layout: post
title:  "Golang读写文件"
date:   2017-12-04 12:50:01 +0800
tag: Golang
---

工作的时候碰到了点小工作就是需要遍历一个文件夹，然后把该文件夹下包含的多个json文件合并为一个，由于以前没有接触过go的文件处理。上网上参考了一些博客和文档，写了一个简单的读写文件的小程序，代码如下：

```golang
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

func writeDataToTargetFile(fileName string) error {
	file, err := os.Open(fileName)
	defer file.Close()
	if nil != err {
		return err
	}
	//find this file's type
	splt := strings.Split(fileName, "\\")

	path := splt[len(splt)-1]
	fmt.Println(path)
	//打开目标文件，并向文件末尾追加信息。
	targetFile, err := os.OpenFile("./"+path, os.O_WRONLY|os.O_APPEND, 0666)
	if err != nil {
		targetFile, err = os.Create("./" + path)
		if nil != err {
			return err
		}
		fmt.Println("Create " + path + " success!")
	}
	defer targetFile.Close()

	buf := make([]byte, 1024)
	for {
		//带缓冲的读取文件
		n, _ := file.Read(buf)
		if 0 == n {
			break
		}
		//将有效的bufer写入到目标文件
		targetFile.Write(buf[0:n])
	}
	return nil
}

func walkFunc(path string, info os.FileInfo, err error) error {
	if info == nil {
		// 文件名称超过限定长度等其他问题也会导致info == nil
		// 如果此时return err 就会显示找不到路径，并停止查找。
		println("can't find:(" + path + ")")
		return nil
	}
	if info.IsDir() {
		println("This is folder:(" + path + ")")
		return nil
	} else {
		//read the file and put the data into targe file
		println("This is file:(" + path + ")")
		writeDataToTargetFile(path)
		return nil
	}
}

func showFileList(root string) {
	err := filepath.Walk(root, walkFunc)
	if err != nil {
		fmt.Printf("filepath.Walk() error: %v\n", err)
	}
	return
}

func main() {
	var path string
	fmt.Println("请输入想要遍历的路径：")
	fmt.Scanf("%s", &path)
	/*获取当前绝对路径
	pwd, err := os.Getwd()
	if nil != err {
		fmt.Println(err.Error())
		return
	}
	fmt.Println("绝对路径" + pwd)
	*/
	fmt.Println("===================================")
	showFileList(path)
}
```
