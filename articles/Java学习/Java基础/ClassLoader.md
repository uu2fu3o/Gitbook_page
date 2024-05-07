Java程序在运行前需要先编译成`class文件`，Java类初始化的时候会调用`java.lang.ClassLoader`加载类字节码，`ClassLoader`会调用JVM的native方法（`defineClass0/1/2`）来定义一个`java.lang.Class`实例。

## 类加载方式

显式 -->动态加载

通过反射或者classLoader进行加载

通过反射

`Class.forName("类名")`默认会初始化被加载类的静态属性和方法，如果不希望初始化类可以使用`Class.forName("类名", 是否初始化类, 类加载器)`

通过ClassLoader

`ClassLoader.loadClass`默认不会初始化类方法。

### 流程

调用public Class<?> loadClass(String name)加载类

调用`findLoadedClass`方法检查类是否已经初始化，如果已经初始化则返回类对象，后面的流程不再继续

创建当前`ClassLoader`时传入了父类加载器（`new ClassLoader(父类加载器)`）就使用父类加载器加载`TestHelloWorld`类，否则使用JVM的`Bootstrap ClassLoader`加载。

若上一步无法加载调用自身的`findClass`方法尝试加载`TestHelloWorld`类

如果当前的`ClassLoader`没有重写`findClass`方法，那么直接返回类加载失败异常。如果当前类重写了`findClass`方法并通过传入的类名找到了对应的类字节码，那么应该调用`defineClass`方法去JVM中注册该类。

如果调用loadClass的时候传入的`resolve`参数为true，那么还需要调用`resolveClass`方法链接类，默认为false。

返回类对象

## URLClasserLoader

继承于java.lang.Classloader,`URLClassLoader`提供了加载远程资源的能力，在写漏洞利用的`payload`或者`webshell`的时候我们可以使用这个特性来加载远程的jar来实现远程的类方法调用