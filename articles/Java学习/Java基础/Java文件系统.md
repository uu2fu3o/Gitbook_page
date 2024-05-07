Java SE中内置了两类文件系统：`java.io`和`java.nio`，`java.nio`的实现是`sun.nio`

### IO文件系统

Java只不过是实现了对文件操作的封装而已，最终读写文件的实现都是通过调用native方法实现的

1. 并不是所有的文件操作都在`java.io.FileSystem`中定义,文件的读取最终调用的是`java.io.FileInputStream#read0、readBytes`、`java.io.RandomAccessFile#read0、readBytes`,而写文件调用的是`java.io.FileOutputStream#writeBytes`、`java.io.RandomAccessFile#write0`。
2. Java有两类文件系统API！一个是基于`阻塞模式的IO`的文件系统，另一是JDK7+基于`NIO.2`的文件系统。

### NIO.2

`java.nio.file.spi.FileSystemProvider`对文件的封装和`java.io.FileSystem`同理

NIO的文件操作在不同的系统的最终实现类也是不一样的，比如Mac的实现类是: `sun.nio.fs.UnixNativeDispatcher`,而Windows的实现类是`sun.nio.fs.WindowsNativeDispatcher`。

合理的利用NIO文件系统这一特性我们可以绕过某些只是防御了`java.io.FileSystem`的`WAF`/`RASP`

## 文件读写

通常读写文件都是使用的阻塞模式，与之对应的也就是`java.io.FileSystem`。`java.io.FileInputStream`类提供了对文件的读取功能，Java的其他读取文件的方法基本上都是封装了`java.io.FileInputStream`类

#### FileInputStream

```java
import java.io.*;
public class filesystem {
    public static void main(String[] args) throws IOException {
        File file =new File("C:\\Users\\Administrator\\IdeaProjects\\javalearn\\src\\javalearn\\src\\test.txt");
        // 打开文件对象并创建文件输入流
        FileInputStream fis = new FileInputStream(file);
        int a =0; //每次读取字节数对象
        byte[] bytes = new byte[1024];//定义缓冲区大小
        // 创建二进制输出流对象
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        // 循环读取文件内容
        while ((a = fis.read(bytes)) != -1) {
            // 截取缓冲区数组中的内容，(bytes, 0, a)其中的0表示从bytes数组的
            // 下标0开始截取，a表示输入流read到的字节数。
            out.write(bytes, 0, a);
        }

        System.out.println(out.toString());
    }
}
```

### FileOutputStream

```java
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
public class filewrite {
    public static  void  main(String[] args) throws IOException {
        //定义写入路径
        File file = new File("C:\\Users\\Administrator\\IdeaProjects\\javalearn\\src\\javalearn\\src\\test.txt");
        //定义内容
        String content ="hello world";
        FileOutputStream fos = new FileOutputStream(file);
        fos.write(content.getBytes());
        fos.flush();
        fos.close();;

    }
}
```

这段代码会覆盖原有的文件内容

### RandomAccessFile

任意文件内容访问类

注意一点，在创建该类对象时

```
            // 创建RandomAccessFile对象,r表示以只读模式打开文件，一共有:r(只读)、rw(读写)、
            // rws(读写内容同步)、rwd(读写内容或元数据同步)四种模式。
            RandomAccessFile raf = new RandomAccessFile(file, "r");
```

### FileSystemProvider

NIO.2的`java.nio.file.spi.FileSystemProvider`,利用`FileSystemProvider`我们可以利用支持异步的通道(`Channel`)模式读取文件内容。

```java
        Path path = Paths.get("C:\\Users\\Administrator\\IdeaProjects\\javalearn\\src\\javalearn\\src\\test.txt");

        try {
            byte[] bytes = Files.readAllBytes(path);
            System.out.println(new String(bytes));
        } catch (IOException e) {
            e.printStackTrace();
        }
```

