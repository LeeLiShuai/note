**Java代码是如何运行起来的**

.java文件编译成.class文件，打包成jar包，使用java -jar命令.启动一个jvm进程。

jvm类加载器加载.class字节码文件，执行类里的逻辑代码。



**jvm类加载机制**

加载-->验证-->准备-->解析-->初始化-->使用-->卸载

将对应的.class字节码文件加载到到内存;

验证.class文件是否符合规范;

为静态便令分配内存空间和变量的初始值,final修饰的静态直接赋原值；

将常量池中的符号引用转为直接引用;

为静态代码块和所有类变量赋值;



**类初始化顺序**

1.父类的静态变量
2.父类的静态代码块
3.子类的静态变量
4.子类的静态代码块
5.父类的非静态变量
6.父类的非静态代码块
7.父类的构造方法
8.子类的非静态变量
9.子类的非静态代码块
10.子类的构造方法



**类加载器**

启动类加载器 Bootstrap ClassLoader,负责加载Java目录下的核心类

拓展类加载器 Extenslon ClassLoader,负责加载java目录下,lib/ext目录的类

应用程序类加载器 Application ClassLoader,负责加载ClassPath环境变量指定的路径中的类

自定义类加载器 可以自己定义类加载器



**双亲委派机制**

类加载器有亲子层级关系，启动类加载是一级，拓展类加载器是耳机，应用程序类加载器是三级，

自定义类加载器是四级。应用程序需要加载一个类的时候，会递归的委派给父类加载器去加载，如果父类加载器在自己负责加载的范围内找不到这个类，才会由子类自己加载。



**不遵守双亲委派机制的情况**

1.一个tomcat可能有多个项目在运行，不同项目可能依赖同一个第三方库，但是版本不同。

2.同版本的第三方库在不同项目中可以共享,同一个版本的类库加载多次浪费。

3.tomcat自己也依赖一些类库，要把这些类库和项目的类库隔离开

4.tomcat支持jsp修改后不重启就生效，jsp也要编译为.class文件运行。

如果使用默认的加载机制，不符合1，默认的加载机制无法加载相同类库的两个不同版本，只根据全限定类型加载。符合2。不符合3。不符合4,jsp重新编译后，因为.class已经加载到内存中，因为累加器没变，所以不会有任何变化。

tomcat实现的策略:

自定义Common ClassLoader,在Application ClassLoader下层,加载路径中的class可以被tomcat和各Webapp访问。

自定义Catalina ClassLoader,在Common ClassLoader下层，tomcat的私有类加载器，对Webapp不可见。

Shared ClassLoader在Common ClassLoader下层，各个Webapp共享的类加载器，对所有Webapp，对tomcat不可见。

WebApp ClassLoader在Shared ClassLoader下层,各Webapp私有的类加载器，只对当前Webapp可见。

Jsp ClassLoader在WebApp下层，每个jsp文件对应一个jsp类加载器。tomcat检测到jsp文件发生修改时，会丢弃当前的jap ClassLoader，重新创建一个新的jsp类加载器加载jsp文件。

tomcat类加载顺序：

1.从缓存中加载

2.如果没有，由Bootstrap类加载器加载

3.如果没有，由WebApp类加载加载

4.如果没有，由父类加载器加载，Extension类加载器---> Application类加载器--->Common类加载器--->Shared类加载器



**jvm内存区域划分**

jvm加载.class文件后需要把文件存起来，一个类文件会用到多次，不可能用一次加载一次，所以必须有一块内存区域存放加载的类信息。JVM中这一区域叫做Metaspace，元数据空间，存放各种类的相关信息。公用。

代码运行起来后要执行一个一个方法， 方法中有很多变量，这些变量需要存放起来，创建的对象也需要存放起来。JVM中这一区域叫做堆，用来存放各种对象。公用。

程序可能有多个线程，每个线程需要记录自己执行到哪一条指令，所以需要一个区域存放这些数据，叫做程序计数器。线程私有。

方法中有一些局部变零，每个线程都有自己的虚拟机栈，这一区域叫做虚拟机栈。线程私有。

Java需要通过调用native方法，调用操作系统里的一些方法。调用这些方法的时候需要本地方法栈。

还有通过DirectByteBuffer引用和操作的对外内存空间。



**堆内存分代**

大部分对象的存活时间都比较短，少部分对象的存活周期较长，如spring项目中的controlelr,service。

为什么要分代？把存活时间短的对象和长的对象分开来，优化gc效率，没有分代时，堆整个堆扫描然后清理。

分代之后只对新生代扫描(没有full gc的情况下)。



**什么时候触发垃圾回收**

创建的对象一般分配在新生代(大对象除外),如果新生代对象越来越多，内存快满了，就会触发垃圾回收,把新生代里没有人引用的对象给回收掉。



**哪些变量引用的对象不能回收**

Java判断对象是否存活使用的是可达性分析，通过判断是否有GC root引用此对象判断对象是否可以被回收。

局部变量，类的静态变量可以作为GC Root,他们引用的对象不会被回收。



**几种引用类型**

强引用，new XXX();最常见的引用,被GC Root引用时，垃圾回收时不会被回收

软引用，new SoftReference<XXX>，软引用，被GC Root引用时，垃圾回收时不会被回收，如果垃圾回收之后，内存还是不足，就会把软引用的对象回收掉

弱引用，new WeakReference<XXX>,弱引用，如果发生垃圾回收，就会被回收。

虚拟用,new PhantomReference<XXX>,虚引用，必须配合ReferenceQueue使用，用户管理对外内存



**几种垃圾回收算法**

复制算法：新生代使用的算法
