---
layout: post
title:  "JavaServlet运行机制"
date:   2017-09-14 18:25:01 +0800
categories: Java
tag: Java
---

* content
{:toc}



### 参考：
* [Servlet 工作原理解析](https://www.ibm.com/developerworks/cn/java/j-lo-servlet/)
* [Servlet运行机制与生命周期](http://blog.csdn.net/suwu150/article/details/51487398)


当浏览器发送给服务器一个Servlet的请求时，如果这个Servlet是第一次被调用，那么服务器将会自动创建一个Servlet实例，并运行它；而如果这个Servlet已经被实例化，那么服务器只是会新启动一个线程来运行它。所以，多个线程有可能会去访问共享的全局变量，因此，在使用这些全局变量时，一定要特别小心，让这些线程不会访问到不同步的数据。除非是需要共享的信息。

下面我们使用例子进行演示Servlet的运行过程，说明只创建一个实例，而进行多次线程调用，首先给这个Servlet增建一个构造函数，并在doGet()函数中也打印一个标记，重新部署，看看界面的显示，代码如下所示： 
```java
package servlet;  
  
import java.io.IOException;  
import java.io.PrintWriter;  
import javax.servlet.ServletException;  
import javax.servlet.http.HttpServlet;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
  
public class ShoeServlet2 extends HttpServlet {  
  
    public ShoeServlet2() {  
        System.out.println("ShoeServlet2:构造函数");  
    }  
  
    protected void processRequest(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
        response.setContentType("text/html;charset=UTF-8");  
        try (PrintWriter out = response.getWriter()) {  
            System.out.println("ShoeServlet2:processRequest");  
        }  
    }  
  
    @Override  
    protected void doGet(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
        System.out.println("ShoeServlet2:doGet");  
        processRequest(request, response);  
    }  
  
    protected void doPost(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
        System.out.println("ShoeServlet2:doPost");  
        processRequest(request, response);  
    }  
  
    @Override  
    public String getServletInfo() {  
        System.out.println("ShoeServlet2:getServletInfo");  
        return "Short description";  
    }// </editor-fold>  
  
}  
```

>运行后在控制台出现如下结果：
第一次进入的显示：
![servlet1]({{'/styles/images/servlet1.png'}})
![servlet2]({{'/styles/images/servlet2.png'}})

从上面显示效果可以看出：Servlet第一次被调用，那么服务器将会自动创建一个Servlet实例，并运行它；而如果这个Servlet已经被实例化，那么服务器只是会新启动一个线程来运行它.

