## Tomcat架构

[浅析tomcat架构](https://goodapple.top/archives/1359)

[详解](https://juejin.cn/post/7055306172265414663)

![image-20240620222017435](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240620222017435.png)

Tomcat本质上是一个servlet容器，最常见的是Catalina,Catalina负责管理Server,Server表示整个服务器

主要分为三个组件Service,Connector,Container

### Service

其中一个Tomcat Server可以包含多个Service,每一个Service都是独立的，他们共享一个JVM以及系统类库，并且一个Service负责维护多个Connector和一个Container。

不同的Service可以通过不同的端口号来进行访问

### Connector

Connector用于连接Service和Container，解析客户端的请求并转发到Container，以及转发来自Container的响应。每一种不同的Connector都可以处理不同的请求协议，包括HTTP/1.1、HTTP/2、AJP等等。

### Container

Container包含4中组件

- **Engine**

  表示整个Catalina的Servlet引擎，用来管理多个虚拟站点,一个Service最多只能有一个Engine,但是一个Engine可以包含多个Host

- **Host**

  表示一个主机或者是一个虚拟站点
  可以给Tomcat配置多个虚拟主机地址,一个虚拟主机地址下可以包含多个Context

- **Context**

  表示一个web应用程序
  一个尾部应用程序可以包含多个Wrapper

- **Wrapper**

  表示一个Servlet容器
  Wrapper作为容器中的最底层,不可以包含子容器

这4种组件是一种树状结构，通信流程如下

![image-20240620144318022](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240620144318022.png)

#### 三大Context

+ ServletContext

  Servlet规范中规定了一个ServletContext接口，其用来保存一个Web应用中所有Servlet的上下文信息，可以通过ServletContext来对某个Web应用的资源进行访问和操作。

  ```java
  public interface ServletContext {
      String TEMPDIR = "javax.servlet.context.tempdir";
      String ORDERED_LIBS = "javax.servlet.context.orderedLibs";
  
      String getContextPath();
  
      ServletContext getContext(String var1);
  
      int getMajorVersion();
  
      int getMinorVersion();
  
      int getEffectiveMajorVersion();
  
      int getEffectiveMinorVersion();
  
      String getMimeType(String var1);
  
      Set<String> getResourcePaths(String var1);
  ..................................
  ```

  

+ ApplicationContext

  在Tomcat中，ServletContext接口的具体实现就是ApplicationContext类，其实现了ServletContext接口中定义的一些方法。

  ![image-20240625134821223](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240625134821223.png)

  ApplicationContext采用了门面模式进行封装

  ![image-20240625135116408](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240625135116408.png)

+ StandardContext

  该类是子容器context的标准实现类，其中包含了对Context子容器中资源的各种操作。四种容器都有自己的实现，继承于ContainerBase类

  ![image-20240625140824190](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240625140824190.png)

  而在ApplicationContext类中，对资源的各种操作实际上是调用了StandardContext中的方法

  ![image-20240625141028755](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240625141028755.png)

### Servlet Api动态注册机制

Servlet、Listener、Filter 由 `javax.servlet.ServletContext` 去加载，无论是使用 xml 配置文件还是使用 Annotation 注解配置，均由 Web 容器进行初始化，读取其中的配置属性，然后向容器中进行注册。

Servlet 3.0 API 允许使 ServletContext 用动态进行注册，在 Web 容器初始化的时候（即建立ServletContext 对象的时候）进行动态注册。

ServletContext 提供了 add*/create* 方法来实现动态注册的功能。
