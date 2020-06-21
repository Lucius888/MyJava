## JVM - 内存模型

![](https://www.pdai.tech/_images/pics/c9ad2bf4-5580-4018-bce4-1b9a71804d9c.png)

### 程序计数器

记录正在执行的虚拟机字节码指令的地址（如果正在执行的是本地方法则为空）。

### Java虚拟机栈

每个 Java 方法在执行的同时会创建一个栈帧用于存储局部变量表、操作数栈、常量池引用等信息，从调用直至执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中入栈和出栈的过程。

![](https://www.pdai.tech/_images/pics/926c7438-c5e1-4b94-840a-dcb24ff1dafe.png)

可以通过 -Xss 这个虚拟机参数来指定每个线程的 Java 虚拟机栈内存大小：java -Xss512M HackTheJava

该区域可能抛出以下异常：

- 当线程请求的栈深度超过最大值，会抛出 StackOverflowError 异常；
- 栈进行动态扩展时如果无法申请到足够内存，会抛出 OutOfMemoryError 异常。

### 本地方法栈

本地方法一般是用其它语言（C、C++ 或汇编语言等）编写的，并且被编译为基于本机硬件和操作系统的程序，对待这些方法需要特别处理。

本地方法栈与 Java 虚拟机栈类似，它们之间的区别只不过是本地方法栈为本地方法服务。

### 堆

所有对象都在这里分配内存，是垃圾收集的主要区域（"GC 堆"）。

现代的垃圾收集器基本都是采用分代收集算法，针对不同类型的对象采取不同的垃圾回收算法，可以将堆分成两块：

- 新生代（Young Generation）1/3的堆空间
- 老年代（Old Generation）2/3的堆空间

新生代可以继续划分成以下三个空间：

- Eden（伊甸园）
- From Survivor（幸存者）
- To Survivor

堆不需要连续内存，并且可以动态增加其内存，增加失败会抛出 OutOfMemoryError 异常。

可以通过 -Xms 和 -Xmx 两个虚拟机参数来指定一个程序的堆内存大小，第一个参数设置初始值，第二个参数设置最大值。java -Xms1M -Xmx2M HackTheJava

### 方法区

用于存放已被加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

和堆一样不需要连续的内存，并且可以动态扩展，动态扩展失败一样会抛出 OutOfMemoryError 异常。

对这块区域进行垃圾回收的主要目标是对常量池的回收和对类的卸载，但是一般比较难实现。

### 运行时常量池

运行时常量池是方法区的一部分。

Class 文件中的常量池（编译器生成的各种字面量和符号引用）会在类加载后被放入这个区域。

除了在编译期生成的常量，还允许动态生成，例如 String 类的 intern()。



**元空间：持久代被移除，被元空间所代替。位于本地内存中，存储类的元数据。**





### 请你谈谈对OOM的认识

- `java.lang.StackOverflowError`:栈空间溢出 ，递归调用卡死
- `java.lang.OutOfMemoryError:Java heap space`:堆内存溢出 ， 对象过大
- `java.lang.OutOfMemoryError:GC overhead limit exceeded`:GC回收时间过长
- `java.lang.OutOfMemoryError:Direct buffer memory`执行内存挂了，比如：NIO
- `java.lang.OutOfMemoryError:unable to create new native thread`
    - 应用创建了太多线程，一个应用进程创建了多个线程，超过系统承载极限
    - 你的服务器并不允许你的应用程序创建这么多线程，linux系统默认允许单个进程可以创建的线程数是1024，超过这个数量，就会报错
    - 解决办法：降低应用程序创建线程的数量，分析应用给是否针对需要这么多线程，如果不是，减到最低修改linux服务器配置
- `java.lang.OutOfMemoryError:Metaspace`:元空间主要存放了虚拟机加载的类的信息、常量池、静态变量、即时编译后的代码

**造成的原因：长生命周期持有短生命周期引用**



## GC

### 判断是否可被GC

- 引用计数法：为对象添加一个计数器，引用时加一，引用失效时减一。计数器为0时可以被回收
- 可达性分析：以 GC Roots 为起始点进行搜索，可达的对象都是存活的，不可达的对象可被回收。

### Java中什么样的对象才能作为gc root，gc roots有哪些

GC管理的主要区域是Java堆，一般情况下只针对堆进行垃圾回收。方法区、栈和本地方法区不被GC所管理,因而选择这些区域内的对象作为GC roots,被GC roots引用的对象不被GC回收。

因此在java中可作为GcROOT有：

- 虚拟机栈中引用的对象
- 本地方法栈中引用的对象
- 方法区中常量引用的对象
- 方法区中的类静态属性引用的对象

### GC算法

- 标记-清除：标记活动对象，清除未被标记的对象并恢复标记（标记和清除的效率都不高，产生内存碎片）
- 标记-整理：让所有活动对象都向一端移动，然后清除掉端边界之外的（需要移动大量对象因此效率低，内存连续）
- 复制：内存一分为二，一边内存使用完毕就将活的对象复制到另一半中，然后清理掉这一半的内容。（内存只用到一半因此效率不高，多用于回收新生代）
- 分代收集：根据新生代和老年代用不同的收集算法。新生代用复制，老年代用标记-清除/整理

### jvm 有哪些垃圾回收器？

- Serial：最早的单线程串行垃圾回收器。
- Serial Old：Serial 垃圾回收器的老年版本，同样也是单线程的，可以作为 CMS 垃圾回收器的备选预案。
- ParNew：是 Serial 的多线程版本。
- Parallel 和 ParNew 收集器类似是多线程的，但 Parallel 是吞吐量优先的收集器，可以牺牲等待时间换取系统的吞吐量。
- Parallel Old 是 Parallel 老生代版本，Parallel 使用的是复制的内存回收算法，Parallel Old 使用的是标记-整理的内存回收算法。
- CMS：一种以获得最短停顿时间为目标的收集器，
- G1：一种兼顾吞吐量和停顿时间的 GC 实现，是 JDK 9 以后的默认 GC 选项。

### 新生代垃圾回收器和老生代垃圾回收器都有哪些？有什么区别？

- 新生代回收器：Serial、Parallel 、ParNew
- 老年代回收器：Serial Old、Parallel Old、**CMS**
- 整堆回收器：**G1**

新生代垃圾回收器一般采用的是复制算法，复制算法的优点是效率高，缺点是内存利用率低；老年代回收器一般采用的是标记-整理的算法进行垃圾回收。



### CMS回收器工作流程及缺点

- 初始标记：仅仅标记与GC ROOT直接关联的对象，速度很快，程序需要停顿
- 并发标记：进行 GC Roots Tracing的过程，耗时最长，不需要停顿
- 重新标记：标记在并发标记过程之中产生变动需要标记的那部分对象，需要停顿
- 并发清除：进行GC，不需要停顿

**优缺点：**

- 低停顿时间上以吞吐量为代价的，cpu利用率低
- 无法处理并发清除阶段产生的浮动垃圾，因此GC之前需要预留一部分空间，如果预留空间不足，则启用Serial Old进行代替。
- CMS的标记清除会产生内存碎片导致无法连续存储对象，不得不触发一次FullGC

### G1回收器的工作流程及缺点

G1的堆结构就是把一整块内存区域切分成多个固定大小的块region。在JVM在启动时来决定每个region的大小。 JVM一般是把一整块堆切分成2000个小region。然后每个小region从1到32Mb不等。这些region最后又被分别标记为Eden,Survivor和old。在这三个之外，还增加了第四种类型，叫Humongous。

youngGC：eden和surviver中存活的对象被转移到一个/或多个survivor 块上去。 如果存活时间达到阀值,这部分对象就会被晋升到老年代。**复制整理**

oldGC：CMS收集器工作流程类似，

- 初始标记：存活对象的初始标记被固定放在年轻代垃圾收集阶段进行的，
- 并发标记阶段：如果发现有空的块, 则会在 重新阶段立即移除
- 重新标记阶段：空的区域块被移除并回收。
- 复制清除阶段：G1选择“活跃度(liveness)”最低的区域块，这些区域可以最快的完成回收。然后这些区域和年轻代GC在同时被垃圾收集 。

G1收集器可以实现在基本不牺牲吞吐量的前提下提前完成低停顿的内存回收, 这是由于它能够极力避免全区域的垃圾收集, 之前的收集器的收集范围都是**整个新生代**或**整个老年代**, 而G1将整个Java堆**划分**为多个大小固定的独立区域(Region), 并且**跟踪**这些区域里面的垃圾堆积程度, 在后台维护一个**优先列表**, 每次**根据允许的收集时间**, **优先回收垃圾最多的区域**(Garbage First名称的由来). 由于优先列表的存在, 使得G1能在有限的时间内获得最高的收集效率

### Minor GC和Full GC

- Minor GC：回收新生代，由于新生代存活时间很短因此Minor GC 会频繁被执行，执行速度也比较快
- Full GC：回收新生代和老生代，老年代对象其存活时间长，因此 Full GC 很少执行，执行速度也比较慢

### Gc触发条件

- Minor GC：当 Eden 空间满时，就将触发一次 Minor GC。
- Full GC
    - 老年代空间不足：出现的场景就是大对象直接进入老年代、长期存活的对象进入老年代，
    - Concurrent Model Failure 错误直接触发FullGC

### 内存分配策略

- 对象优先分配在Eden区域，空间不足时触发MinorGC
- 大对象直接进入老年代
- 长期存活的对象进入老年代，触发MinorGC时仍旧存活的对象将进入survivor区，年龄加一，到一定年龄则进入老年代。MaxTenuringThreshold 用来定义年龄的阈值。
- 动态对象年龄判定：如果在 Survivor 中相同年龄所有对象大小的总和大于 Survivor 空间的一半，则年龄大于或等于该年龄的对象可以直接进入老年代，无需等到 MaxTenuringThreshold 中要求的年龄。



## java 中都有哪些引用类型

- 强引用：被强引用关联的对象不会被回收；可使用new一个新对象的方式来创建强引用
- 软引用：被软引用的对象只有在内存不够的情况下才会被回收；使用SoftReference类来创建软引用
- 弱引用：被弱引用关联的对象一定会被回收，只能存活到下一次垃圾回收之前；使用WeakReference类类创建弱引用
- 虚引用：无法通过虚引用来得到一个对象，为一个对象设置虚引用的唯一目的是在这个对象回收时收到一个系统通知；使用PhantomReference类来创建虚引用；

## 类加载机制

### 类的生命周期

其中类加载的过程包括了`加载`、`验证`、`准备`、`解析`、`初始化`五个阶段。在这五个阶段中，`加载`、`验证`、`准备`和`初始化`这四个阶段发生的顺序是确定的，*而`解析`阶段则不一定，它在某些情况下可以在初始化阶段之后开始，这是为了支持Java语言的运行时绑定（也成为动态绑定或晚期绑定）*。另外注意这里的几个阶段是按顺序开始，而不是按顺序进行或完成，因为这些阶段通常都是互相交叉地混合进行的，通常在一个阶段执行的过程中调用或激活另一个阶段。

1. 加载：查找并加载类的二进制数据
2. 验证：验证加载的类的正确性；
3. 准备：给类中的静态变量分配内存空间；
4. 解析：把类中的符号引用转换为直接引用
5. 初始化：对静态变量和静态代码块执行初始化工作。

### 类加载器

#### 类加载器的层次

![](https://www.pdai.tech/_images/jvm/java_jvm_classload_3.png)

这里父类加载器并不是通过继承关系来实现的，而是采用组合实现的。

站在Java虚拟机的角度来讲，只存在两种不同的类加载器：启动类加载器：它使用C++实现（这里仅限于`Hotspot`，也就是JDK1.5之后默认的虚拟机，有很多其他的虚拟机是用Java语言实现的），是虚拟机自身的一部分；所有其他的类加载器：这些类加载器都由Java语言实现，独立于虚拟机之外，并且全部继承自抽象类`java.lang.ClassLoader`，这些类加载器需要由启动类加载器加载到内存中之后才能去加载其他的类。

### 站在Java开发人员，类加载器可以大致划分为以下三类

`启动类加载器`：Bootstrap ClassLoader，负责加载存放在JDK\jre\lib(JDK代表JDK的安装目录，下同)下，或被-Xbootclasspath参数指定的路径中的，并且能被虚拟机识别的类库（如rt.jar，所有的java.*开头的类均被Bootstrap ClassLoader加载）。启动类加载器是无法被Java程序直接引用的。

`扩展类加载器`：Extension ClassLoader，该加载器由`sun.misc.Launcher$ExtClassLoader`实现，它负责加载JDK\jre\lib\ext目录中，或者由java.ext.dirs系统变量指定的路径中的所有类库（如javax.*开头的类），开发者可以直接使用扩展类加载器。

`应用程序类加载器`：Application ClassLoader，该类加载器由`sun.misc.Launcher$AppClassLoader`来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

### 类的加载方式

- 命令行启动应用时候由JVM初始化加载
- 通过Class.forName()方法动态加载
- 通过ClassLoader.loadClass()方法动态加载

### JVM类加载机制

- `全盘负责`，当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入

- `父类委托`，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类

- `缓存机制`，缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效

- `双亲委派机制`, 如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。

    - 当AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。
    - 当ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader去完成。
    - 如果BootStrapClassLoader加载失败（例如在$JAVA_HOME/jre/lib里未查找到该class），会使用ExtClassLoader来尝试加载；
    - 若ExtClassLoader也加载失败，则会使用AppClassLoader来加载，如果AppClassLoader也加载失败，则会报出异常ClassNotFoundException。

    ### 双亲委派模型介绍

    每一个类都有一个对应它的类加载器。系统中的 ClassLoder 在协同工作的时候会默认使用 **双亲委派模型** 。即在类加载的时候，系统会首先判断当前类是否被加载过。已经被加载的类会直接返回，否则才会尝试加载。加载的时候，首先会把该请求委派该父类加载器的 `loadClass()` 处理，因此所有的请求最终都应该传送到顶层的启动类加载器 `BootstrapClassLoader` 中。当父类加载器无法处理时，才由自己来处理。当父类加载器为null时，会使用启动类加载器 `BootstrapClassLoader` 作为父类加载器。

    ![参考-JavaGuide-ClassLoader](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/classloader_WPS图片.png)

    ```java
    public class ClassLoaderDemo {
        public static void main(String[] args) {
            System.out.println("ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader());
            System.out.println("The Parent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader().getParent());
            System.out.println("The GrandParent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader().getParent().getParent());
        }
    }
    ClassLodarDemo's ClassLoader is sun.misc.Launcher$AppClassLoader@18b4aac2
    The Parent of ClassLodarDemo's ClassLoader is sun.misc.Launcher$ExtClassLoader@1b6d3586
    The GrandParent of ClassLodarDemo's ClassLoader is null
    ```

    `AppClassLoader`的父类加载器为`ExtClassLoader`
    `ExtClassLoader`的父类加载器为null，**null并不代表`ExtClassLoader`没有父类加载器，而是 `BootstrapClassLoader`** 。

    其实这个双亲翻译的容易让别人误解，我们一般理解的双亲都是父母，这里的双亲更多地表达的是“父母这一辈”的人而已，并不是说真的有一个 Mother ClassLoader 和一个 Father ClassLoader 。另外，类加载器之间的“父子”关系也不是通过继承来体现的，是由“优先级”来决定。

    ### 双亲委派模型的好处

    双亲委派模型保证了Java程序的稳定运行，可以**避免类的重复加载**（JVM 区分不同类的方式不仅仅根据类名，相同的类文件被不同的类加载器加载产生的是两个不同的类），也保证了 Java 的核心 API 不被篡改。如果没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，比如我们编写一个称为 `java.lang.Object` 类的话，那么程序运行的时候，系统就会出现多个不同的 `Object` 类。

    ### 如果我们不想用双亲委派模型怎么办？

    为了避免双亲委托机制，我们可以自己定义一个类加载器，然后重写 `loadClass()` 即可。

    ### 能否自定义 Java.Object.String 的类加载器 ？
    
    不能，因为针对java.*开头的类，jvm的实现中已经保证了必须由bootstrp来加载
    
    

### 说一下 jvm 调优的工具？

JDK 自带了很多监控工具，都位于 JDK 的 bin 目录下，其中最常用的是 jconsole 和 jvisualvm 这两款视图监控工具。



- jconsole：用于对 JVM 中的内存、线程和类等进行监控；
- jvisualvm：JDK 自带的全能分析工具，可以分析：内存快照、线程快照、程序死锁、监控内存的变化、gc 变化等。