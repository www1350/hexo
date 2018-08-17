---
title: dubbo源码解析（十一）SPI内核
tags:
  - dubbo
  - rpc
categories:
  - 中间件
  - 源码
abbrlink: 7583e99e
date: 2018-07-12 14:38:29
---

关于java SPI和dubbo SPI的简单阐述https://www1350.github.io/#post/114

拿解析一的ServiceConfig的doExportUrlsFor1Protocol为例：

```java
Exporter<?> exporter = protocol.export(wrapperInvoker);
```

这里的protocol来自

```java
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

但是我们debug的时候会发现这个protocol命名是com.alibaba.dubbo.rpc.Protocol$Adaptive,可见是动态生成的。



通过ExtensionLoader的createAdaptiveExtensionClassCode方法。我们可以获取到这么一段代码。

```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```



接口如下：

```java
@SPI("dubbo")
public interface Protocol {
    
    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```

我们可以看到动态生成的代码里面，通过getAdaptiveExtension获取的时候，如果没有注解@Adaptive就会把实现类写成UnsupportedOperationException，而对于@Adaptive我们看下一个动态生成的代码：





```java
package com.alibaba.dubbo.remoting;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Transporter$Adaptive implements com.alibaba.dubbo.remoting.Transporter {
    public com.alibaba.dubbo.remoting.Client connect(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([client, transporter])");
        com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
        return extension.connect(arg0, arg1);
    }

    public com.alibaba.dubbo.remoting.Server bind(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("server", url.getParameter("transporter", "netty"));
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([server, transporter])");
        com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
        return extension.bind(arg0, arg1);
    }
}
```



我们注意到所不同的地方在于

```java
String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
```

和

```java
com.alibaba.dubbo.common.URL url = arg0.getUrl(); 
String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
```



```java
@SPI("netty")
public interface Transporter {
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    //client    transporter
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;

}
```

现在我们大胆猜测，@Adaptive如果没有设置参数，extName就会拿url.get+接口名；如果有设置参数，extName就会拿最后一个参数和SPI的name获取url.getParameter(最后一个参数名, SPI名)，得到的结果在用url.getParameter(倒数第二个参数名，上一个结果)，一直这样直到取完参数。另外还通过官方文档了解到，接口方法不注解@Adaptive，在实现类注解@Adaptive将会自动激活这个拓展。

另外一点我们注意到不管参数顺序如何，都能拿到url，这里可能进行了类型判断。接下来我们就验证下想象。



```java
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        //单例double check
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //这里是关键
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
```



```java
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    //实现类有注解Adaptive，就直接返回
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //没有实现类，类上注解Adaptive，动态生成
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```



```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    //DCL
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

获取SPI，并获取

```java
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if (value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    // /META-INF/dubbo/internal/
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    //   META-INF/dubbo/
    loadFile(extensionClasses, DUBBO_DIRECTORY);
    //   META-INF/services/  因为这里是存到cache里面的，所以优先级自然是反过来services>dubbo>internal
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```



我们进入到了SPI机制的核心,通过loadFile可以加载到所有配置的类：

```java
private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL url = urls.nextElement();
                try {
                    BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
                    try {
                        String line = null;
                        while ((line = reader.readLine()) != null) {
                            //去除注释
                            final int ci = line.indexOf('#');
                            if (ci >= 0) line = line.substring(0, ci);
                            line = line.trim();
                            if (line.length() > 0) {
                                try {
                                    String name = null;
                                    int i = line.indexOf('=');
                                    if (i > 0) {
                                        //获取name
                                        name = line.substring(0, i).trim();
                                        //获取class 全限定类名
                                        line = line.substring(i + 1).trim();
                                    }
                                    if (line.length() > 0) {
                                        Class<?> clazz = Class.forName(line, true, classLoader);
                         //类上注解Adaptive，缓存到cachedAdaptiveClass
                                        if (clazz.isAnnotationPresent(Adaptive.class)) {
                                            if (cachedAdaptiveClass == null) {
                                                cachedAdaptiveClass = clazz;
                                            }  else {
                                            try {
                                                //有构造方法是带一个参数且参数类型是接口
                                                clazz.getConstructor(type);
                                                Set<Class<?>> wrappers = cachedWrapperClasses;
                                                if (wrappers == null) {
                          //没有Adaptive注解，反射所有实现类带参构造放入cachedWrapperClasses
                                                    cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                                    wrappers = cachedWrapperClasses;
                                                }
                                                wrappers.add(clazz);
                                            } 
                                        }
                                    }
                                }
                            }
                        } // end of while read lines
                    } finally {
                        reader.close();
                    }
                } catch (Throwable t) {
                    logger.error("Exception when load extension class(interface: " +
                            type + ", class file: " + url + ") in " + url, t);
                }
            } // end of while urls
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class(interface: " +
                type + ", description file: " + fileName + ").", t);
    }
}
```





```java
@SuppressWarnings("unchecked")
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```



```java
private Class<?> createAdaptiveExtensionClass() {
    //生成拼接源码
    String code = createAdaptiveExtensionClassCode();
    //获取自定义ClassLoader ->ExtensionLoader
    ClassLoader classLoader = findClassLoader();
    //获取到AdaptiveCompiler
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    //最终使用JavassistCompiler编译
    return compiler.compile(code, classLoader);
}
```


下面是$Adaptive类生成的代码：

```java
private String createAdaptiveExtensionClassCode() {
    StringBuilder codeBuidler = new StringBuilder();
    Method[] methods = type.getMethods();
    boolean hasAdaptiveAnnotation = false;
    //接口是否有@Adaptive注解
    for (Method m : methods) {
        if (m.isAnnotationPresent(Adaptive.class)) {
            hasAdaptiveAnnotation = true;
            break;
        }
    }
    // 没有Adaptive注解直接报错
    if (!hasAdaptiveAnnotation)
        throw new IllegalStateException("No adaptive method on extension " + type.getName() + ", refuse to create the adaptive class!");
//接口所在包 "package com.alibaba.dubbo.rpc;"
    codeBuidler.append("package " + type.getPackage().getName() + ";");
// "import com.alibaba.dubbo.common.extension.ExtensionLoader;"
    codeBuidler.append("\nimport " + ExtensionLoader.class.getName() + ";");
// 接口名+$Adaptive生成类名   "public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {"
    codeBuidler.append("\npublic class " + type.getSimpleName() + "$Adaptive" + " implements " + type.getCanonicalName() + " {");
//遍历方法
    for (Method method : methods) {
        //返回
        Class<?> rt = method.getReturnType();
        //入参
        Class<?>[] pts = method.getParameterTypes();
        //异常
        Class<?>[] ets = method.getExceptionTypes();
        //获取方法Adaptive注解
        Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
        StringBuilder code = new StringBuilder(512);
        //没有Adaptive注解的 拼接实现为“throw new UnsupportedOperationException”
        if (adaptiveAnnotation == null) {
            code.append("throw new UnsupportedOperationException(\"method ")
                    .append(method.toString()).append(" of interface ")
                    .append(type.getName()).append(" is not adaptive method!\");");
        } else {
            int urlTypeIndex = -1;
            //获取类型为url的参数
            for (int i = 0; i < pts.length; ++i) {
                if (pts[i].equals(URL.class)) {
                    urlTypeIndex = i;
                    break;
                }
            }
            // 有url类型参数
            if (urlTypeIndex != -1) {
                // 判空 “if (arg0 == null) throw new IllegalArgumentException”
                String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
                        urlTypeIndex);
                code.append(s);
                //“Url url = ”
                s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex);
                code.append(s);
            }
            // 参数里面没有url类型的参数，取每个参数里面getxxx看有没有getUrl
            else {
                String attribMethod = null;

                // 找到getUrl，LBL_PTS为了一个break跳出两层循环
                LBL_PTS:
                for (int i = 0; i < pts.length; ++i) {
                    Method[] ms = pts[i].getMethods();
                    for (Method m : ms) {
                        String name = m.getName();
                        if ((name.startsWith("get") || name.length() > 3)
                                && Modifier.isPublic(m.getModifiers())
                                && !Modifier.isStatic(m.getModifiers())
                                && m.getParameterTypes().length == 0
                                && m.getReturnType() == URL.class) {
                            urlTypeIndex = i;
                            attribMethod = name;
                            break LBL_PTS;
                        }
                    }
                }
                if (attribMethod == null) {
                    throw new IllegalStateException("fail to create adaptive class for interface " + type.getName()
                            + ": not found url parameter or url attribute in parameters of method " + method.getName());
                }

                // 判空
                String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");",
                        urlTypeIndex, pts[urlTypeIndex].getName());
                code.append(s);
                s = String.format("\nif (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");",
                        urlTypeIndex, attribMethod, pts[urlTypeIndex].getName(), attribMethod);
                code.append(s);

                s = String.format("%s url = arg%d.%s();", URL.class.getName(), urlTypeIndex, attribMethod);
                code.append(s);
            }

            String[] value = adaptiveAnnotation.value();
            // Adaptive注解没有参数get+接口名
            if (value.length == 0) {
                char[] charArray = type.getSimpleName().toCharArray();
                StringBuilder sb = new StringBuilder(128);
                for (int i = 0; i < charArray.length; i++) {
                    if (Character.isUpperCase(charArray[i])) {
                        if (i != 0) {
                            sb.append(".");
                        }
                        sb.append(Character.toLowerCase(charArray[i]));
                    } else {
                        sb.append(charArray[i]);
                    }
                }
                value = new String[]{sb.toString()};
            }

            boolean hasInvocation = false;
            for (int i = 0; i < pts.length; ++i) {
                if (pts[i].getName().equals("com.alibaba.dubbo.rpc.Invocation")) {
                    // Invoker判空
                    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"invocation == null\");", i);
                    code.append(s);
                    s = String.format("\nString methodName = arg%d.getMethodName();", i);
                    code.append(s);
                    hasInvocation = true;
                    break;
                }
            }

            String defaultExtName = cachedDefaultName;
            String getNameCode = null;
            for (int i = value.length - 1; i >= 0; --i) {
                if (i == value.length - 1) {
                    if (null != defaultExtName) {
                        if (!"protocol".equals(value[i]))
                            if (hasInvocation)
                                getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                        else
                            //如果是protocol，取defaultExtName，也就是SPI的value
                            getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                    } else {
                        if (!"protocol".equals(value[i]))
                            if (hasInvocation)
                                getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                        else
                            getNameCode = "url.getProtocol()";
                    }
                } else {
                    if (!"protocol".equals(value[i]))
                        if (hasInvocation)
                            getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                        else
                            getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                    else
                        getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
                }
            }
            code.append("\nString extName = ").append(getNameCode).append(";");
            // check extName == null?
            String s = String.format("\nif(extName == null) " +
                            "throw new IllegalStateException(\"Fail to get extension(%s) name from url(\" + url.toString() + \") use keys(%s)\");",
                    type.getName(), Arrays.toString(value));
            code.append(s);

            s = String.format("\n%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);",
                    type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
            code.append(s);

            // return statement
            if (!rt.equals(void.class)) {
                code.append("\nreturn ");
            }

            s = String.format("extension.%s(", method.getName());
            code.append(s);
            for (int i = 0; i < pts.length; i++) {
                if (i != 0)
                    code.append(", ");
                code.append("arg").append(i);
            }
            code.append(");");
        }

        codeBuidler.append("\npublic " + rt.getCanonicalName() + " " + method.getName() + "(");
        for (int i = 0; i < pts.length; i++) {
            if (i > 0) {
                codeBuidler.append(", ");
            }
            codeBuidler.append(pts[i].getCanonicalName());
            codeBuidler.append(" ");
            codeBuidler.append("arg" + i);
        }
        codeBuidler.append(")");
        if (ets.length > 0) {
            codeBuidler.append(" throws ");
            for (int i = 0; i < ets.length; i++) {
                if (i > 0) {
                    codeBuidler.append(", ");
                }
                codeBuidler.append(ets[i].getCanonicalName());
            }
        }
        codeBuidler.append(" {");
        codeBuidler.append(code.toString());
        codeBuidler.append("\n}");
    }
    codeBuidler.append("\n}");
    if (logger.isDebugEnabled()) {
        logger.debug(codeBuidler.toString());
    }
    return codeBuidler.toString();
}
```







现在我们还有个疑问？如果只分析这些代码,会认为最开始的解析一例子，暴露的是RegistryProtocol，然而外面又包裹了两层：ProtocolListenerWrapper和ProtocolFilterWrapper，这是如何实现的呢？

``` com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);```

拿到的是ProtocolListenerWrapper，下面我们分析下getExtension，他最终通过createExtension创建对象实例

```java
private T createExtension(String name) {
    //获取从配置文件得到的类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        //得到实例
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //将其他实例注入这个实例的set方法
        //这里是RegistryProtocol的实例
        injectExtension(instance);
        //获取接口的所有实现类 ，ProtocolFilterWrapper、ProtocolListenerWrapper
        //RegistryProtocol注入ProtocolFilterWrapper构造方法得到实例，接着注入ProtocolListenerWrapper得到实例
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        //返回ProtocolListenerWrapper
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```