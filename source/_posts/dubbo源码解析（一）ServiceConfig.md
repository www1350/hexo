---
title: dubbo源码解析（一）ServiceConfig
abbrlink: 34093d70
date: 2018-04-03 22:43:34
tags: [dubbo,rpc]
categories: [中间件,源码]
---

## 什么是Dubbo?

Dubbo是阿里巴巴公司开源的一个高性能优秀的服务框架，使得应用可通过高性能的RPC实现服务的输出和输入功能，以及SOA服务治理方案，和spring框架无缝集成。

dubbo作为一个非常好的rpc项目，广泛在国内使用。

官网：http://dubbo.incubator.apache.org/

github地址：https://github.com/apache/incubator-dubbo

<!-- more -->

我们先看下项目结构：
![image](https://user-images.githubusercontent.com/7789698/38068732-4c032100-3345-11e8-9a0b-ff30047a09ec.png)

模块说明：

- **dubbo-common 公共逻辑模块**：包括 Util 类和通用模型。
- **dubbo-remoting 远程通讯模块**：相当于 Dubbo 协议的实现，如果 RPC 用 RMI协议则不需要使用此包。
- **dubbo-rpc 远程调用模块**：抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理。
- **dubbo-cluster 集群模块**：将多个服务提供方伪装为一个提供方，包括：负载均衡, 容错，路由等，集群的地址列表可以是静态配置的，也可以是由注册中心下发。
- **dubbo-registry 注册中心模块**：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象。
- **dubbo-monitor 监控模块**：统计服务调用次数，调用时间的，调用链跟踪的服务。
- **dubbo-config 配置模块**：是 Dubbo 对外的 API，用户通过 Config 使用D ubbo，隐藏 Dubbo 所有细节。
- **dubbo-container 容器模块**：是一个 Standlone 的容器，以简单的 Main 加载 Spring 启动，因为服务通常不需要 Tomcat/JBoss 等 Web 容器的特性，没必要用 Web 容器去加载服务。

整体上按照分层结构进行分包，与分层的不同点在于：

- container 为服务容器，用于部署运行服务，没有在层中画出。
- protocol 层和 proxy 层都放在 rpc 模块中，这两层是 rpc 的核心，在不需要集群也就是只有一个提供者时，可以只使用这两层完成 rpc 调用。
- transport 层和 exchange 层都放在 remoting 模块中，为 rpc 调用的通讯基础。
- serialize 层放在 common 模块中，以便更大程度复用。

![dubbo-framework](https://user-images.githubusercontent.com/7789698/38149891-0c82215c-348f-11e8-8bb7-12c61bc8f3d2.jpg)


图例说明：

- 图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。
- 图中从下至上分为十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用，其中，Service 和 Config 层为 API，其它各层均为 SPI。
- 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。
- 图中蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调时链，紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。

## 各层说明

- **config 配置层**：对外配置接口，以 `ServiceConfig`, `ReferenceConfig` 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
- **proxy 服务代理层**：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 `ServiceProxy` 为中心，扩展接口为 `ProxyFactory`
- **registry 注册中心层**：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 `RegistryFactory`, `Registry`, `RegistryService`
- **cluster 路由层**：封装多个提供者的路由及负载均衡，并桥接注册中心，以 `Invoker` 为中心，扩展接口为 `Cluster`, `Directory`, `Router`, `LoadBalance`
- **monitor 监控层**：RPC 调用次数和调用时间监控，以 `Statistics` 为中心，扩展接口为 `MonitorFactory`, `Monitor`, `MonitorService`
- **protocol 远程调用层**：封装 RPC 调用，以 `Invocation`, `Result` 为中心，扩展接口为 `Protocol`, `Invoker`, `Exporter`
- **exchange 信息交换层**：封装请求响应模式，同步转异步，以 `Request`, `Response` 为中心，扩展接口为 `Exchanger`, `ExchangeChannel`, `ExchangeClient`, `ExchangeServer`
- **transport 网络传输层**：抽象 mina 和 netty 为统一接口，以 `Message` 为中心，扩展接口为 `Channel`, `Transporter`, `Client`, `Server`, `Codec`
- **serialize 数据序列化层**：可复用的一些工具，扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`
  ![image-201803301749139](https://user-images.githubusercontent.com/7789698/38149952-44d81a98-348f-11e8-86d2-af78b14b4ce7.png)



## 启动demo

dubbo-demo里面有一个例子，一个消费者一个提供者

### dubbo-demo-api

```java
public interface DemoService {

    String sayHello(String name);

}
```



### dubbo-demo-provider

```java
public class DemoServiceImpl implements DemoService {

    public String sayHello(String name) {
        System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response from provider: " + RpcContext.getContext().getLocalAddress();
    }

}
```



```java
public class Provider {

    public static void main(String[] args) throws Exception {
        //Prevent to get IPV6 address,this way only work in debug mode
        //But you can pass use -Djava.net.preferIPv4Stack=true,then it work well whether in debug mode or not
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-provider.xml"});
        context.start();

        System.in.read(); // press any key to exit
    }

}
```

dubbo-demo-provider.xml

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- provider's application name, used for tracing dependency relationship -->
    <dubbo:application name="demo-provider"/>

    <!-- use multicast registry center to export service -->
    <dubbo:registry address="zookeeper://localhost:2181"/>

    <!-- use dubbo protocol to export service on port 20880 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- service implementation, as same as regular local bean -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>

    <!-- declare the service interface to be exported -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>

</beans>
```



### dubbo-demo-consumer

```java
public class Consumer {

    public static void main(String[] args) {
        //Prevent to get IPV6 address,this way only work in debug mode
        //But you can pass use -Djava.net.preferIPv4Stack=true,then it work well whether in debug mode or not
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-consumer.xml"});
        context.start();
        DemoService demoService = (DemoService) context.getBean("demoService"); // get remote service proxy
        while (true) {
            try {
                Thread.sleep(1000);
                String hello = demoService.sayHello("world"); // call remote method
                System.out.println(hello); // get result
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
        }

    }
}
```



dubbo-demo-consumer.xml

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- consumer's application name, used for tracing dependency relationship (not a matching criterion),
    don't set it same as provider -->
    <dubbo:application name="demo-consumer"/>

    <!-- use multicast registry center to discover service -->
    <dubbo:registry address="zookeeper://localhost:2181"/>

    <!-- generate proxy for the remote service, then demoService can be used in the same way as the
    local regular interface -->
    <dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService"/>

</beans>
```



启动provider后

我们可以看到zk里面的状况

会出现两个节点/dubbo/com.alibaba.dubbo.demo.DemoService/configurators和/dubbo/com.alibaba.dubbo.demo.DemoService/providers

<img width="563" alt="image-201803291409570" src="https://user-images.githubusercontent.com/7789698/38073175-83cdfdfa-335c-11e8-84fd-68271bb820e2.png">






我们对他解码可以看到

dubbo://172.17.8.254:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=85385&side=provider&timestamp=1522303604681



启动consumer后

zk的节点/dubbo/com.alibaba.dubbo.demo.DemoService下多了routers, consumers两个节点

consumers再次解码可以看到

consumer://172.17.8.254/com.alibaba.dubbo.demo.DemoService?application=demo-consumer&category=consumers&check=false&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=85452&qos.port=33333&side=consumer&timestamp=1522303933804



最后我们看到控制台每秒打印
<img width="403" alt="image-201803291414597" src="https://user-images.githubusercontent.com/7789698/38073163-75ecd68e-335c-11e8-8c03-a8b8c49d3443.png">



## xml解析分析

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    public void init() {
        //分别解析到对应的config类
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```

DubboBeanDefinitionParser里面包含了解析xml的行为，就不具体展开


## config 配置层
### ServiceBean源码分析

![image](https://user-images.githubusercontent.com/7789698/38073283-e8c810ba-335c-11e8-94d6-77e0cb2adff7.png)



ApplicationContextAware

```java
public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    //加入SpringExtensionFactory
    SpringExtensionFactory.addApplicationContext(applicationContext);
    if (applicationContext != null) {
        SPRING_CONTEXT = applicationContext;
        try {
            //反射AbstractApplicationContext的addApplicationListener将本类实例加入，监听容器加载完成后调用onApplicationEvent方法
            Method method = applicationContext.getClass().getMethod("addApplicationListener", new Class<?>[]{ApplicationListener.class}); // backward compatibility to spring 2.0.1
            method.invoke(applicationContext, new Object[]{this});
            supportedApplicationListener = true;
        } catch (Throwable t) {
            if (applicationContext instanceof AbstractApplicationContext) {
                try {
                    Method method = AbstractApplicationContext.class.getDeclaredMethod("addListener", new Class<?>[]{ApplicationListener.class}); // backward compatibility to spring 2.0.1
                    if (!method.isAccessible()) {
                        method.setAccessible(true);
                    }
                    method.invoke(applicationContext, new Object[]{this});
                    supportedApplicationListener = true;
                } catch (Throwable t2) {
                }
            }
        }
    }
}
```

继承`ApplicationListener<ContextRefreshedEvent>` 监听事件，bean加载的最后调用

```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    //delay是-1或者没配置就延迟到Spring 初始化完成后，再暴露服务
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        export();
    }
}
```

bean被初始化的时候利用InitializingBean

```java
public void afterPropertiesSet() throws Exception {
    if (getProvider() == null) {
        Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
        if (providerConfigMap != null && providerConfigMap.size() > 0) {
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            if ((protocolConfigMap == null || protocolConfigMap.size() == 0)
                    && providerConfigMap.size() > 1) { // backward compatibility
                List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() != null && config.isDefault().booleanValue()) {
                        providerConfigs.add(config);
                    }
                }
                if (!providerConfigs.isEmpty()) {
                    setProviders(providerConfigs);
                }
            } else {
                ProviderConfig providerConfig = null;
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (providerConfig != null) {
                            throw new IllegalStateException("Duplicate provider configs: " + providerConfig + " and " + config);
                        }
                        providerConfig = config;
                    }
                }
                if (providerConfig != null) {
                    setProvider(providerConfig);
                }
            }
        }
    }
    if (getApplication() == null
            && (getProvider() == null || getProvider().getApplication() == null)) {
        //"demo-provider" -> "<dubbo:application name="demo-provider" id="demo-provider" />"
        Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
        if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
            ApplicationConfig applicationConfig = null;
            for (ApplicationConfig config : applicationConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (applicationConfig != null) {
                        throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                    }
                    applicationConfig = config;
                }
            }
            if (applicationConfig != null) {
                setApplication(applicationConfig);
            }
        }
    }
    if (getModule() == null
            && (getProvider() == null || getProvider().getModule() == null)) {
        Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
        if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
            ModuleConfig moduleConfig = null;
            for (ModuleConfig config : moduleConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (moduleConfig != null) {
                        throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                    }
                    moduleConfig = config;
                }
            }
            if (moduleConfig != null) {
                setModule(moduleConfig);
            }
        }
    }
    //<dubbo:registry address="zookeeper://localhost:2181" id="com.alibaba.dubbo.config.RegistryConfig" />
    //
    if ((getRegistries() == null || getRegistries().isEmpty())
            && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries().isEmpty())
        //<dubbo:application name="demo-provider" id="demo-provider" />
            && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
        //"com.alibaba.dubbo.config.RegistryConfig" -> "<dubbo:registry address="zookeeper://localhost:2181" id="com.alibaba.dubbo.config.RegistryConfig" />"
        Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
        if (registryConfigMap != null && registryConfigMap.size() > 0) {
            List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
            for (RegistryConfig config : registryConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    registryConfigs.add(config);
                }
            }
            if (registryConfigs != null && !registryConfigs.isEmpty()) {
                super.setRegistries(registryConfigs);
            }
        }
    }
    
    if (getMonitor() == null
            && (getProvider() == null || getProvider().getMonitor() == null)
            && (getApplication() == null || getApplication().getMonitor() == null)) {
        Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
        if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
            MonitorConfig monitorConfig = null;
            for (MonitorConfig config : monitorConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (monitorConfig != null) {
                        throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                    }
                    monitorConfig = config;
                }
            }
            if (monitorConfig != null) {
                setMonitor(monitorConfig);
            }
        }
    }
    //dubbo:service没设置protocol，也没设置provider，或者设置了provider但是指向的dubbo:provider的protocols没设置
    if ((getProtocols() == null || getProtocols().isEmpty())
            && (getProvider() == null || getProvider().getProtocols() == null || getProvider().getProtocols().isEmpty())) {
        //拿"dubbo" -> "<dubbo:protocol name="dubbo" port="20880" id="dubbo" />"
        Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
        if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
            List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
            for (ProtocolConfig config : protocolConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    protocolConfigs.add(config);
                }
            }
            if (protocolConfigs != null && !protocolConfigs.isEmpty()) {
                super.setProtocols(protocolConfigs);
            }
        }
    }
    
    if (getPath() == null || getPath().length() == 0) {
        if (beanName != null && beanName.length() > 0
                && getInterface() != null && getInterface().length() > 0
                && beanName.startsWith(getInterface())) {
            //dubbo:service设置了path就用不然设置bean名为path com.alibaba.dubbo.demo.DemoService
            setPath(beanName);
        }
    }
    //先看dubbo:service有没有设置delay，没有则取dubbo:provider的delay，假如都没设置或者是-1则返回false
    if (!isDelay()) {
        //有delay则延迟delay秒调用doExport
        export();
    }
}
```

最后最重要的doExport

```java
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("Already unexported!");
    }
    if (exported) {
        return;
    }
    exported = true;
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
    }
    checkDefault();
    if (provider != null) {
        if (application == null) {
            application = provider.getApplication();
        }
        if (module == null) {
            module = provider.getModule();
        }
        if (registries == null) {
            registries = provider.getRegistries();
        }
        if (monitor == null) {
            monitor = provider.getMonitor();
        }
        if (protocols == null) {
            protocols = provider.getProtocols();
        }
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {
            monitor = module.getMonitor();
        }
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {
            monitor = application.getMonitor();
        }
    }
    if (ref instanceof GenericService) {
        interfaceClass = GenericService.class;
        if (StringUtils.isEmpty(generic)) {
            generic = Boolean.TRUE.toString();
        }
    } else {
        try {
            //这里是最重要的地方，反射
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        checkInterfaceAndMethods(interfaceClass, methods);
        checkRef();
        generic = Boolean.FALSE.toString();
    }
    if (local != null) {
        if ("true".equals(local)) {
            local = interfaceName + "Local";
        }
        Class<?> localClass;
        try {
            localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        if (!interfaceClass.isAssignableFrom(localClass)) {
            throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
        }
    }
    if (stub != null) {
        if ("true".equals(stub)) {
            stub = interfaceName + "Stub";
        }
        Class<?> stubClass;
        try {
            stubClass = ClassHelper.forNameWithThreadContextClassLoader(stub);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        if (!interfaceClass.isAssignableFrom(stubClass)) {
            throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
        }
    }
    checkApplication();
    checkRegistry();
    checkProtocol();
    appendProperties(this);
    checkStubAndMock(interfaceClass);
    //服务发现的path，缺省为接口名
    if (path == null || path.length() == 0) {
        path = interfaceName;
    }
    //生成url   
  //  dubbo://172.17.8.254:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.17.8.254&bind.port=20880&channel.readonly.sent=true&codec=dubbo&dubbo=2.0.0&generic=false&heartbeat=60000&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=6755&qos.port=22222&side=provider&timestamp=1522308929779
    //host由this.findConfigedHosts(protocolConfig, registryURLs, map); 取 dubbo:protocol host，没有的话取本地ip 并且设置到bind.ip
     // port由this.findConfigedPorts(protocolConfig, name, map);取 dubbo:protocol port，没有的话取本地ip 并且设置到bind.port
    //拼接registry:协议，registryURLs
    doExportUrls();
    //转化成ProviderModel注册
    ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
    ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}
```



doExportUrlsFor1Protocol部分重要源码

```java
    for (URL registryURL : registryURLs) {
        url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
        URL monitorUrl = loadMonitor(registryURL);
        if (monitorUrl != null) {
            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
        }
        //通过动态代理工厂生成实现类调用器Invoker, Invoker相当于动态代理类
        // private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
        //默认是JavassistProxyFactory
        //ref指的是service类这里就会被代理成一个Invoker类
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
//private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
        //ProtocolListenerWrapper->ProtocolFilterWrapper->RegistryProtocol，实际最后是DubboProtocol，这里是因为SPI，SPI比较复杂，后面阐述
        Exporter<?> exporter = protocol.export(wrapperInvoker);
        exporters.add(exporter);
    }
```

incubator-dubbo/dubbo-rpc/dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/com.alibaba.dubbo.rpc.ProxyFactory

```properties
stub=com.alibaba.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper
jdk=com.alibaba.dubbo.rpc.proxy.jdk.JdkProxyFactory
javassist=com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory
```

## proxy 服务代理层

### JavassistProxyFactory源码分析

```java
public class JavassistProxyFactory extends AbstractProxyFactory {
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // 包含'$'无法处理
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
}
```





```java
public abstract class AbstractProxyInvoker<T> implements Invoker<T> {

    private final T proxy;

    private final Class<T> type;

    private final URL url;

    public AbstractProxyInvoker(T proxy, Class<T> type, URL url) {
        if (proxy == null) {
            throw new IllegalArgumentException("proxy == null");
        }
        if (type == null) {
            throw new IllegalArgumentException("interface == null");
        }
        if (!type.isInstance(proxy)) {
            throw new IllegalArgumentException(proxy.getClass().getName() + " not implement interface " + type);
        }
        this.proxy = proxy;
        this.type = type;
        this.url = url;
    }

    public Class<T> getInterface() {
        return type;
    }

    public URL getUrl() {
        return url;
    }

    public boolean isAvailable() {
        return true;
    }

    public void destroy() {
    }

    public Result invoke(Invocation invocation) throws RpcException {
        try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    protected abstract Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable;

    @Override
    public String toString() {
        return getInterface() + " -> " + (getUrl() == null ? " " : getUrl().toString());
    }

}
```



```java
public class DelegateProviderMetaDataInvoker<T> implements Invoker {
    protected final Invoker<T> invoker;
    private ServiceConfig metadata;

    public DelegateProviderMetaDataInvoker(Invoker<T> invoker,ServiceConfig metadata) {
        this.invoker = invoker;
        this.metadata = metadata;
    }

    public Class<T> getInterface() {
        return invoker.getInterface();
    }

    public URL getUrl() {
        return invoker.getUrl();
    }

    public boolean isAvailable() {
        return invoker.isAvailable();
    }

    public Result invoke(Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }

    public void destroy() {
        invoker.destroy();
    }

    public ServiceConfig getMetadata() {
        return metadata;
    }
}
```

incubator-dubbo/dubbo-rpc/dubbo-rpc-dubbo/src/main/resources/META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol

```properties
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
```

## protocol 远程调用层

### DubboProtocol源码分析

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // url转化为com.alibaba.dubbo.demo.DemoService:20880
    String key = serviceKey(url);
    //DelegateProviderMetaDataInvoker封装到DubboExporter
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    //缓存起来，每次被调用的时候直接使用
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }
//server = Exchangers.bind(url, requestHandler);
    openServer(url);
    optimizeSerialization(url);
    return exporter;
}
```

* openServer最后是调用了`server = Exchangers.bind(url, requestHandler)`后文会详细说明

```java
public class DubboExporter<T> extends AbstractExporter<T> {

    private final String key;

    private final Map<String, Exporter<?>> exporterMap;

    public DubboExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
        super(invoker);
        this.key = key;
        this.exporterMap = exporterMap;
    }

    @Override
    public void unexport() {
        super.unexport();
        exporterMap.remove(key);
    }

}
```

关于SPI可以参考

http://www1350.github.io/#post/114



### Exporter

```java
public interface Exporter<T> {

    /**
     * get invoker.
     *
     * @return invoker
     */
    Invoker<T> getInvoker();

    /**
     * unexport.
     * <p>
     * <code>
     * getInvoker().destroy();
     * </code>
     */
    void unexport();

}
```

### Invoker

```java
public interface Invoker<T> extends Node {

    /**
     * get service interface.
     *
     * @return service interface.
     */
    Class<T> getInterface();

    /**
     * invoke.
     *
     * @param invocation
     * @return result
     * @throws RpcException
     */
    Result invoke(Invocation invocation) throws RpcException;

}
```



```java
public interface Node {

    /**
     * get url.
     *
     * @return url.
     */
    URL getUrl();

    /**
     * is available.
     *
     * @return available.
     */
    boolean isAvailable();

    /**
     * destroy.
     */
    void destroy();

}
```

下图阐述了这个过程，但是并不是直接调用DubboProtocol，而是RegistryProtocol，这个后面阐述。

![sequencediagram133](https://user-images.githubusercontent.com/7789698/38093235-e75677b8-339c-11e8-9165-58b6380f68bf.png)