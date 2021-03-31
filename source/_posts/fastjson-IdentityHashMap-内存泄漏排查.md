---
title: fastjson IdentityHashMap 内存泄漏排查
abbrlink: bc931ae8
date: 2018-08-17 13:39:23
tags:
categories:
---

一个安稳的周末，突然线上传来报警，保留现场过后紧急重启下，然后开始分析。让运维把oom 的dump数据和jstack数据传来



dump文件太大，传过来之前先分析下jstack日志。

jstack发现了一丝异样

> "http-nio-8080-exec-197" #7490 daemon prio=5 os_prio=0 tid=0x00007fdd5806b000 nid=0xed1 waiting for monitor entry [0x00007fdd1b7d5000]
>    java.lang.Thread.State: BLOCKED (on object monitor)
> 	at org.apache.catalina.webresources.CachedResource.validateResources(CachedResource.java:125)
> 	- waiting to lock <0x000000008015c660> (a org.apache.catalina.webresources.CachedResource)
> 	at org.apache.catalina.webresources.Cache.getResources(Cache.java:129)
> 	at org.apache.catalina.webresources.StandardRoot.getResources(StandardRoot.java:315)
> 	at org.apache.catalina.webresources.StandardRoot.getClassLoaderResources(StandardRoot.java:231)
> 	at org.apache.catalina.loader.WebappClassLoaderBase.findResources(WebappClassLoaderBase.java:995)
> 	at java.lang.ClassLoader.getResources(ClassLoader.java:1142)
> 	at com.alibaba.fastjson.util.ServiceLoader.load(ServiceLoader.java:33)
> 	at com.alibaba.fastjson.parser.ParserConfig.getDeserializer(ParserConfig.java:459)
> 	at com.alibaba.fastjson.parser.ParserConfig.getDeserializer(ParserConfig.java:354)
> 	at com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:639)
> 	at com.alibaba.fastjson.JSON.parseObject(JSON.java:350)
> 	at com.alibaba.fastjson.JSON.parseObject(JSON.java:318)
> 	at com.alibaba.fastjson.JSON.parseObject(JSON.java:281)

看到这个线程是阻塞状态，也就是tomcat请求http-nio-8080-exec-197现在是阻塞状态，等待<0x000000008015c660>释放。而且waiting to lock <0x000000008015c660>出现了11次，也就是有11个线程正在等待释放

关于线程状态：

![image](https://user-images.githubusercontent.com/7789698/44249880-86ddb800-a224-11e8-9e28-084e8bfebbb9.png)

**新建（New）**

创建后尚未启动。

**可运行（Runnable）**

可能正在运行，也可能正在等待 CPU 时间片。

包含了操作系统线程状态中的 Running 和 Ready。

**阻塞（Blocking）**

等待获取一个排它锁，如果其线程释放了锁就会结束此状态。

**无限期等待（Waiting）**

等待其它线程显式地唤醒，否则不会被分配 CPU 时间片。

| 进入方法                                   | 退出方法                             |
| ------------------------------------------ | ------------------------------------ |
| 没有设置 Timeout 参数的 Object.wait() 方法 | Object.notify() / Object.notifyAll() |
| 没有设置 Timeout 参数的 Thread.join() 方法 | 被调用的线程执行完毕                 |
| LockSupport.park() 方法                    | -                                    |

**限期等待（Timed Waiting）**

无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒。

调用 Thread.sleep() 方法使线程进入限期等待状态时，常常用“使一个线程睡眠”进行描述。

调用 Object.wait() 方法使线程进入限期等待或者无限期等待时，常常用“挂起一个线程”进行描述。

睡眠和挂起是用来描述行为，而阻塞和等待用来描述状态。

阻塞和等待的区别在于，阻塞是被动的，它是在等待获取一个排它锁。而等待是主动的，通过调用 Thread.sleep() 和 Object.wait() 等方法进入。

| 进入方法                                 | 退出方法                                        |
| ---------------------------------------- | ----------------------------------------------- |
| Thread.sleep() 方法                      | 时间结束                                        |
| 设置了 Timeout 参数的 Object.wait() 方法 | 时间结束 / Object.notify() / Object.notifyAll() |
| 设置了 Timeout 参数的 Thread.join() 方法 | 时间结束 / 被调用的线程执行完毕                 |
| LockSupport.parkNanos() 方法             | -                                               |
| LockSupport.parkUntil() 方法             | -                                               |

**死亡（Terminated）**

可以是线程结束任务之后自己结束，或者产生了异常而结束。





往下一看

> at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
> at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:212)
> at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:94)
> at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:496)
> at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:141)
> at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79)
> at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:88)
> at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:502)
> at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1132)
> at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:684)
> at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1539)
> at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.run(NioEndpoint.java:1495)
>
> - locked <0x0000000094c40718> (a org.apache.tomcat.util.net.NioChannel)
>   at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
>   at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
>   at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
>   at java.lang.Thread.run(Thread.java:745)

通过查找<0x000000008015c660> 发现有个线程badge-thread-18锁住了它 ，

> "badge-thread-18" #438 prio=5 os_prio=0 tid=0x00007fdd6403d800 nid=0x5b36 runnable [0x00007fdd282bb000]
>    java.lang.Thread.State: RUNNABLE
> 	at java.util.zip.ZipFile.open(Native Method)
> 	at java.util.zip.ZipFile.<init>(ZipFile.java:219)
> 	at java.util.zip.ZipFile.<init>(ZipFile.java:149)
> 	at java.util.jar.JarFile.<init>(JarFile.java:166)
> 	at java.util.jar.JarFile.<init>(JarFile.java:130)
> 	at org.apache.tomcat.util.compat.JreCompat.jarFileNewInstance(JreCompat.java:170)
> 	at org.apache.tomcat.util.compat.JreCompat.jarFileNewInstance(JreCompat.java:155)
> 	at org.apache.catalina.webresources.AbstractArchiveResourceSet.openJarFile(AbstractArchiveResourceSet.java:316)
> 	- locked <0x00000000903a4828> (a java.lang.Object)
> 	at org.apache.catalina.webresources.AbstractSingleArchiveResourceSet.getArchiveEntry(AbstractSingleArchiveResourceSet.java:96)
> 	at org.apache.catalina.webresources.AbstractArchiveResourceSet.getResource(AbstractArchiveResourceSet.java:265)
> 	at org.apache.catalina.webresources.StandardRoot.getResourcesInternal(StandardRoot.java:327)
> 	at org.apache.catalina.webresources.CachedResource.validateResources(CachedResource.java:127)
> 	- locked <0x000000008015c660> (a org.apache.catalina.webresources.CachedResource)
> 	at org.apache.catalina.webresources.Cache.getResources(Cache.java:147)
> 	at org.apache.catalina.webresources.StandardRoot.getResources(StandardRoot.java:315)
> 	at org.apache.catalina.webresources.StandardRoot.getClassLoaderResources(StandardRoot.java:231)
> 	at org.apache.catalina.loader.WebappClassLoaderBase.findResources(WebappClassLoaderBase.java:995)
> 	at java.lang.ClassLoader.getResources(ClassLoader.java:1142)
> 	at com.alibaba.fastjson.util.ServiceLoader.load(ServiceLoader.java:33)
> 	at com.alibaba.fastjson.parser.ParserConfig.getDeserializer(ParserConfig.java:459)
> 	at com.alibaba.fastjson.parser.ParserConfig.getDeserializer(ParserConfig.java:354)
> 	at com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:639)
> 	at com.alibaba.fastjson.JSON.parseObject(JSON.java:350)
> 	at com.alibaba.fastjson.JSON.parseObject(JSON.java:318)
> 	at com.alibaba.fastjson.JSON.parseObject(JSON.java:281)

通过上面的堆栈，我们先猜测下跟ParserConfig的getDeserializer方法可能有若干关系，先看下代码：

```java
public ObjectDeserializer getDeserializer(Class<?> clazz, Type type) {
    ObjectDeserializer derializer = deserializers.get(type);
    if (derializer != null) {
        return derializer;
    }

    if (type == null) {
        type = clazz;
    }

    derializer = deserializers.get(type);
    if (derializer != null) {
        return derializer;
    }

    {
        JSONType annotation = clazz.getAnnotation(JSONType.class);
        if (annotation != null) {
            Class<?> mappingTo = annotation.mappingTo();
            if (mappingTo != Void.class) {
                return getDeserializer(mappingTo, mappingTo);
            }
        }
    }

    if (type instanceof WildcardType || type instanceof TypeVariable || type instanceof ParameterizedType) {
        derializer = deserializers.get(clazz);
    }

    if (derializer != null) {
        return derializer;
    }

    String className = clazz.getName();
    className = className.replace('$', '.');

    if (className.startsWith("java.awt.") //
        && AwtCodec.support(clazz)) {
        if (!awtError) {
            try {
                deserializers.put(Class.forName("java.awt.Point"), AwtCodec.instance);
                deserializers.put(Class.forName("java.awt.Font"), AwtCodec.instance);
                deserializers.put(Class.forName("java.awt.Rectangle"), AwtCodec.instance);
                deserializers.put(Class.forName("java.awt.Color"), AwtCodec.instance);
            } catch (Throwable e) {
                // skip
                awtError = true;
            }

            derializer = AwtCodec.instance;
        }
    }

    if (!jdk8Error) {
        try {
            if (className.startsWith("java.time.")) {
                
                deserializers.put(Class.forName("java.time.LocalDateTime"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.LocalDate"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.LocalTime"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.ZonedDateTime"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.OffsetDateTime"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.OffsetTime"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.ZoneOffset"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.ZoneRegion"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.ZoneId"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.Period"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.Duration"), Jdk8DateCodec.instance);
                deserializers.put(Class.forName("java.time.Instant"), Jdk8DateCodec.instance);
                
                derializer = deserializers.get(clazz);
            } else if (className.startsWith("java.util.Optional")) {
                
                deserializers.put(Class.forName("java.util.Optional"), OptionalCodec.instance);
                deserializers.put(Class.forName("java.util.OptionalDouble"), OptionalCodec.instance);
                deserializers.put(Class.forName("java.util.OptionalInt"), OptionalCodec.instance);
                deserializers.put(Class.forName("java.util.OptionalLong"), OptionalCodec.instance);
                
                derializer = deserializers.get(clazz);
            }
        } catch (Throwable e) {
            // skip
            jdk8Error = true;
        }
    }

    if (className.equals("java.nio.file.Path")) {
        deserializers.put(clazz, MiscCodec.instance);
    }

    if (clazz == Map.Entry.class) {
        deserializers.put(clazz, MiscCodec.instance);
    }

    final ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    try {
        for (AutowiredObjectDeserializer autowired : ServiceLoader.load(AutowiredObjectDeserializer.class,classLoader)) {
            for (Type forType : autowired.getAutowiredFor()) {
                deserializers.put(forType, autowired);
            }
        }
    } catch (Exception ex) {
        // skip
    }

    if (derializer == null) {
        derializer = deserializers.get(type);
    }

    if (derializer != null) {
        return derializer;
    }

    if (clazz.isEnum()) {
        derializer = new EnumDeserializer(clazz);
    } else if (clazz.isArray()) {
        derializer = ObjectArrayCodec.instance;
    } else if (clazz == Set.class || clazz == HashSet.class || clazz == Collection.class || clazz == List.class
               || clazz == ArrayList.class) {
        derializer = CollectionCodec.instance;
    } else if (Collection.class.isAssignableFrom(clazz)) {
        derializer = CollectionCodec.instance;
    } else if (Map.class.isAssignableFrom(clazz)) {
        derializer = MapDeserializer.instance;
    } else if (Throwable.class.isAssignableFrom(clazz)) {
        derializer = new ThrowableDeserializer(this, clazz);
    } else {
        derializer = createJavaBeanDeserializer(clazz, type);
    }

    putDeserializer(type, derializer);

    return derializer;
}
```

deserializers又指的是：

```java
private final IdentityHashMap<Type, ObjectDeserializer> deserializers         = new IdentityHashMap<Type, ObjectDeserializer>();
```

大致逻辑是先通过缓存的IdentityHashMap查找，找不到就判断是否注解@JSONType，从中解析。如果还是找不到如果类型是WildcardType、TypeVariable 、 ParameterizedType从中解析。还是不行就使用当前线程类加载器 查找 META-INF/services/AutowiredObjectDeserializer.class实现类。余下就不分析了，当大多数json解析走至此处就要想一下，为什么从缓存的IdentityHashMap查找不到该类型？

通过dump分析可以得到com.alibaba.fastjson.util.IdentityHashMap非常大，也验证了我的看法。

IdentityHashMap是通过System.identityHashCode获取的key，但是这个几乎是唯一的，就算是同一个类型并不是同一个引用都将会有问题



引发这段代码的业务代码大致如下：

```java
@Test
public void leak(){
    Student student=new Student();
    student.setName("1");
    CacheWrapper cacheWrapper = new CacheWrapper();
    cacheWrapper.setCacheObject(student);
    byte[] bytes = JSON.toJSONBytes(cacheWrapper);
    while (true){
        Object o = JSON.parseObject(bytes, new ParameterizedTypeImpl(new Type[]{Student.class}, CacheWrapper.class.getDeclaringClass(), CacheWrapper.class));
    }
}
```

```java
@Data
public class Student implements Serializable{
    private String name;
}
```



```java
public class CacheWrapper<T> implements Serializable, Cloneable {
    private static final long serialVersionUID=1L;


    /**
     * 缓存数据
     */
    private T cacheObject;

    /**
     * 缓存时长
     */
    private int expire;

    public CacheWrapper() {
    }

    public CacheWrapper(T cacheObject, int expire) {
        this.cacheObject=cacheObject;
        this.expire=expire;
    }


    @Override
    public Object clone() throws CloneNotSupportedException {
        @SuppressWarnings("unchecked")
        CacheWrapper<T> tmp=(CacheWrapper<T>)super.clone();
        tmp.setCacheObject(this.cacheObject);
        return tmp;
    }

    public T getCacheObject() {
        return cacheObject;
    }

    public void setCacheObject(T cacheObject) {
        this.cacheObject = cacheObject;
    }

    public int getExpire() {
        return expire;
    }

    public void setExpire(int expire) {
        this.expire = expire;
    }
}
```

通过 -Xmx50m配置我们就能得到

java.lang.OutOfMemoryError: GC overhead limit exceeded

	at java.util.Arrays.copyOf(Arrays.java:3236)
	at java.lang.StringCoding.safeTrim(StringCoding.java:79)
	at java.lang.StringCoding.access$300(StringCoding.java:50)
	at java.lang.StringCoding$StringEncoder.encode(StringCoding.java:305)
	at java.lang.StringCoding.encode(StringCoding.java:344)
	at java.lang.String.getBytes(String.java:918)
	at java.io.UnixFileSystem.getBooleanAttributes0(Native Method)
	at java.io.UnixFileSystem.getBooleanAttributes(UnixFileSystem.java:242)
	at java.io.File.exists(File.java:819)
	at sun.misc.URLClassPath$FileLoader.getResource(URLClassPath.java:1245)
	at sun.misc.URLClassPath$FileLoader.findResource(URLClassPath.java:1212)
	at sun.misc.URLClassPath$1.next(URLClassPath.java:240)
	at sun.misc.URLClassPath$1.hasMoreElements(URLClassPath.java:250)
	at java.net.URLClassLoader$3$1.run(URLClassLoader.java:601)
	at java.net.URLClassLoader$3$1.run(URLClassLoader.java:599)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader$3.next(URLClassLoader.java:598)
	at java.net.URLClassLoader$3.hasMoreElements(URLClassLoader.java:623)
	at sun.misc.CompoundEnumeration.next(CompoundEnumeration.java:45)
	at sun.misc.CompoundEnumeration.hasMoreElements(CompoundEnumeration.java:54)
	at com.alibaba.fastjson.util.ServiceLoader.load(ServiceLoader.java:34)
	at com.alibaba.fastjson.parser.ParserConfig.getDeserializer(ParserConfig.java:459)
	at com.alibaba.fastjson.parser.ParserConfig.getDeserializer(ParserConfig.java:354)
	at com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:639)
	at com.alibaba.fastjson.JSON.parseObject(JSON.java:350)
	at com.alibaba.fastjson.JSON.parseObject(JSON.java:318)
	at com.alibaba.fastjson.JSON.parseObject(JSON.java:281)
	at com.alibaba.fastjson.JSON.parseObject(JSON.java:381)
	at com.alibaba.fastjson.JSON.parseObject(JSON.java:361)
通过分析发现，每次new一个ParameterizedTypeImpl，就算泛型是一个类，也会在IdentityHashMap储存两遍，这样造成了内存泄漏。



ParameterizedTypeImpl的代码如下：

```java
public class ParameterizedTypeImpl implements ParameterizedType {

    private final Type[] actualTypeArguments;
    private final Type   ownerType;
    private final Type   rawType;

    public ParameterizedTypeImpl(Type[] actualTypeArguments, Type ownerType, Type rawType){
        this.actualTypeArguments = actualTypeArguments;
        this.ownerType = ownerType;
        this.rawType = rawType;
    }

    public Type[] getActualTypeArguments() {
        return actualTypeArguments;
    }

    public Type getOwnerType() {
        return ownerType;
    }

    public Type getRawType() {
        return rawType;
    }


    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        ParameterizedTypeImpl that = (ParameterizedTypeImpl) o;

        // Probably incorrect - comparing Object[] arrays with Arrays.equals
        if (!Arrays.equals(actualTypeArguments, that.actualTypeArguments)) return false;
        if (ownerType != null ? !ownerType.equals(that.ownerType) : that.ownerType != null) return false;
        return rawType != null ? rawType.equals(that.rawType) : that.rawType == null;

    }

    @Override
    public int hashCode() {
        int result = actualTypeArguments != null ? Arrays.hashCode(actualTypeArguments) : 0;
        result = 31 * result + (ownerType != null ? ownerType.hashCode() : 0);
        result = 31 * result + (rawType != null ? rawType.hashCode() : 0);
        return result;
    }
}
```

如果通过hashcode而不是System.identityHashCode就不会有内存泄漏的问题。所以我们尝试把ParameterizedTypeImpl缓存一下。代码大致如下：

```java
static ConcurrentMap<Type, Type> classTypeCache
            = new ConcurrentHashMap<Type, Type>(16, 0.75f, 1);
@Test
public void fixLeak(){
    Student student=new Student();
    student.setName("1");
    CacheWrapper cacheWrapper = new CacheWrapper();
    cacheWrapper.setCacheObject(student);
    byte[] bytes = JSON.toJSONBytes(cacheWrapper);
    while (true){
        Type argkey = new ParameterizedTypeImpl(new Type[]{Student.class}, CacheWrapper.class.getDeclaringClass(), CacheWrapper.class);
        Type cachedType = classTypeCache.get(argkey);
        if (cachedType == null) {
            classTypeCache.putIfAbsent(argkey, argkey);
            cachedType = classTypeCache.get(argkey);
        }
        Object o = JSON.parseObject(bytes, cachedType);
    }
}
```

通过测试发现不会在发生问题了



上线后监控一段时间问题解决.