### JDBC Connection

+ 为什么第一步是Class.forName(CLASS_NAME);//注册JDBC驱动类

  Class.forName会调用静态代码块以及构造方法，而在com.mysql.jdbc.Driver下，静态代码块中注册了驱动包

  如果反射某个类又不想初始化类方法有两种途径：

  1. 使用`Class.forName("xxxx", false, loader)`方法，将第二个参数传入false。
  2. ClassLoader.load("xxxx");

+ 为什么可以省去Class.forName

  Java的SPI特性，DriverManager`在初始化的时候会调用`java.util.ServiceLoader`类提供的SPI机制，Java会自动扫描jar包中的`META-INF/services`目录下的文件，并且还会自动的`Class.forName(文件中定义的类)

