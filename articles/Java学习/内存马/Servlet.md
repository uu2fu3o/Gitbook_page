> 本篇旨在从0开始进入javaweb章节，记录一下学习的过程

在进入Servelet之前，我们首先要先学会如何构建一个web项目

## 构建web项目

打开idea，新建项目中选择

![image-20240618214345253](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240618214345253.png)

可自行添加tomcat等应用程序的服务器，这里入门暂时可以不用考虑

## Servlet入门

要实现一个http服务，我们通常需要处理较多的事件，工作量过大

JavaEE提供了Servlet API，我们使用Servlet API编写自己的Servlet来处理HTTP请求，Web服务器实现Servlet API接口，实现底层功能：

```ascii
                 ┌───────────┐
                 │My Servlet │
                 ├───────────┤
                 │Servlet API│
┌───────┐  HTTP  ├───────────┤
│Browser│<──────>│Web Server │
└───────┘        └───────────┘
```

```java
package org.example.java_web;

import java.io.*;
import jakarta.servlet.http.*;
import jakarta.servlet.annotation.*;

//定义访问地址
@WebServlet(name = "helloServlet", value = "/hello-servlet")
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        doPost(request, response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws IOException {
        PrintWriter out = response.getWriter();
        out.println("Hello World~");
        out.flush();
        out.close();
    }
}
```

一个Servlet总是继承自`HttpServlet`，然后覆写`doGet()`或`doPost()`方法。注意到`doGet()`方法传入了`HttpServletRequest`和`HttpServletResponse`两个对象，分别代表HTTP请求和响应。我们使用Servlet API时，并不直接与底层TCP交互，也不需要解析HTTP协议，因为`HttpServletRequest`和`HttpServletResponse`就已经封装好了请求和响应。以发送响应为例，我们只需要设置正确的响应类型，然后获取`PrintWriter`，写入响应即可。

> 这里需要注意你的tomcat版本和servlert依赖版本问题，要兼容才能调用

![image-20240619151458453](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240619151458453.png)

从代码来看，我们这里是基于注解的方式进行访问，还有基于web.xml的方式进行访问，比较繁琐，需要手动添加定义，这里不做介绍

### Request & Response

在`B/S架构`中最重要的就是浏览器和服务器端交互，`Java EE`将其封装为`请求`和`响应对象`，即 `request(HttpServletRequest)` 和 `response(HttpServletResponse)`，一个负责处理请求，一个负责响应请求

HttpServletRequest常用方法

| 方法                            | 说明                                       |
| ------------------------------- | ------------------------------------------ |
| getParameter(String name)       | 获取请求中的参数，该参数是由name指定的     |
| getParameterValues(String name) | 返回请求中的参数值，该参数值是由name指定的 |
| getRealPath(String path)        | 获取Web资源目录                            |
| getAttribute(String name)       | 返回name指定的属性值                       |
| getAttributeNames()             | 返回当前请求的所有属性的名字集合           |
| getCookies()                    | 返回客户端发送的Cookie                     |
| getSession()                    | 获取session回话对象                        |
| getInputStream()                | 获取请求主题的输入流                       |
| getReader()                     | 获取请求主体的数据流                       |
| getMethod()                     | 获取发送请求的方式，如GET、POST            |
| getParameterNames()             | 获取请求中所有参数的名称                   |
| getRemoteAddr()                 | 获取客户端的IP地址                         |
| getRemoteHost()                 | 获取客户端名称                             |
| getServerPath()                 | 获取请求的文件的路径                       |

HttpServletResponse常用方法

| 方法                                 | 说明                                 |
| ------------------------------------ | ------------------------------------ |
| getWriter()                          | 获取响应打印流对象                   |
| getOutputStream()                    | 获取响应流对象                       |
| addCookie(Cookie cookie)             | 将指定的Cookie加入到当前的响应中     |
| addHeader(String name,String value)  | 将指定的名字和值加入到响应的头信息中 |
| sendError(int sc)                    | 使用指定状态码发送一个错误到客户端   |
| sendRedirect(String location)        | 发送一个临时的响应到客户端           |
| setDateHeader(String name,long date) | 将给出的名字和日期设置响应的头部     |
| setHeader(String name,String value)  | 将给出的名字和值设置响应的头部       |
| setStatus(int sc)                    | 给当前响应设置状态码                 |
| setContentType(String ContentType)   | 设置响应的MIME类型                   |

## [Servlet进阶](https://www.liaoxuefeng.com/wiki/1252599548343744/1328761739935778)

这部分主要是开发的知识，可以看一下

### [Cookie和Session](https://www.javasec.org/javaweb/Cookie&Session/)

`Cookie` 是最常用的Http会话跟踪机制，且所有`Servlet容器`都应该支持。当客户端不接受`Cookie`时，服务端可使用`URL重写`的方式作为会话跟踪方式。会话`ID`必须被编码为URL字符串中的一个路径参数，参数的名字必须是 `jsessionid`。

浏览器和服务端创建会话(`Session`)后，服务端将生成一个唯一的会话ID(`sessionid`)用于标识用户身份，然后会将这个会话ID通过`Cookie`的形式返回给浏览器，浏览器接受到`Cookie`后会在每次请求后端服务的时候带上服务端设置`Cookie`值，服务端通过读取浏览器的`Cookie`信息就可以获取到用于标识用户身份的会话ID，从而实现会话跟踪和用户身份识别。

## JSP

类似于.php这样的脚本语言，可以在.jsp文件中直接调用java代码来实现后端逻辑，摆脱了传统的servlet处理流程

整个JSP的内容实际上是一个HTML，但是稍有不同：

- 包含在`<%--`和`--%>`之间的是JSP的注释，它们会被完全忽略；
- 包含在`<%`和`%>`之间的是Java代码，可以编写任意Java代码；
- 如果使用`<%= xxx %>`则可以快捷输出一个变量的值。

### 三大指令

```
<%@ page ... %> 定义网页依赖属性，比如脚本语言、error页面、缓存需求等等

<%@ include ... %> 包含其他文件（静态包含）

<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %> 引入标签库的定义
```

### 九大对象

从本质上说 JSP 就是一个Servlet，JSP 引擎在调用 JSP 对应的 jspServlet 时，会传递或创建 9 个与 web 开发相关的对象供 jspServlet 使用。 JSP 技术的设计者为便于开发人员在编写 JSP 页面时获得这些 web 对象的引用，特意定义了 9 个相应的变量，开发人员在JSP页面中通过这些变量就可以快速获得这 9 大对象的引用。

如下：

| 变量名      | 类型                | 作用                                        |
| ----------- | ------------------- | ------------------------------------------- |
| pageContext | PageContext         | 当前页面共享数据，还可以获取其他8个内置对象 |
| request     | HttpServletRequest  | 客户端请求对象，包含了所有客户端请求信息    |
| session     | HttpSession         | 请求会话                                    |
| application | ServletContext      | 全局对象，所有用户间共享数据                |
| response    | HttpServletResponse | 响应对象，主要用于服务器端设置响应信息      |
| page        | Object              | 当前Servlet对象,`this`                      |
| out         | JspWriter           | 输出对象，数据输出到页面上                  |
| config      | ServletConfig       | Servlet的配置对象                           |
| exception   | Throwable           | 异常对象                                    |

### jsp和servlet的关系

两者并没有任何区别，JSP在执行前首先被编译成一个Servlet，只不过无需配置映射路径，Web Server会根据路径查找对应的`.jsp`文件，如果找到了，就自动编译成Servlet再执行

```jsp
<html>
<head>
    <title>Hello World - JSP</title>
</head>
<body>
    <%-- JSP Comment --%>
    <h1>Hello World!</h1>
    <p>
    <%
         out.println("Your IP address is ");
    %>
    <span style="color:red">
        <%= request.getRemoteAddr() %>
    </span>
    </p>
</body>
</html>
```

一个简单的jsp示例

## Filter

javax.servlet.Filter，servlet2.3之后新的特性，主要用于过滤URL请求，通过Filter我们可以实现URL请求资源权限验证、用户登陆检测等功能

![image-20240619221957747](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240619221957747.png)

我们需要重写这三个方法(事实上只要求重写doFilter方法)

```java
package org.example.javaweb;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;
import java.io.PrintWriter;

@WebFilter(filterName="TestFilter",value = "/*")
public class TestFilter implements Filter {

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        String pwd = servletRequest.getParameter("pwd");
        if ("114".equals(pwd)){
            //传递给下一条filter链使用
            filterChain.doFilter(servletRequest,servletResponse);
        }else{
            //输出错误信息
            PrintWriter printWriter = servletResponse.getWriter();
            printWriter.println("pwd error");
            printWriter.flush();
            printWriter.close();
        }
    }
}

```

![image-20240619222914979](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240619222914979.png)

### Servlet和Filter的关系

两者都通过注释或者web.xml来定义，都可以处理http请求，`Filter`和`Servlet`虽然概念上不太一样，但都可以处理Http请求，都可以用来实现MVC控制器

这里概括一下不同点

Filter更偏向于实现对Java Web请求资源的拦截过滤，针对一些攻击行为的禁止

Servlet主要做后端的业务逻辑处理

> 代码审计的重点是Servlet的逻辑问题以及Filter的过滤问题，是否完备

