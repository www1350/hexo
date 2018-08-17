---
title: SPI
abbrlink: 2400ed9b
date: 2017-09-01 22:40:29
tags: [字节码,java,SPI]
categories: 基础
---

SPI指的是Service Provider Interface，服务提供接口

我们系统里抽象的各个模块，往往有很多不同的实现方案，比如日志模块的方案，xml解析模块、jdbc模块的方案等。面向的对象的设计里，我们一般推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。

为了实现在模块装配的时候能不在程序里动态指明，这就需要一种服务发现机制。Java spi就是提供这样的一个机制：为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要。


当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 

基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk提供服务实现查找的一个工具类：java.util.ServiceLoader. 



```java
package spi;  
public interface Search {  
  public void search();  
}  



package spi;  
public class FileSearch implements Search {  
  @Override  
  public void search() {  
    System.out.println("哥是文件搜索");  
  }  
}  



package spi;  
public class DataBaseSearch implements Search {  
  @Override  
  public void search() {  
    System.out.println("哥是database搜索");  
  }  
}  


public class DoSearch {  
  public static void main(String[] args) {  
    ServiceLoader<Search> sl = ServiceLoader.load(Search.class);  
    Iterator<Search> s = sl.iterator();  
    if (s.hasNext()) {  
      Search ss = s.next();  
      ss.search();  
    }  
  }  
}  
```




最后在META-INF/services目录下创建spi.Search(包名+接口名)文件，
当文件内容为spi.FileSearch（包名+实现类名）时，程序输出结果为：哥是文件搜索
当内容为spi.DataBaseSearch时，程序输出结果为：哥是database搜索.
由此可以看出DOSearch类中没有任何和具体实现有关的代码，而是基于spi的机制去查找服务的实现



dubbo中基于SPI思想的实现

Dubbo改进了JDK标准的SPI的以下问题：

1.JDK标准的SPI会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。

2.如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK标准的ScriptEngine，通过getName();获取脚本类型的名称，但如果RubyScriptEngine因为所依赖的jruby.jar不存在，导致RubyScriptEngine类加载失败，这个失败原因被吃掉了，和ruby对应不起来，当用户执行ruby脚本时，会报不支持ruby，而不是真正失败的原因。

3.增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。


```java
public @interface SPI {  
    /**  
     * 缺省扩展点名。  
     */   
    String value() default "";  
}  
```


```java
@SPI("spring")   
public interface Container {  
}  
```

dubbo/dubbo-container/src/main/resources/META-INF/dubbo/internal

```properties
jetty=com.alibaba.dubbo.container.jetty.JettyContainer
log4j=com.alibaba.dubbo.container.log4j.Log4jContainer
logback=com.alibaba.dubbo.container.logback.LogbackContainer
spring=com.alibaba.dubbo.container.spring.SpringContainer
```

`ExtensionLoader.getExtensionLoader(Container.class).getExtension(name)`

ExtensionLoader
```java
    // 此方法已经getExtensionClasses方法同步过。
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if(defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if(value != null && (value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if(names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if(names.length == 1) cachedDefaultName = names[0];
            }
        }
        
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }



    private static final String SERVICES_DIRECTORY = "META-INF/services/";

    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";
```


cachedAdaptiveClass : 当前Extension类型对应的AdaptiveExtension类型(只能一个)

cachedWrapperClasses : 当前Extension类型对应的所有Wrapper实现类型(无顺序)

cachedActivates : 当前Extension实现自动激活实现缓存(map,无序)

cachedNames : 扩展点实现类对应的名称(如配置多个名称则值为第一个)

cachedDefaultName : 当前扩展点的默认实现名称

cachedClasses : 扩展点实现名称对应的实现类(一个实现类可能有多个名称)

loadFile最后调用的是Class.forName(line, true, classLoader);

然后injectExtension 注入

```java
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```


```java
public class SpiExtensionFactory implements ExtensionFactory {

    public <T> T getExtension(Class<T> type, String name) {
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (loader.getSupportedExtensions().size() > 0) {
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}
```


```java
public class SpringExtensionFactory implements ExtensionFactory {
    
    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
    
    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
    }

    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) {
                Object bean = context.getBean(name);
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }
        return null;
    }

}

```

![1](https://user-images.githubusercontent.com/7789698/29960840-e18362a4-8f2f-11e7-9891-4dc0d0fd7ad6.png)