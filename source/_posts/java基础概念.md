---
title: java基础概念
tags: java
categories: 基础
abbrlink: a7ba99a3
date: 2018-04-09 17:24:07
---

## 基本概念

JDK：包括编译器（javac.exe）、开发工具（javadoc.exe、jar.exe、keytool.exe、jconsole.exe）和更多的类库（如tools.jar）等。总结就是Java语言、Java虚拟机、Java API类库
JRE： 支持java运行的基本环境

![image](https://user-images.githubusercontent.com/7789698/29321251-65a89fd0-820c-11e7-93e7-b15d5c2db752.png)


JIT：just in time

运行时数据区：
![image](https://user-images.githubusercontent.com/7789698/29417094-9fc9b106-839a-11e7-9be9-310174cd0d60.png)

## JVM的内存结构

### 程序计数器：

  当前代码行号指示器。各个线程计数器互不影响，独立存储。如果是执行Java方法，是虚拟机字节码地址。*如果是native方法，计数器为空。*唯一一个没有OOM的区域。是当前线程所执行的字节码的行号指示器，每条线程都要有一个独立的程序计数器，这类内存也称为“**线程私有**”的内存。

程序计数器是指CPU中的寄存器，它保存的是程序当前执行的指令的地址（也可以说保存下一条指令的所在存储单元的地址），当CPU需要执行指令时，需要从程序计数器中得到当前需要执行的指令所在存储单元的地址，然后根据得到的地址获取到指令，在得到指令之后，程序计数器便自动加1或者根据转移指针得到下一条指令的地址，如此循环，直至执行完所有的指令。　

在JVM规范中规定，如果线程执行的是非native方法，则程序计数器中保存的是当前需要执行的指令的地址；如果线程执行的是native方法，则程序计数器中的值是undefined。由于程序计数器中存储的数据所占空间的大小不会随程序的执行而发生改变，因此，对于程序计数器是不会发生内存溢出现象(OutOfMemory)的。


### Java虚拟机栈：
  描述Java方法执行的内存模型：每个执行时会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等。每个方法调用到执行完成过程，对应一个栈帧在虚拟机栈中入栈到出栈的过程。**线程私有**。

  - 局部变量表：

    存放编译期可知的各种基本数据类型、对象引用类型（一个指向对象起始地址的引用指针）、returnAddress类型（一个字节码指令地址）。其中64位长度long和double会占用2个局部变量空间，其余占一个。局部变量表所需内存大小是在编译期间完成分配的，方法运行期间不会改变。如果栈深度大于虚拟机允许的深度，将抛出StackOverflowError异常；如果虚拟机栈动态扩张时无法申请到足够的内存，就会抛出OutOfMemoryError异常。



<!-- more -->

### 本地方法栈：
  为虚拟机使用的Native方法服务。一样会抛出StackOverflowError和OutOfMemoryError。**线程私有**。
### Java堆：
  存放对象实例，被所有**线程共享**。所有对象实例以及数组都要在堆上分配，但随着JIT编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致所有对象都在堆上分配也不是那么绝对。从内存分配看，可以划分出多个线程私有分配缓冲区。
### 方法区：
  用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。（各个线程共享）。方法区是堆的一个逻辑部分，但是他有个别名非堆，为了和Java堆区区分。

- 运行时常量：

  运行时常量是方法区的一部分（无法再申请内存时自然OOM）。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译期生成的字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。Java虚拟机对于Class文件每一部分（包括常量池）格式都有严格规定，每一个字节用于存储哪种数据都必须符合规范才会被认可、装载和运行。但是运行时常量池，每个虚拟机提供商都可以按自己需要实现。一般Class文件中描述的符号引用和直接引用都存储在其中。除了编译器外，运行过程也可以放入，比如String的intern()方法。

### 直接内存：
  JDK1.4引入的NIO，引入了一种基于通道与缓冲区的I/O方式，可以使用native函数库直接分配堆外内存，通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，这样避免Java堆和native堆中来回复制数据。（内存受到RAM、SWAP或分页大小以及处理器寻址空间的限制）

## 类的加载过程



![image](https://user-images.githubusercontent.com/7789698/29859692-e6762e90-8d95-11e7-8311-ebabbc6f1ad1.png)

### 加载

1.通过一个类的全限定名获取类的二进制字节流

2.字节流代表的静态存储结构转化为方法区运行时数据结构

3.在内存中生成一个代表类的java.lang.Class对象作为方法区这个类各个数据的访问入口

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需要的格式存储在方法区之中，方法区中的数据存储格式由虚拟机实现自行定义，虚拟机规范未规定此区域的具体数据结构。然后在内存中实例化一个java.lang.Class类的对象，这个对象将作为程序访问方法区中的这些类型数据的外部接口。

### 验证

验证时连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全，主要包含以下内容

#### 文件格式验证：

1.第一阶段要验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理包含以下内容
是否以魔数0xCAFEBABE开头

2.主次版本号是否在当前虚拟机处理范围之内

3.常量池的常量中是否有不被支持的常量类型

4.指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量

5.CONSTANT_Utf8_info 型的常量中是否有不符合UFT8编码的数据

6.Class文件中各个部分及文件本身是否有本删除的或附加的其他信息

·····

只有通过了这个阶段的验证后，字节流才会进入内存的方法区中进行存储，所以后面的3个验证阶段全部是基于方法区的存储结构进行的，不会再直接操作字节流。

#### 元数据验证

第二阶段是对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言的规范的要求，包含以下信息
1.这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）

2.这个类的父类是否继承了不允许被继承的类

3.如果这个类不是抽象类，是否实现了其父类或者接口要求实现的所有的方法

4.类中的字段，方法是否与父类产生矛盾

······

#### 字节码验证

第三阶段是整个验证过程中最复杂的一个阶段，主要目的是通过数据流和控制流分析，确定程序语法是否是合法的，符合逻辑的。在第二阶段对元数据信息中的数据类型做完校验后，这个阶段将对类的方法体进行校验分析，保证被校验类的方法在运行时候不会做出对虚拟机有危害的事情。包含如下内容

1.保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似这种情况：在操作栈中放置一个int类型数据，使用却将他按照long类型使用

2.保证跳转指令不会跳转到方法体以外的字节码指令上

3.保证方法体中的类型转换是有效的，例如可以把一个子类对象赋给父类数据类型，这是安全的，但是把父类对象赋值给子类数据类型则是危险的

······

#### 符号引用验证

最后一个阶段的校验发生在虚拟机将符号引用转化为直接引用的时候，这个转换动作发生在连接的第三阶段--解析中发生，符号引用可以看做是对类自身以外的信息进行匹配性校验，通常需要校验一下内容：

1.符号引用中通过字符串描述的全限定名是否能找到对应的类

2.在指定的类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段

3.符号引用中的类，字段，方法的访问性是否可以被当前类访问。



### 准备

准备阶段是正式为 类变量 分配内存并设置变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配，记住，只为类变量分配内存，不包括实例变量，实例变量将会在对象实例化时随对象一起分配在java堆中

![image](https://user-images.githubusercontent.com/7789698/29861734-cfc3bb0c-8d9c-11e7-8c61-b2ae4d86272b.png)

**如果类字段的字段属性表中存在ConstantValue属性，那么在准备阶段变量value就会被初始化为ConstantValue属性所指定的值**，例如：

public static final int value = 123;//在准备阶段虚拟机就会根据ConstantValue设置将value赋值为123

### 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，对同一个符号引用进行多次解析请求是很常见的事情，虚拟机不会重新再解析而是通过缓存去拿出解析的数据，但是invokedynamic指令除外，它会每次被解析都会被重新解析，解析动作主要针对类，接口，字段，类方法，接口方法，方法类型，方法句柄和调用点限定符7类符号引用进行,主要包含以下内容

#### 符号引用：

符号引用以一组符号来描述所引用的目标，符号可以是任意形式的字面量，只要使用时能够无歧义的定位到目标即可。

#### 直接引用：

直接引用可以是直接指向目标的指针，相对偏移量或是一个能间接定位到目标的句柄。

1.类或接口的解析
2.字段解析
3.类方法解析
4.接口方法解析

解析阶段并不是一定的：Java的运行时绑定（动态绑定）。

### 初始化

类初始化阶段是类加载过程的最后一步，前面的类加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作全部由虚拟机主导和控制，到了初始化阶段，才真正开始执行类中定义的Java程序代码，在准备阶段变量已经赋过一次系统要求的初始值，而在初始化阶段则通过程序制定的主观计划去初始化变量和其他资源，从另一个角度理解就是执行类构造器`<clinit>()`方法的过程`<clinit>()`方法是由编译器自动收集类中的所有变量的赋值动作和静态语句块中的语句合并产生的，他按照代码中出现的顺序收集，静态语句块中只能访问到定义在静态语句块之前的变量，定义在他之后的，在静态语句块中只能赋值不能访问

```
public class Test {
  static{
      i = 1;//可以赋值
      System.out.println(i);//不能访问
  }
  static int i = 0;
}
```

1.`<clinit>()`方法在执行之前必须保证自己父类的类构造器方法已经执行完毕,因此在虚拟机中第一个被执行的`<clinit>()`方法的类肯定是java.lang.Object
2.由于父类的`<clinit>()`方法优先执行，意味着父类中定义的静态语句块要优先于子类的变量赋值操作
3.`<clinit>()`并不是必须的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成`<clinit>()`方法。
4.接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成`<clinit>()`方法，但是接口与类不同的是，执行接口的`<clinit>()`方法不需要先执行父类接口的`<clinit>()`方法，只有父类接口中定义的变量使用时父类接口才会初始化，另外接口实现类在初始化时也一样不会执行接口的`<clinit>()`方法
5.虚拟机会保证一个类的`<clinit>()`方法在多线程环境中被正确的加锁，同步



初始化：
1.new、getstatic、putstatic、invokestatic这四条字节码指令（new实例化对象、读取或者设置一个静态字段、调用一个类的静态方法）

2.java.lang.reflect反射

3.初始化一个类，父类也初始化。但是一个接口初始化时，并不要求父类接口全部完成初始化，真正使用父接口才会初始化

4.虚拟机启动的时候，主类

5.java.lang.invoke.MethodHandle实例最后解析结果REF_getStatic、REF_putStatic、REF_invokeStatic方法句柄

```java
public class SuperClass {
    static{
        System.out.println("SuperClass init!");
    }
    public static int value = 100;
}

public class SubClass extends SuperClass{
    static {
        System.out.println("SubClass init!");
    }
}

public class NotInitialization {
    public static void main(String[] args) {
        System.out.println(SubClass.value);
    }
}
```

上面代码运行后只会输出“SuperClass init! ”,对于静态代码字段，只有直接定义这个字段的类才会被初始化，因此通过子类引用父类中静态字段，只会触发父类的初始化而不会触发子类的初始化。-XX:+TraceClassLoading会导致子类被加载

```java
public class NotInitialization {
    public static void main(String[] args) {
        SuperClass[] sca = new SuperClass[10];
    }
}
```

通过数组引用不会初始化类

```java
public class ConstClass {
    static {
        System.out.println("ConstantClass init!");
    }

    public static final String HELLO = "hello world";
}

public class NotInitialization {
    public static void main(String[] args) {
        System.out.println(ConstClass.HELLO);
    }
}
```

常量在编译阶段会存入调用类的常量池中本质上没有直接引用到定义常量的类，因此不会触发定义常量类的初始化



## new过程

new的时候：1.检查指令参数是否能在常量池中定位到一个类的符号引用2.检查这个符号引用代表的类是否已被加载、解析、初始化过3.有就执行相应类加载过程4.类加载检查通过后，虚拟机为新生对象分配内存（假设Java堆内存绝对规整，用过的一边没用过的一边，中间放着指针作为分界点的指示器，指针则挪动和对象大小的距离，这叫做“指针碰撞”；假如不规整，已使用和空闲交错，虚拟机就必须维护一个列表，记录哪些内存块可用，在分配的时候从列表找到一个足够大的空间划分给对象实例，并更新列表上的记录，这叫做“空闲列表”；使用Serial、ParNew等带Compact过程收集器，采用指针碰撞，使用CMS基于Mark-Sweep算法收集器，采用空闲列表）

并发情况下不是线程安全的，存在正在给A分配内存，指针还没来得及修改，对象B又同时使用原来指针来分配内存。有两种解决方法，一、对分配内存空间的动作进行同步处理——虚拟机采用CAS+失败重试保证更新原子性；二、按内存分配动作按线程划分不同空间进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓存（TLAB）。哪个线程要分配，就在TLAB上分配，用完才分配新的，才需要同步锁定。（-XX:+/-UseTLAB）

内存分配后，会初始化内存的零值。接下来，虚拟机对对象进行必要设置，存放信息（对象是哪个类实例、如何找类元信息、哈希码、GC分代年龄）在对象头。从虚拟机角度对象已经创建，从java角度对象才刚刚开始创建init还没开始执行，所有字段为0。执行new指令以后接着执行init（invokespecial）
![image](https://user-images.githubusercontent.com/7789698/29488667-520dadd8-8542-11e7-99fc-c50fdad18138.png)

## 对象的内存布局

在HotSpot虚拟机中，对象在内存中存储的布局可以分为3块区域：对象头(Header)、实例数据(Instance Data)和对齐填充(Padding)。

### 对象头

对象头包括两部分信息：标记字段和类型指针。

#### 标记字段 Mark Word

用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。

32bit中25bit存储对象哈希码，4bit存储对象分代年龄，2bit存储锁标志位，1bit固定为0
![image](https://user-images.githubusercontent.com/7789698/29488278-1a557e0a-853a-11e7-81d6-ef16d53f9f85.png)

#### 类型指针 Klass Pointer

对象头另一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中无法确定数组的大小。 
(并不是所有的虚拟机实现都必须在对象数据上保留类型指针，换句话说，查找对象的元数据并不一定要经过对象本身，可参考对象的访问定位)

### Monitor

每一个Java对象都有成为Monitor的潜质，因为在Java的设计中 ，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁。 

Monitor 是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联（对象头的MarkWord中的LockWord指向monitor的起始地址），同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。其结构如下：  

![image](https://user-images.githubusercontent.com/7789698/38797787-3bf97a92-4192-11e8-8a76-1475f4cfeb6b.png)

- Owner：初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL； 
- EntryQ:关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程。 
- RcThis:表示blocked或waiting在该monitor record上的所有线程的个数。 
- Nest:用来实现重入锁的计数。 
- HashCode:保存从对象头拷贝过来的HashCode值（可能还包含GC age）。 
- Candidate:用来避免不必要的阻塞或等待线程唤醒，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate只有两种可能的值0表示没有需要唤醒的线程1表示要唤醒一个继任线程来竞争锁。 摘自：Java中synchronized的实现原理与应用） 



### 数据类型

这部分分配顺序会受到虚拟机分配策略参数和字段在Java源码中定义顺序的影响。HotSpot虚拟机默认的分配策略为longs/doubles、ints、shorts/chars、bytes/booleans、oop，从分配策略中可以看出，相同宽度的字段总是分配到一起。

### 对齐填充

HotSpot虚拟机要求对象的起始地址必须是8字节的整数倍，也就是对象的大小必须是8字节的整数倍。而对象头部分正好是8字节的倍数（1倍或者2倍），因此，当对象实例数据部分没有对齐的时候，就需要通过对齐填充来补全。

### 对象访问定位

Java程序需要通过栈上的引用数据来操作堆上的具体对象。对象的访问方式取决于虚拟机实现，目前主流的访问方式有使用句柄和直接指针两种。

#### 句柄

可以理解为指向指针的指针，维护指向对象的指针变化，而对象的句柄本身不发生变化；指针，指向对象，代表对象的内存地址。

![image](https://user-images.githubusercontent.com/7789698/29493798-0f40d4ec-85d0-11e7-9137-0b82675cd697.png)

优势：引用中存储的是稳定的句柄地址，在对象被移动(垃圾收集时移动对象是非常普遍的行为)时只会改变句柄中的实例数据指针，而引用本身不需要修改。

#### 直接指针

如果使用直接指针访问，那么Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而引用中存储的直接就是对象地址。

![image](https://user-images.githubusercontent.com/7789698/29493827-ab95007a-85d0-11e7-82b1-b87f0a823406.png)


优势：速度更快，节省了一次指针定位的时间开销。由于对象的访问在Java中非常频繁，因此这类开销积少成多后也是非常可观的执行成本。

## 垃圾回收算法
#### 引用计数法
  原理：给每个对象添加一个计数器，被引用计数器就加一；引用失效就减一；计数器为0就对象回收
  缺陷：两对象之间循环引用则无法回收

#### 可达性分析
原理：通过一系列“GC Roots”对象称为起始点，从这些结点开始向下搜索，搜索走过的链路称为引用链路，当GC不可达的时候会被判定可回收。

Java里，可以作为GC Roots对象的有：

1.**虚拟机栈**（栈帧本地变量）引用的对象

2.方法区中**类静态属性**引用的对象

3.方法区中**常量**引用的对象

4.**本地方法栈**（JNI）引用的对象

### 引用

- 强引用：Object a = new Object() 只要强引用存在，永远不会回收
- 软引用：描述有用非必须。系统内存即将发生溢出异常之前，会在回收范围内进行回收。SoftReference
- 弱引用：非必需。强度比软引用弱，被弱引用关联对象只能生存到下一次垃圾收集之前。无论内存是否足够，都会回收被弱引用关联的对象。WeakReference
- 虚引用：最弱的。完全不会对生存时间构成影响，无法通过虚引用取到一个对象实例。作用是这个对象被收集器回收时收到一个系统通知。PhantomReference
### JVM内存分配策略

- 优先在Eden区分配
  在[JVM内存模型](http://blog.csdn.net/zjf280441589/article/details/53437703)一文中, 我们大致了解了VM年轻代堆内存可以划分为一块Eden区和两块Survivor区. 在大多数情况下, 对象在新生代Eden区中分配, 当Eden区没有足够空间分配时, VM发起一次Minor GC, 将Eden区和其中一块Survivor区内尚存活的对象放入另一块Survivor区域, 如果在Minor GC期间发现新生代存活对象无法放入空闲的Survivor区, 则会通过空间分配担保机制使对象提前进入老年代(空间分配担保见下).
- 大对象直接进入老年代
  Serial和ParNew两款收集器提供了-XX:PretenureSizeThreshold的参数, 令大于该值的大对象直接在老年代分配, 这样做的目的是避免在Eden区和Survivor区之间产生大量的内存复制(大对象一般指 需要大量连续内存的Java对象, 如很长的字符串和数组), 因此大对象容易导致还有不少空闲内存就提前触发GC以获取足够的连续空间.
- 长期存活的对象将进入老年代
  VM为每个对象定义了一个对象年龄(Age)计数器, 对象在Eden出生如果经第一次Minor GC后仍然存活, 且能被Survivor容纳的话, 将被移动到Survivor空间中, 并将年龄设为1. 以后对象在Survivor区中每熬过一次Minor GC年龄就+1. 当增加到一定程度(-XX:MaxTenuringThreshold, 默认15), 将会晋升到老年代.
- 幸存区相同年龄对象的占幸存区空间的多于其一半，将进入老年代
  然而VM并不总是要求对象的年龄必须达到
  MaxTenuringThreshold才能晋升老年代: 如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半, 年龄大于或等于该年龄的对象就可以直接进入老年代, 而无须等到晋升年龄.
- 空间担保分配（老年代剩余空间需多于幸存区的一半，否则要Full GC）

### 垃圾回收算法
- 标记-清除算法：标记(可达性分析)出所有需要回收的对象，标记完成后统一回收。

  不足：效率低、不连续**碎片多**（没有足够连续空间分配内存，提前触发另一次垃圾回收）

  适用：对象存活率高的**老年代**。

  ![image](https://user-images.githubusercontent.com/7789698/38491054-6d24af94-3c1d-11e8-8670-811ccd356fa5.png)


- 复制算法：将可用内存分为两块，每次只使用其中一块，内存用完了就将存活对象复制到另一块并清理。一般会分为内存较大的Eden和两块内存较小的Surivivor（默认8:1:1），内存回收的时候会将Eden和Surivivor存活的对象一次性复制到另一块Surivivor，并且清理空间。（Surivivor不够用则向Old区进行分配担保）

  不足：内存缩小为原来的一般，代价高。浪费50%的空间。

  适用：**新生代**。

  ![image](https://user-images.githubusercontent.com/7789698/38491082-7e1a3620-3c1d-11e8-8663-edb261529e43.png)


- 标记-整理算法：标记出所有需要回收的对象，所有存活对象都向一端移动，直接清理端以外内存

  适用：**老年代**。

  

  ![img](https://user-images.githubusercontent.com/7789698/38491187-bb8d974a-3c1d-11e8-8a26-6b7f42afe9e7.png)

#### 新生代和老年代的回收策略

Eden区满后触发minor GC,将所有存活对象复制到一个Survivor区，另一Survivor区存活的对象也复制到这个Survivor区中，始终保证有一个Survivor是空的。当Survivor空间不够用(不足以保存尚存活的对象)时, 需要依赖Old区进行空间分配担保机制, 这部分内存直接进入Old区。

Young区Survivor满后触发minor GC后仍然存活的对象存到Old区，如果Survivor区放不下Eden区的对象或者Survivor区对象足够老了，直接放入Old区，如果Old区放不下则触发Full GC。

#### 永久代-方法区回收

在方法区进行垃圾回收一般”性价比”较低, 因为在方法区主要回收两部分内容: **废弃常量**和**无用的类**。 回收废弃常量与回收其他年代中的对象类似, 但要判断一个类是否无用则条件相当苛刻:

- 该类所有的**实例**都已经**被回收**, Java堆中不存在该类的任何实例;
- 该类对应的Class对象**没有**在任何地方**被引用**(也就是在任何地方都无法通过反射访问该类的方法);
- 加载该类的**ClassLoader**已经被回收.

但即使满足以上条件也未必一定会回收, Hotspot VM还提供了-Xnoclassgc参数控制(关闭CLASS的垃圾回收功能). 因此在大量使用动态代理、CGLib等字节码框架的应用中一定要关闭该选项, 开启VM的类卸载功能, 以保证方法区不会溢出。

#### 空间分配担保

在执行Minor GC前, VM会首先检查老年代是否有足够的空间存放新生代尚存活对象, 由于新生代使用复制收集算法, 为了提升内存利用率, 只使用了其中一个Survivor作为轮换备份, 因此当出现大量对象在Minor GC后仍然存活的情况时, 就需要老年代进行分配担保, 让Survivor无法容纳的对象直接进入老年代, 但前提是老年代需要有足够的空间容纳这些存活对象. 但存活对象的大小在实际完成GC前是无法明确知道的, 因此Minor GC前, VM会先首先检查**老年代连续空间**是否大于**新生代对象总大小或历次晋升的平均大小**, 如果条件成立, 则进行Minor GC, 否则进行Full GC(让老年代腾出更多空间).

然而取历次晋升的对象的平均大小也是有一定风险的, 如果某次Minor GC存活后的对象突增,远远高于平均值的话,依然可能导致担保失败(Handle Promotion Failure, 老年代也无法存放这些对象了), 此时就只好在失败后重新发起一次Full GC(让老年代腾出更多空间).


**枚举根结点**  必须停顿所有Java线程（STW stop the world），采用OopMap数据结构检查所有上下文和全局的引用位置

**安全点** 每条指令都生成OopMap很耗费空间，实际上HotSpot没有为每条指令生成OopMap，而是只在特定位置记录，这些位置被称为“安全点”。（标准是 是否让程序长时间执行，如循环、方法调用等）。让GC在安全点停顿，HotSpot采用主动式中断（当GC需要中断线程的时候，只是标记，各个线程轮训这个标志，发现这个标志为真则挂起线程，和安全点重合则加上创建线程需要分配的位置）

新生代GC（Minor GC）

老年代GC （Major／Full GC）



### 新生代

- Serial收集器

  Serial收集器是Hotspot运行在Client模式下的默认新生代收集器, 它的特点是 只用一个CPU/一条收集线程去完成GC工作, 且在进行垃圾收集时必须暂停其他所有的工作线程(“Stop The World” -后面简称STW).

  ![image](https://user-images.githubusercontent.com/7789698/38507171-6dbd8b16-3c4e-11e8-9eee-e04ba8af9f62.png)

  ​

  虽然是单线程收集, 但它却简单而高效, 在VM管理内存不大的情况下(收集几十M~一两百M的新生代), 停顿时间完全可以控制在几十毫秒~一百多毫秒内.



- ParNew收集器

  ParNew收集器其实是前面Serial的多线程版本, 除使用多条线程进行GC外, 包括Serial可用的所有控制参数、收集算法、STW、对象分配规则、回收策略等都与Serial完全一样(也是VM启用CMS收集器-XX: +UseConcMarkSweepGC的默认新生代收集器).

  ![image](https://user-images.githubusercontent.com/7789698/38507517-4fb1f96c-3c4f-11e8-8f4d-78196df53dc2.png)

  由于存在线程切换的开销, ParNew在单CPU的环境中比不上Serial, 且在通过超线程技术实现的两个CPU的环境中也不能100%保证能超越Serial. 但随着可用的CPU数量的增加, 收集效率肯定也会大大增加(ParNew收集线程数与CPU的数量相同, 因此在CPU数量过大的环境中, 可用-XX:ParallelGCThreads参数控制GC线程数).

  ​

- Parallel Scavenge收集器

  与ParNew类似, Parallel Scavenge也是使用复制算法, 也是并行多线程收集器. 但与其他收集器关注尽可能缩短垃圾收集时间不同, Parallel Scavenge更关注系统吞吐量:

  系统吞吐量=运行用户代码时间(运行用户代码时间+垃圾收集时间)

  停顿时间越短就越适用于用户交互的程序-良好的响应速度能提升用户的体验;而高吞吐量则适用于后台运算而不需要太多交互的任务-可以最高效率地利用CPU时间,尽快地完成程序的运算任务. Parallel Scavenge提供了如下参数设置系统吞吐量:

  | Parallel Scavenge参数      | 描述                                                         |
  | -------------------------- | ------------------------------------------------------------ |
  | MaxGCPauseMillis           | (毫秒数) 收集器将尽力保证内存回收花费的时间不超过设定值, 但如果太小将会导致GC的频率增加. |
  | GCTimeRatio                | (整数:0 < GCTimeRatio < 100) 是垃圾收集时间占总时间的比率    |
  | -XX:+UseAdaptiveSizePolicy | 启用GC自适应的调节策略: 不再需要手工指定-Xmn、-XX:SurvivorRatio、-XX:PretenureSizeThreshold等细节参数, VM会根据当前系统的运行情况收集性能监控信息, 动态调整这些参数以提供最合适的停顿时间或最大的吞吐量 |

### 老年代

- Serial Old收集器

  Serial Old是Serial收集器的老年代版本, 同样是单线程收集器,使用“标记-整理”算法:
  ![image](https://user-images.githubusercontent.com/7789698/38507541-638579a0-3c4f-11e8-88e8-f9c730be7707.png)

  - Serial Old应用场景如下:
    - JDK 1.5之前与Parallel Scavenge收集器搭配使用;
    - 作为CMS收集器的后备预案, 在并发收集发生Concurrent Mode Failure时启用(见下:CMS收集器).


- Parallel Old收集器

  Parallel Old是Parallel Scavenge收老年代版本, 使用多线程和“标记－整理”算法, 吞吐量优先, 主要与Parallel Scavenge配合在 注重吞吐量 及 CPU资源敏感 系统内使用:

  ![image](https://user-images.githubusercontent.com/7789698/38508265-4e6bfac4-3c51-11e8-82b3-f653b5d44508.png)



- CMS收集器

  CMS(Concurrent Mark Sweep)收集器是一款具有划时代意义的收集器, 一款真正意义上的并发收集器, 虽然现在已经有了理论意义上表现更好的G1收集器, 但现在主流互联网企业线上选用的仍是CMS(如Taobao、微店).

  CMS是一种以获取最短回收停顿时间为目标的收集器(CMS又称多并发低暂停的收集器), 基于”标记-清除”算法实现, 整个GC过程分为以下4个步骤:

  1. 初始标记(CMS initial mark)

  2. 并发标记(CMS concurrent mark: GC Roots Tracing过程)

  3. 重新标记(CMS remark)

  4. 并发清除(CMS concurrent sweep: 已死象将会就地释放, 注意: 此处没有压缩)

  其中两个加粗的步骤(初始标记、重新标记)仍需STW. 但初始标记仅只标记一下GC Roots能直接关联到的对象, 速度很快; 而重新标记则是为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录, 虽然一般比初始标记阶段稍长, 但要远小于并发标记时间.
  ![image](https://user-images.githubusercontent.com/7789698/38507565-772a1a74-3c4f-11e8-928a-b0f21e9edf59.png)

  (由于整个GC过程耗时最长的并发标记和并发清除阶段的GC线程可与用户线程一起工作, 所以总体上CMS的GC过程是与用户线程一起并发地执行的.

  由于CMS收集器将整个GC过程进行了更细粒度的划分, 因此可以实现并发收集、低停顿的优势, 但它也并非十分完美, 其存在缺点及解决策略如下:

  1. CMS默认启动的回收线程数=(CPU数目+3)4

     当CPU数>4时, GC线程最多占用不超过25%的CPU资源, 但是当CPU数<=4时, GC线程可能就会过多的占用用户CPU资源, 从而导致应用程序变慢, 总吞吐量降低.

  2. 无法处理浮动垃圾, 可能出现Promotion Failure、Concurrent Mode Failure而导致另一次Full GC的产生: 浮动垃圾是指在CMS并发清理阶段用户线程运行而产生的新垃圾. 由于在GC阶段用户线程还需运行, 因此还需要预留足够的内存空间给用户线程使用, 导致CMS不能像其他收集器那样等到老年代几乎填满了再进行收集. 因此CMS提供了-XX:CMSInitiatingOccupancyFraction参数来设置GC的触发百分比(以及-XX:+UseCMSInitiatingOccupancyOnly来启用该触发百分比), 当老年代的使用空间超过该比例后CMS就会被触发(JDK 1.6之后默认92%). 但当CMS运行期间预留的内存无法满足程序需要, 就会出现上述Promotion Failure等失败, 这时VM将启动后备预案: 临时启用Serial Old收集器来重新执行Full GC(CMS通常配合大内存使用, 一旦大内存转入串行的Serial GC, 那停顿的时间就是大家都不愿看到的了).

  3. 最后, 由于CMS采用”标记-清除”算法实现, 可能会产生大量内存碎片. 内存碎片过多可能会导致无法分配大对象而提前触发Full GC. 因此CMS提供了-XX:+UseCMSCompactAtFullCollection开关参数, 用于在Full GC后再执行一个碎片整理过程. 但内存整理是无法并发的, 内存碎片问题虽然没有了, 但停顿时间也因此变长了, 因此CMS还提供了另外一个参数-XX:CMSFullGCsBeforeCompaction用于设置在执行N次不进行内存整理的Full GC后, 跟着来一次带整理的(默认为0: 每次进入Full GC时都进行碎片整理).

- 分区收集- G1收集器

  - G1(Garbage-First)是一款面向服务端应用的收集器, 主要目标用于配备多颗CPU的服务器治理大内存.
  - G1 is planned as the long term replacement for the Concurrent Mark-Sweep Collector (CMS).
  - -XX:+UseG1GC 启用G1收集器.

与其他基于分代的收集器不同, G1将整个Java堆划分为多个大小相等的独立区域(Region), 虽然还保留有新生代和老年代的概念, 但新生代和老年代不再是物理隔离的了, 它们都是一部分Region(不需要连续)的集合.

![image](https://user-images.githubusercontent.com/7789698/38507575-7ccdcea8-3c4f-11e8-8356-c269b0a308ac.png)

每块区域既有可能属于O区、也有可能是Y区, 因此不需要一次就对整个老年代/新生代回收. 而是当线程并发寻找可回收的对象时, 有些区块包含可回收的对象要比其他区块多很多. 虽然在清理这些区块时G1仍然需要暂停应用线程, 但可以用相对较少的时间优先回收垃圾较多的Region(这也是G1命名的来源). 这种方式保证了G1可以在有限的时间内获取尽可能高的收集效率.





### 类加载器

把类加载阶段中“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放到java虚拟机外部去实现，以便让程序自己决定如何去获取所需要的类，实现这个动作的代码模块称为“类加载器”

1）类与类的加载器

比较两个类是否相等，只有在这两个类是由同一个类加载器加载的前提下才有意义，即使两个类来源于同一个Class文件，被同一个虚拟机加载，只要他们的类加载器不一样，那么这两个类必定不相等（equals（） isAssignableFrom（） isInstance（））

2）双亲委派模型

从java虚拟机的角度来讲，只存在两种不同的类加载器：一种是启动类加载器，是虚拟机的一部分，另一种是所有其他的类加载器，这些加载器由java语言实现，独立虚拟机之外，都继承抽象类java.lang.ClassLoader
类加载器可以分为以下几种

1.启动类加载器（Bootstrap ClassLoader）(存放在%JAVA_HOME%\lib或-XBootclasspath指定的)

2.扩展类加载器（Extension ClassLoader）（%JAVA_HOME%\lib\ext或java.ext.dirs）

3.应用程序类加载器（Application ClassLoader）：一般情况下这个是程序默认的类加载器

以下是类加载器的双亲委派模型

<img width="549" alt="1" src="https://user-images.githubusercontent.com/7789698/29862824-3b7c960e-8da0-11e7-905c-4397d2f7f8d4.png">



![f2c882b8b4618500c310a033ab23e493131f0533](https://user-images.githubusercontent.com/7789698/32713595-eef6db1c-c884-11e7-8fbe-7ad5a4f486c2.png)

![20151101162452934](https://user-images.githubusercontent.com/7789698/32713262-688b3222-c883-11e7-9fd5-83477c144d6d.jpg)




`public abstract class ClassLoader`

```java
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 在加载类之前先调用findLoadedClass方法检查该类是否已经被加载过，findLoadedClass会返回一个Class类型的对象，如果该类已经被加载过，那么就可以直接返回该对象
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // 如果以上两个步骤都没有成功的加载到类，那么调用自己的findClass(name)方法来加载类。
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
//链接指定的类。这个方法给Classloader用来链接一个类，如果这个类已经被链接过了，那么这个方法只做一个简单的返回。否则，这个类将被按照 Java™规范中的Execution描述进行链接……

                resolveClass(c);
            }
            return c;
        }
    }
```


```java
    protected Object getClassLoadingLock(String className) {
        Object lock = this;
        if (parallelLockMap != null) {
            Object newLock = new Object();
            lock = parallelLockMap.putIfAbsent(className, newLock);
            if (lock == null) {
                lock = newLock;
            }
        }
        return lock;
    }


    private ClassLoader(Void unused, ClassLoader parent) {
        this.parent = parent;
        if (ParallelLoaders.isRegistered(this.getClass())) {
            parallelLockMap = new ConcurrentHashMap<>();
            package2certs = new ConcurrentHashMap<>();
            domains =
                Collections.synchronizedSet(new HashSet<ProtectionDomain>());
            assertionLock = new Object();
        } else {
            // no finer-grained lock; lock on the classloader instance
            parallelLockMap = null;
            package2certs = new Hashtable<>();
            domains = new HashSet<>();
            assertionLock = this;
        }
    }

//封装了并行的可装载的类型的集合。
 private static class ParallelLoaders {
        private ParallelLoaders() {}

        // the set of parallel capable loader types
        private static final Set<Class<? extends ClassLoader>> loaderTypes =
            Collections.newSetFromMap(
                new WeakHashMap<Class<? extends ClassLoader>, Boolean>());
        static {
            synchronized (loaderTypes) { loaderTypes.add(ClassLoader.class); }
        }

        /**
         * Registers the given class loader type as parallel capabale.
         * Returns {@code true} is successfully registered; {@code false} if
         * loader's super class is not registered.
         */
        static boolean register(Class<? extends ClassLoader> c) {
            synchronized (loaderTypes) {
                if (loaderTypes.contains(c.getSuperclass())) {
                    // register the class loader as parallel capable
                    // if and only if all of its super classes are.
                    // Note: given current classloading sequence, if
                    // the immediate super class is parallel capable,
                    // all the super classes higher up must be too.
                    loaderTypes.add(c);
                    return true;
                } else {
                    return false;
                }
            }
        }

        /**
         * Returns {@code true} if the given class loader type is
         * registered as parallel capable.
         */
        static boolean isRegistered(Class<? extends ClassLoader> c) {
            synchronized (loaderTypes) {
                return loaderTypes.contains(c);
            }
        }
    }
```

## 类装载方式:
1. 隐式装载， 程序在运行过程中当碰到通过new 等方式生成对象时，隐式调用类装载器加载对应的类到jvm中。 
2. 显式装载， 通过class.forname()等方法，显式加载需要的类


## java内存模型

在程序运行中，会将运行所需要的数据复制一份到CPU高速缓存中，在进行运算时CPU不再也主存打交道，而是直接从高速缓存中读写数据，只有当运行结束后才会将数据刷新到主存中。

所有实例域、静态域和数组元素存储在堆内存中，堆内存在线程之间共享。局部变量（Local variables），方法定义参数（java语言规范称之为formal method parameters）和异常处理器参数（exception handler parameters）不会在线程之间共享，它们不会有内存可见性问题，也不受内存模型的影响。

Java线程之间的通信由Java内存模型（本文简称为JMM）控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（main memory）中，每个线程都有一个私有的本地内存（local memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化。Java内存模型的抽象示意图如下：

![image](https://user-images.githubusercontent.com/7789698/42419093-c0eebcaa-82e0-11e8-8a56-7b87f18304a2.png)

从上图来看，线程A与线程B之间如要通信的话，必须要经历下面2个步骤：

1. 首先，线程A把本地内存A中更新过的共享变量刷新到主内存中去。
2. 然后，线程B到主内存中去读取线程A之前已更新过的共享变量。

下面通过示意图来说明这两个步骤：

![image](https://user-images.githubusercontent.com/7789698/42419102-d90a5ede-82e0-11e8-9e99-30c0806cc4e4.png)

如上图所示，本地内存A和B有主内存中共享变量x的副本。假设初始时，这三个内存中的x值都为0。线程A在执行时，把更新后的x值（假设值为1）临时存放在自己的本地内存A中。当线程A和线程B需要通信时，线程A首先会把自己本地内存中修改后的x值刷新到主内存中，此时主内存中的x值变为了1。随后，线程B到主内存中去读取线程A更新后的x值，此时线程B的本地内存的x值也变为了1。

从整体来看，这两个步骤实质上是线程A在向线程B发送消息，而且这个通信过程必须要经过主内存。JMM通过控制主内存与每个线程的本地内存之间的交互，来为java程序员提供内存可见性保证。

### 重排序

在执行程序时为了提高性能，编译器和处理器常常会对指令做重排序。重排序分三类：

1、编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。

2、指令级并行的重排序。现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

3、内存系统的重排序。由于处理器使用缓存和读／写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

从 Java 源代码到最终实际执行的指令序列，会分别经历下面三种重排序：

![image](https://user-images.githubusercontent.com/7789698/42419111-fe27d1b0-82e0-11e8-9c4d-4732bcae7690.png)

上面的这些重排序都可能导致多线程程序出现内存可见性问题。对于编译器，JMM 的编译器重排序规则会禁止特定类型的编译器重排序（不是所有的编译器重排序都要禁止）。对于处理器重排序，JMM 的处理器重排序规则会要求 Java 编译器在生成指令序列时，插入特定类型的内存屏障指令，通过内存屏障指令来禁止特定类型的处理器重排序（不是所有的处理器重排序都要禁止）。

JMM 属于语言级的内存模型，它确保在不同的编译器和不同的处理器平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。

### 处理器重排序

现代的处理器使用写缓冲区来临时保存向内存写入的数据。写缓冲区可以保证指令流水线持续运行，它可以避免由于处理器停顿下来等待向内存写入数据而产生的延迟。同时，通过以批处理的方式刷新写缓冲区，以及合并写缓冲区中对同一内存地址的多次写，可以减少对内存总线的占用。虽然写缓冲区有这么多好处，但每个处理器上的写缓冲区，仅仅对它所在的处理器可见。这个特性会对内存操作的执行顺序产生重要的影响：处理器对内存的读/写操作的执行顺序，不一定与内存实际发生的读/写操作顺序一致！

举个例子：

| Processor A                                            | Processor B            |
| ------------------------------------------------------ | ---------------------- |
| a = 1; //A1x = b; //A2                                 | b = 2; //B1y = a; //B2 |
| 初始状态：a = b = 0处理器允许执行后得到结果：x = y = 0 |                        |

假设处理器A和处理器B按程序的顺序并行执行内存访问，最终却可能得到 x = y = 0。具体的原因如下图所示：

![image](https://user-images.githubusercontent.com/7789698/42419136-690ede1a-82e1-11e8-92f7-617e9179cd1f.png)

处理器 A 和 B 同时把共享变量写入在写缓冲区中（A1、B1），然后再从内存中读取另一个共享变量（A2、B2），最后才把自己写缓冲区中保存的脏数据刷新到内存中（A3、B3）。当以这种时序执行时，程序就可以得到 x = y = 0 的结果。

从内存操作实际发生的顺序来看，直到处理器 A 执行 A3 来刷新自己的写缓存区，写操作 A1 才算真正执行了。虽然处理器 A 执行内存操作的顺序为：A1 -> A2，但内存操作实际发生的顺序却是：A2 -> A1。此时，处理器 A 的内存操作顺序被重排序了。

这里的关键是，由于写缓冲区仅对自己的处理器可见，它会导致处理器执行内存操作的顺序可能会与内存实际的操作执行顺序不一致。由于现代的处理器都会使用写缓冲区，因此现代的处理器都会允许对写-读操作重排序。

### 内存屏障指令 

为了保证内存可见性，Java 编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM 把内存屏障指令分为下列四类：

![image](https://user-images.githubusercontent.com/7789698/42419147-8bd36902-82e1-11e8-834d-dc4bfdcc8309.png)

### final

如果一个类包含final字段，且在构造函数中初始化，那么**正确的构造一个对象后**，final字段被设置后对于其它线程是可见的。一个类被**final**修饰后，它的方法默认被修饰为**final** ，这时方法的内联起到作用了。

对于 final 域，编译器和处理器要遵守两个重排序规则：

1.在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

2.初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

#### 写 final 域的重排序规则

写 final 域的重排序规则禁止把 final 域的写重排序到构造函数之外。这个规则的实现包含下面2个方面：

- JMM 禁止编译器把 final 域的写重排序到构造函数之外。
- 编译器会在 final 域的写之后，构造函数 return 之前，插入一个StoreStore屏障。这个屏障禁止处理器把 final 域的写重排序到构造函数之外。

#### 读 final 域的重排序规则

在一个线程中，初次读对象引用与初次读该对象包含的 final 域，JMM 禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）。编译器会在读 final 域操作的前面插入一个 LoadLoad 屏障。

#### final 域是引用类型

对于引用类型，写 final 域的重排序规则对编译器和处理器增加了如下约束：

在构造函数内对一个 final 引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

### Synchronization 

> synchronized可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性

Java中每一个对象都可以作为锁，这是synchronized实现同步的基础：

1. 普通同步方法，锁是当前实例对象
2. 静态同步方法，锁是当前类的class对象
3. 同步方法块，锁是括号里面的对象

- 同步代码块：monitorenter指令插入到同步代码块的开始位置，monitorexit指令插入到同步代码块的结束位置，JVM需要保证每一个monitorenter都有一个monitorexit与之相对应。任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁； 
- 同步方法：synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象。(摘自：http://www.cnblogs.com/javaminer/p/3889023.html)

### volatile

在程序运行中，会将运行所需要的数据复制一份到CPU高速缓存中，在进行运算时CPU不再也主存打交道，而是直接从高速缓存中读写数据，只有当运行结束后才会将数据刷新到主存中。

解决缓存一致性方案有两种（volatile根据处理器不同可能用下面两种其一去实现可见性）：

1. 通过在总线加LOCK#锁的方式
2. 通过缓存一致性协议

方案1 它是采用一种独占的方式来实现的，即总线加LOCK#锁的话，只能有一个CPU能够运行，其他CPU都得阻塞，效率较为低下。

方案2 缓存一致性协议（MESI协议）它确保每个缓存中使用的共享变量的副本是一致的。其核心思想如下：当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。

原子性  即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。 

可见性 当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。 

有序性 程序执行的顺序按照代码的先后顺序执行。

volatile可以保证线程可见性且提供了一定的有序性，无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。

### 阻止编译器重排序

![image](https://user-images.githubusercontent.com/7789698/42420286-f670a7de-82f5-11e8-8896-e0cd5b7e77c4.png)

### 阻止处理器重排序

内存屏障（[memory barrier](http://en.wikipedia.org/wiki/Memory_barrier)）是组个CPU指令。编译器和CPU可以在保证输出结果一样的情况下对指令重排序，使性能得到优化。插入一个内存屏障，相当于告诉CPU和编译器先于这个命令的必须先执行，后于这个命令的必须后执行。

内存屏障分为以下4个：

1. LoadLoad屏障（Load1，LoadLoad， Load2）
   确保Load1所要读入的数据能够在被Load2和后续的load指令访问前读入。通常能执行预加载指令或/和支持乱序处理的处理器中需要显式声明Loadload屏障，因为在这些处理器中正在等待的加载指令能够绕过正在等待存储的指令。 而对于总是能保证处理顺序的处理器上，设置该屏障相当于无操作。
2. LoadStore屏障（Load1，LoadStore， Store2）
   确保Load1的数据在Store2和后续Store指令被刷新之前读取。在等待Store指令可以越过loads指令的乱序处理器上需要使用LoadStore屏障。
3. StoreStore屏障（Store1，StoreStore，Store2）
   确保Store1的数据在Store2以及后续Store指令操作相关数据之前对其它处理器可见（例如向主存刷新数据）。通常情况下，如果处理器不能保证从写缓冲或/和缓存向其它处理器和主存中按顺序刷新数据，那么它需要使用StoreStore屏障。
4. StoreLoad屏障（Store1，StoreLoad，Load2）
   确保Store1的数据在被Load2和后续的Load指令读取之前对其他处理器可见。StoreLoad屏障可以防止一个后续的load指令 不正确的使用了Store1的数据，而不是另一个处理器在相同内存位置写入一个新数据。

![image](https://user-images.githubusercontent.com/7789698/42420294-1427cc94-82f6-11e8-8f01-5c0bdecf5f43.png)




## 线程池

### ThreadPoolExecutor

![image](https://user-images.githubusercontent.com/7789698/38788998-dba9647c-4169-11e8-8f08-410fed67f06e.png)

线程池类为 java.util.concurrent.ThreadPoolExecutor，常用构造方法为：

`ThreadPoolExecutor(int corePoolSize, int maximumPoolSize,`
`long keepAliveTime, TimeUnit unit,`
`BlockingQueue<Runnable> workQueue,`
`RejectedExecutionHandler handler)`
- corePoolSize： 线程池维护线程的最少数量
- maximumPoolSize：线程池维护线程的最大数量
- keepAliveTime： 线程池维护线程所允许的空闲时间
- unit： 线程池维护线程所允许的空闲时间的单位
- workQueue： 线程池所使用的缓冲队列
- handler： 线程池对拒绝任务的处理策略

  一个任务通过 execute(Runnable)方法被添加到线程池，任务就是一个 Runnable类型的对象，任务的执行方法就是Runnable类型对象的run()方法。

当一个任务通过execute(Runnable)方法欲添加到线程池时：
1. 如果此时线程池中的数量小于corePoolSize，即使线程池中的线程都处于空闲状态，也要创建新的线程来处理被添加的任务。
2. 如果此时线程池中的数量等于 corePoolSize，但是缓冲队列 workQueue未满，那么任务被放入缓冲队列。
3. 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量小于maximumPoolSize，建新的线程来处理被添加的任务。
4. 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量等于maximumPoolSize，那么通过 handler所指定的策略来处理此任务。也就是：处理任务的优先级为：核心线程corePoolSize、任务队列workQueue、最大线程maximumPoolSize，如果三者都满了，使用handler处理被拒绝的任务。
5. 当线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止。这样，线程池可以动态的调整池中的线程数。

handler有四个选择：

```java
ThreadPoolExecutor.AbortPolicy()
```

抛出java.util.concurrent.RejectedExecutionException异常


```java
ThreadPoolExecutor.CallerRunsPolicy()
```

当抛出RejectedExecutionException异常时，会调用rejectedExecution方法
(如果主线程没有关闭，则主线程调用run方法,源码如下

``` java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
)
```


```
ThreadPoolExecutor.DiscardOldestPolicy()
```

抛弃旧的任务


```
ThreadPoolExecutor.DiscardPolicy()
```

抛弃当前的任务



#### Executor

```java
public interface Executor {
    void execute(Runnable command);
}
```

#### ExecutorService

```java
public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

- ExecutorService                             真正的线程池接口。
- ScheduledExecutorService             能和Timer/TimerTask类似，解决那些需要任务重复执行的问题。
- ThreadPoolExecutor                       ExecutorService的默认实现。
- ScheduledThreadPoolExecutor       继承ThreadPoolExecutor的ScheduledExecutorService接口实现，周期性任务调度的类实现。

### newFixedThreadPool

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

```

很明显，这个线程池内部每个Runnable是用LinkedBlockingQueue管理的。创建一个最大线程数目固定的线程池，该线程池用一个共享的无界队列来存储提交的任务。参数nThreads指定线程池的最大线程数，参数threadFactory是线程工厂类，主要用于自定义线程池中创建新线程时的行为。需要说明的是，创建线程池时，如果线程池没有接收到任何任务，则线程池中不会创建新线程，在线程池中线程数目少于最大线程数时，每来一个新任务就创建一个新线程，当线程数达到最大线程数时，不再创建新线程，新来的任务存储在队列中，之后线程数目不再变化！使用如下：
（java8）

```java
//实现Callable
public static int longOperation() {
    System.out.println("Running on thread #"+ Thread.currentThread().getId());
    // [...]
    return 42;
}
```

```java
ExecutorService executorService = Executors.newFixedThreadPool(10);
Future[] answers = {
    executorService.submit(() -> longOperation()),
    executorService.submit(ThreadGoodies::longOperation)
};
Arrays.stream(answers).forEach(Unchecked.consumer(
    f -> System.out.println(f.get())
));
```

jdk8以前

```java
ExecutorService executorService = Executors.newFixedThreadPool(10);
    executorService.execute(t1);
    executorService.execute(t2);
        pool.shutdown();
```

这部分：http://www.tuicool.com/articles/2iI7b23
###  newWorkStealingPool（JDK7引入）

parallelism表示并行数

```java
    public static ExecutorService newWorkStealingPool(int parallelism) {
        return new ForkJoinPool
            (parallelism,
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```

```java
      TreeNode tree = new TreeNode(5,
                new TreeNode(3), new TreeNode(2,
                new TreeNode(2), new TreeNode(8)));

        ForkJoinPool forkJoinPool = ForkJoinPool.commonPool();
        int sum = forkJoinPool.invoke(new CountingTask(tree));
```

创建ForkJoin框架中用到的ForkJoinPool线程池。ForkJoinPool实现了工作窃取算法（work-stealing），线程会主动寻找新创建的任务去执行，从而保证较高的线程利用率。它使用守护线程（deamon）来执行任务，因此无需对他显示的调用shutdown()来关闭。
https://www.ibm.com/developerworks/cn/java/j-lo-forkjoin/
###  newScheduledThreadPool

```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());
ScheduledFuture<?> future = executor.schedule(task, 3, TimeUnit.SECONDS);

TimeUnit.MILLISECONDS.sleep(1337);

long remainingDelay = future.getDelay(TimeUnit.MILLISECONDS);
System.out.printf("Remaining Delay: %sms", remainingDelay);
```

```java
        CountDownLatch lock = new CountDownLatch(3);

        ScheduledExecutorService executor = Executors.newScheduledThreadPool(5);
        ScheduledFuture<?> future = executor.scheduleAtFixedRate(() -> {
            System.out.println("Hello World");
            lock.countDown();
        }, 500, 100, TimeUnit.MILLISECONDS);

        lock.await();
        future.cancel(true);
```



tasks ：每秒的任务数，假设为500～1000
taskcost：每个任务花费时间，假设为0.1s
responsetime：系统允许容忍的最大响应时间，假设为1s
做几个计算
corePoolSize = 每秒需要多少个线程处理？ 
threadcount = tasks/(1/taskcost) =tasks*taskcout =  (500～1000)*0.1 = 50～100 个线程。corePoolSize设置应该大于50
根据8020原则，如果80%的每秒任务数小于800，那么corePoolSize设置为80即可
queueCapacity = (coreSizePool/taskcost)*responsetime
计算可得 queueCapacity = 80/0.1*1 = 80。意思是队列里的线程可以等待1s，超过了的需要新开线程来执行
切记不能设置为Integer.MAX_VALUE，这样队列会很大，线程数只会保持在corePoolSize大小，当任务陡增时，不能新开线程来执行，响应时间会随之陡增。
maxPoolSize = (max(tasks)- queueCapacity)/(1/taskcost)
计算可得 maxPoolSize = (1000-80)/10 = 92
（最大任务数-队列容量）/每个线程每秒处理能力 = 最大线程数
rejectedExecutionHandler：根据具体情况来决定，任务不重要可丢弃，任务重要则要利用一些缓冲机制来处理  （https://my.oschina.net/u/169390/blog/97415）
keepAliveTime和allowCoreThreadTimeout采用默认通常能满足



## 可重入性

​    可 重入（reentrant）函数可以由多于一个任务并发使用，而不必担心数据错误。相反，不可重入（non-reentrant）函数不能由超过一个任务所共享，除非能确保函数的互斥（或者使用信号量，或者在代码的关键部分禁用中断）。可重入函数可以在任意时刻被中断，稍后再继续运行，不会丢失数据。可重入函数要么使用本地变量，要么在使用全局变量时保护自己的数据。



## 锁优化

### 自旋锁

让该线程等待一段时间，不会被立即挂起，看持有锁的线程是否会很快释放锁。怎么等待呢？执行一段无意义的循环即可（自旋）。 

自旋锁在JDK 1.4.2中引入，默认关闭，但是可以使用-XX:+UseSpinning开开启，在JDK1.6中默认开启。同时自旋的默认次数为10次，可以通过参数-XX:PreBlockSpin来调整； 

### 适应自旋锁

如果通过参数-XX:preBlockSpin来调整自旋锁的自旋次数，会带来诸多不便。假如我将参数调整为10，但是系统很多线程都是等你刚刚退出的时候就释放了锁（假如你多自旋一两次就可以获取锁）。线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。 

### 锁消除

为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制，但是在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。锁消除的依据是逃逸分析的数据支持。 

### 锁粗化

将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。

### 轻量级锁

减少传统的重量级锁使用操作系统互斥量产生的性能消耗。当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁。

### 偏向锁

为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径。

### 重量级锁

重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。

![image](https://user-images.githubusercontent.com/7789698/38798131-45870b1e-4193-11e8-830e-72585851526d.png)

###  Little-Endian

低位字节排放在内存的低地址端，高位字节排放在内存的高地址端。

```
低地址 ------------------> 高地址
0x78  |  0x56  |  0x34  |  0x12
```

### Big-Endian

高位字节排放在内存的低地址端，低位字节排放在内存的高地址端。

```
低地址 -----------------> 高地址
0x12  |  0x34  |  0x56  |  0x78
```



参考：

https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247483784&idx=1&sn=672cd788380b2096a7e60aae8739d264&chksm=fa497e39cd3ef72fcafe7e9bcc21add3dce0d47019ab6e31a775ba7a7e4adcb580d4b51021a9&scene=21#wechat_redirect

https://www.jianshu.com/p/f68d6ef2dcf0

https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247483775&idx=1&sn=e3c249e55dc25f323d3922d215e17999&chksm=fa497ececd3ef7d82a9ce86d6ca47353acd45d7d1cb296823267108a06fbdaf71773f576a644&scene=21#wechat_redirect