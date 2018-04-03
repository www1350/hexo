---
title: Java跟踪利器
date: 2018-04-03 22:41:05
tags:
categories:
---

# Java跟踪利器---BTrace
地址：https://github.com/btraceio/btrace

> BTrace 是基于动态字节码修改技术(Hotswap)来实现运行时 java 程序的跟踪和替换。大体的原理可以用下面的公式描述：Client(Java compile api + attach api) + Agent（脚本解析引擎 + ASM + JDK6 Instumentation） + Socket其实 BTrace 就是使用了 java attach api 附加 agent.jar ，然后使用脚本解析引擎+asm来重写指定类的字节码，再使用 instrument 实现对原有类的替换。

# Btrace可以做什么？
1. 接口性能变慢，分析每个方法的耗时情况；
2. 当在Map中插入大量数据，分析其扩容情况；
3. 分析哪个方法调用了System.gc()，调用栈如何；
4. 执行某个方法抛出异常时，分析运行时参数；
5. ….

## BTrace重要概念与局限性
虽然BTrace很强大,但Btrace脚本就是一个普通的用@Btrace注解的Java类,其中包含一个或多个public static void修饰的方法,为了保证对目标程序不造成影响,Btrace脚本对其可以执行的动作做了很多限制

1. 不能创建对象
2. 不能抛出或者捕获异常
3. 不能用synchronized关键字
4. 不能对目标程序中的instace或者static变量
5. 不能调用目标程序的instance或者static方法
6. 脚本的field、method都必须是static的
7. 脚本不能包括outer,inner,nested class
8. 脚本中不能有循环,不能继承任何类,任何接口与assert语句


## 1.安装
https://github.com/btraceio/btrace/releases/tag/v1.3.10.1 下载到包解压

## 2.配置环境
```
export JAVA_HOME=/home/wenwei/jdk1.8.0_111
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib 
export PATH=${JAVA_HOME}/bin:$PATH
export BTRACE_HOME=/home/wenwei/btrace
export PATH=$PATH:$BTRACE_HOME/bin
```
## 3.预编译：
执行之前可以用预编译命令检查脚本的正确性，预编译命令为 btracec，它是一个 javac-like 命令

`btracec JStack.java`

## 4.使用BTrace
1.
` btrace [-I <include-path>] [-p <port>] [-cp <classpath>] <pid> <btrace-script> [<args>]`
- -I:没有这个表明跳过预编译
- include-path:指定用来编译脚本的头文件路径(关于预编译可参考例子ThreadBean.java)
- port:btrace agent端口,默认是2020
- classpath:编译所需类路径,一般是指btrace-client.jar等类所在路径
- pid:java进程id
- btrace-script:btrace脚本可以是.java文件,也可以是.class文件
- args:传递给btrace脚本的参数, 在脚本中可以通过$(), $length()来获取这些参数(定义在BTraceUtils中)

例如：
`btrace -cp lib/servlet-api.jar -p 2021 53523 JStack.java`

2.
`java -javaagent:btrace-agent.jar=[<agent-arg>[,<agent-arg>]*]? <launch-args>`

参数：
- noServer - don't start the socket server
- bootClassPath - boot classpath to be used
- systemClassPath - system classpath to be used
- debug - turns on verbose debug messages (true/false)
- unsafe - do not check for btrace restrictions violations (true/false)
- dumpClasses - dump the transformed bytecode to files (true/false)
- dumpDir - specifies the folder where the transformed classes will be dumped to
- stdout - redirect the btrace output to stdout instead of writing it to an arbitrary file (true/false)
- probeDescPath - the path to search for probe descriptor XMLs
- startupRetransform - enable retransform of all the loaded classes at attach (true/false)
- scriptdir - the path to a directory containing scripts to be run at the agent startup
- scriptOutputFile - the path to a file the btrace agent will store its output
- script - colon separated list of tracing scripts to be run at the agent startup


## 5.脚本
```
package com.sun.btrace.samples;

import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.Sys.*;
import static com.sun.btrace.BTraceUtils.Threads.*;

/*
 * A simple sample prints stack traces and exits. This
 * BTrace program mimics the jstack command line tool in JDK.
 */
@BTrace
public class JStack {
    static {
        deadlocks(false);
        jstackAll();
        exit(0);
    }
}
```

## 6.参数说明
- @OnMethod:指定使用当前注解的方法应该在什么情况下触发,claszz属性指定要匹配的类的全限定类名,可以用正则表达式:/类名的Pattern/匹配,用”+类名”匹配所有子类,用”@某某注解”匹配用该注解注解过的类method属性指定要匹配的方法名称,可以用正则表达式:/方法名称的Pattern/匹配type属性:void(java.lang.String)可以用于匹配:public void funcName(String param) throws Exception,location属性用@Location来表明,匹配了clazz,method情况,在方法执行的何时去执行脚本(前,后,异常,行,某个方法调用)
- @OnTimer:指定一个定时任务
- @OnExit:当脚本运行Sys.exit(code)时触发
- @OnError:当脚本运行抛出异常时触发
- @OnEvent:脚本运行时Ctrl+C可以发送事件
- @OnLowMemory:让你指定一个阀值,内存低于阀值触发
- @OnProbe:可以用一个xml文件来描述你想在什么时候触发该方法


- @Self:目标对象本身
- @Retrun:目标程序方法返回值(Kind.RETURN)
- @ProbeClassName:目标类名
- @ProbeMethodName:目标方法名
- @targetInstance:@Location指定的clazz,method的目标(Kind.CALL)
- @targetMethodOrField:@Location指定的clazz,method的目标的方法或字段(Kind.CALL)
- @Duration:目标方法执行时间,单位是纳秒,需要与 Kind.RETURN 或者 Kind.ERROR 一起使用


### 正则表达式定位
可以用表达式，批量定义需要监控的类与方法。正则表达式需要写在两个 "/" 中间。

下例监控javax.swing下的所有类的所有方法....可能会非常慢，建议范围还是窄些。
```
@OnMethod(clazz="/javax\\.swing\\..*/", method="/.*/")
public static void swingMethods( @ProbeClassName String probeClass, @ProbeMethodName String probeMethod) {
   print("entered " + probeClass + "."  + probeMethod);
}
```
通过在拦截函数的定义里注入@ProbeClassName String probeClass, @ProbeMethodName String probeMethod 参数，告诉脚本实际匹配到的类和方法名。

另一个例子，监控Statement的executeUpdate(), executeQuery() 和 executeBatch() 三个方法，见JdbcQueries.java

 

###  按接口，父类，Annotation定位
比如我想匹配所有的Filter类，在接口或基类的名称前面，加个+ 就行
`@OnMethod(clazz="+com.vip.demo.Filter", method="doFilter")`

也可以按类或方法上的annotaiton匹配，前面加上@就行
`@OnMethod(clazz="@javax.jws.WebService", method="@javax.jws.WebMethod")`

 

### 其他
1. 构造函数的名字是 <init>
  `@OnMethod(clazz="java.net.ServerSocket", method="<init>")`

2. 静态内部类的写法，是在类与内部类之间加上"$"
  `@OnMethod(clazz="com.vip.MyServer$MyInnerClass", method="hello")`

3. 如果有多个同名的函数，想区分开来，可以在拦截函数上定义不同的参数列表（见4.1）。


### 拦截时机
可以为同一个函数的不同的Location，分别定义多个拦截函数。

###  Kind.Entry与Kind.Return
`@OnMethod( clazz="java.net.ServerSocket", method="bind" )`
不写Location，默认就是刚进入函数的时候(Kind.ENTRY)。

但如果你想获得函数的返回结果或执行时间，则必须把切入点定在返回(Kind.RETURN)时。

```
OnMethod(clazz = "java.net.ServerSocket", method = "getLocalPort", location = @Location(Kind.RETURN))

public static void onGetPort(@Return int port, @Duration long duration)
```
duration的单位是纳秒，要除以 1,000,000 才是毫秒。


### Kind.Error, Kind.Throw和 Kind.Catch
异常抛出(Throw)，异常被捕获(Catch)，异常没被捕获被抛出函数之外(Error)，主要用于对某些异常情况的跟踪。

在拦截函数的参数定义里注入一个Throwable的参数，代表异常。

```
@OnMethod(clazz = "java.net.ServerSocket", method = "bind", location = @Location(Kind.ERROR))

public static void onBind(Throwable exception, @Duration long duration)

```




###  Kind.Call与Kind.Line
下例定义监控bind()函数里调用的所有其他函数：
```
@OnMethod(clazz = "java.net.ServerSocket", method = "bind", location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))

public static void onBind(@Self Object self, @TargetInstance Object instance, @TargetMethodOrField String method, @Duration long duration)
```
所调用的类及方法名所注入到@TargetInstance与 @TargetMethodOrField中。

​静态函数中，instance的值为空。如果想获得执行时间，必须把Where定义成AFTER。
如果想获得执行时间，必须 把Where定义成AFTER。

注意这里，一定不要像下面这样大范围的匹配，否则这性能是神仙也没法救了：

`@OnMethod(clazz = "/javax\\.swing\\..*/", method = "/.*/", location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/"))
`
下例监控代码是否到达了Socket类的第363行。

```
@OnMethod(clazz = "java.net.ServerSocket", location = @Location(value = Kind.LINE, line = 363))
public static void onBind4() {
   println("socket bind reach line:363");
}
```

line还可以为-1，然后每行都会打印出来，加参数int line 获得的当前行数。此时会显示函数里完整的执行路径，但肯定又非常慢。

### 打印this，参数 与 返回值
###  定义注入
```
import com.sun.btrace.AnyType;
@OnMethod(clazz = "java.io.File", method = "createTempFile", location = @Location(value = Kind.RETURN))
public static void o(@Self Object self, String prefix, String suffix, @Return AnyType result)
```
如果想打印它们，首先按顺序定义用@Self 注释的this， 完整的参数列表，以及用@Return 注释的返回值。

需要打印哪个就定义哪个，不需要的就不要定义。但定义一定要按顺序，比如参数列表不能跑到返回值的后面。

Self：
如果是静态函数， self为空。

前面提到，如果上述使用了非JDK的类，命令行里要指定classpath。不过，如前所述，因为BTrace里不允许调用类的方法，所以定义具体类很多时候也没意思，所以self定义为Object就够了。

参数：
参数数列表要么不要定义，要定义就要定义完整，否则BTrace无法处理不同参数的同名函数。

如果有些参数你实在不想引入非JDK类，又不会造成同名函数不可区分，可以用AnyType来定义（不能用Object）。

如果拦截点用正则表达式中匹配了多个函数，函数之间的参数个数不一样，你又还是想把参数打印出来时，可以用AnyType[] args来定义。

但不知道是不是当前版本的bug，AnyType[] args 不能和 location＝Kind.RETURN 同用，否则会进入一种奇怪的静默状态，只要有一个函数定义错了，整个Btrace就什么都打印不出来。

结果：
同理，结果也可以用AnyType来定义，特别是用正则表达式匹配多个函数的时候，连void都可以表示。

 

###  打印
再次强调，为了保证性能不受影响，Btrace不允许调用任何实例方法。
比如不能调用getter方法（怕在getter里有复杂的计算），只会通过直接反射来读取属性名。
又比如，除了JDK类，其他类toString时只会打印其类名＋System.IdentityHashCode。
println, printArray，都按上面的规律进行，所以只能打打基本类型。

如果想打印一个Object的属性，用printFields()来反射。

如果只想反射某个属性，参照下面打印Port属性的写法。从性能考虑，应把field用静态变量缓存起来。

注意JDK类与非JDK类的区别：

```
import java.lang.reflect.Field;

//JDK的类这样写就行
private static Field fdFiled = field("java.io,FileInputStream", "fd");

//非JDK的类，要给出ClassLoader，否则ClassNotFound
private static Field portField = field(classForName("com.vip.demo.MyObject", contextClassLoader()), "port");

public static void onChannelRead(@Self Object self) {
    println("port:" + getInt(portField, self));
}
```




### TLS，拦截函数间的通信机制
如果要多个拦截函数之间要通信，可以使用@TLS定义 ThreadLocal的变量来共享
```
@TLS
private static int port = -1;

@OnMethod(clazz = "java.net.ServerSocket", method = "<init>")
public static void onServerSocket(int p){
    port = p;
}

@OnMethod(clazz = "java.net.ServerSocket", method = "bind")
public static void onBind(){
  println("server socket at " + port);
}
```


## 典型场景
###  打印慢调用
下例打印所有用时超过1毫秒的filter。
```
@OnMethod(clazz = "+com.vip.demo.Filter", method = "doFilter", location = @Location(Kind.RETURN))
public static void onDoFilter2(@ProbeClassName String pcn,  @Duration long duration) {
    if (duration > 1000000) {
        println(pcn + ",duration:" + (duration / 100000));
    }
}
```

最好能抽取了打印耗时的函数，减少代码重复度。
定位到某一个Filter慢了之后，可以直接用Location(Kind.CALL)，进一步找出它里面的哪一步慢了。

 

###  谁调用了这个函数
比如，谁调用了System.gc() ?
```
@OnMethod(clazz = "java.lang.System", method = "gc")
public static void onSystemGC() {
    println("entered System.gc()");
    jstack();
}
```


###  捕捉异常，或进入了某个特定代码行时，this对象及参数的值
按之前的提示，自己组合一下即可。


###  打印函数的调用/慢调用的统计信息
如果你已经看到了这里，那基本也不用我再啰嗦了，自己看Samples的Histogram.java, HistoOnEvent.java
可以用AtomicInteger构造计数器，然后定时(@OnTimer)，或根据事件(@OnEvent)输出结果(ctrl+c后选择发送事件)。
