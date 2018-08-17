---
title: dubbo源码解析（四）registry
abbrlink: ee087dca
date: 2018-04-03 22:44:16
tags: [dubbo,rpc]
categories: [中间件,源码]
---

registry层

通过dubbo解析一我们分析了整个ServiceConfig，留下了RegistryProtocol这个没有分析，接下来我们来分析分析RegistryProtocol

![reg333321](https://user-images.githubusercontent.com/7789698/42802903-e087d48c-89d6-11e8-820b-99af7de93450.jpg)

在ServiceConfig的loadRegistries方法里面，当spring容器初始化，服务提供方（消费方也差不多）暴露过程，就会拼接生成registryURL

registryURL:

*registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=9966&qos.port=22222&registry=zookeeper&timestamp=1522650082230*

<!-- more -->

RegistryProtocol

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    URL registryUrl = getRegistryUrl(originInvoker);

    //获取registry provider
    //return registryFactory.getRegistry(registryUrl);
    //zookeeper=com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistryFactory
    //ZookeeperRegistryFactory.getRegistry
    //最后返回ZookeeperRegistry
    final Registry registry = getRegistry(originInvoker);
    //获取url，去除monitor、bind.ip、bind.port、qos.enable、qos.port、qos.accept.foreign.ip
    //zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&export=dubbo://172.17.8.254:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.17.8.254&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=10350&qos.port=22222&side=provider&timestamp=1522656496788&pid=10350&qos.port=22222&timestamp=1522656486762
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);

    //是否延迟发布
    boolean register = registedProviderUrl.getParameter("register", true);

    //"com.alibaba.dubbo.demo.DemoService" -> ConcurrentHashSet<ConsumerInvokerWrapper>
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

    if (register) {
        //立马注册
        //ZookeeperRegistry(FailbackRegistry)
        // registry.register(registedProviderUrl);
        //registry.doRegister 下面讲
        register(registryUrl, registedProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
    //加category=configurators&check=false
    // provider://172.17.8.254:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=10350&side=provider&timestamp=1522656496788
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    //订阅，最后调用overrideSubscribeListener#notify
    //url和listener进行绑定订阅
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //Ensure that a new exporter instance is returned every time export
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
}
```





```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
    //去掉dynamic、enabled两参数得到url作为key
    String key = getCacheKey(originInvoker);
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                //调用DubboProtocol.export
                //DubboExporter放入ExporterChangeableWrapper
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
                bounds.put(key, exporter);
            }
        }
    }
    return exporter;
}
```



protocol和registryFactory这里又是啥呢？我们看下

```java
    public static RegistryProtocol getRegistryProtocol() {
        if (INSTANCE == null) {
            ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(Constants.REGISTRY_PROTOCOL); // load
        }
        return INSTANCE;
    }

//因为含有set开头的public，会被注入Protocol，而默认的是dubbo也就是DubboProtocol
    public void setProtocol(Protocol protocol) {
        this.protocol = protocol;
    }

//因为含有set开头的public，会注入registryFactory，也就是ZookeeperRegistryFactory
    public void setRegistryFactory(RegistryFactory registryFactory) {
        this.registryFactory = registryFactory;
    }
```



```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {

    private ZookeeperTransporter zookeeperTransporter;

    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }

    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }

}
```

![image](https://user-images.githubusercontent.com/7789698/38189092-202df524-3691-11e8-9445-0be328b6b5c4.png)

ZookeeperRegistry

```java
private final ZookeeperClient zkClient;

public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
    if (!group.startsWith(Constants.PATH_SEPARATOR)) {
        group = Constants.PATH_SEPARATOR + group;
    }
    //有group则root是/group 否则就是/dubbo
    this.root = group;
    zkClient = zookeeperTransporter.connect(url);
    zkClient.addStateListener(new StateListener() {
        public void stateChanged(int state) {
            if (state == RECONNECTED) {
                try {
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
}
```

注册,父类的register最终调用了子类的doRegister

```java
protected void doRegister(URL url) {
    try {
        //根据路径创建结点，dynamic为true就是临时结点否则就是实体结点
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}

//路径
//  /dubbo/com.alibaba.dubbo.demo.DemoService/providers/dubbo://172.17.8.254:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=10350&side=provider&timestamp=1522656496788
private String toUrlPath(URL url) {
    return toCategoryPath(url) + Constants.PATH_SEPARATOR + URL.encode(url.toFullString());
}
```

订阅，registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);，先缓存到

```java
ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
```

里面最终调用了doSubscribe


```java
protected void doSubscribe(final URL url, final NotifyListener listener) {
    try {
        //如果要订阅的服务是 *
        if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
            // 拿到root， /group 或/dubbo
            String root = toRootPath();
            ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
            if (listeners == null) {
                zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                listeners = zkListeners.get(url);
            }
            ChildListener zkListener = listeners.get(listener);
            if (zkListener == null) {
                　　//添加监听子节点变化事件，变化则子节点调用订阅逻辑
                listeners.putIfAbsent(listener, new ChildListener() {
                    public void childChanged(String parentPath, List<String> currentChilds) {
                        for (String child : currentChilds) {
                            child = URL.decode(child);
                            if (!anyServices.contains(child)) {
                                anyServices.add(child);
                                subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child,
                                        Constants.CHECK_KEY, String.valueOf(false)), listener);
                            }
                        }
                    }
                });
                zkListener = listeners.get(listener);
            }
            //创建root结点
            zkClient.create(root, false);
            // 监听zk上root子节点变化事件，变化则返回子节点名，也就是services名
            List<String> services = zkClient.addChildListener(root, zkListener);
            if (services != null && !services.isEmpty()) {
                for (String service : services) {
                    service = URL.decode(service);
                    anyServices.add(service);
                    subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service,
                            Constants.CHECK_KEY, String.valueOf(false)), listener);
                }
            }
        } else {
            List<URL> urls = new ArrayList<URL>();
            for (String path : toCategoriesPath(url)) {
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    //加watch
                    listeners.putIfAbsent(listener, new ChildListener() {
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                zkClient.create(path, false);
                //overrideSubscribeListener注册到结点上
                List<String> children = zkClient.addChildListener(path, zkListener);
                if (children != null) {
                    urls.addAll(toUrlsWithEmpty(url, path, children));
                }
            }
            //通知
            notify(url, listener, urls);
        }
    } catch (Throwable e) {
        throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```



override协议：

1. 禁用提供者：(通常用于临时踢除某台提供者机器，相似的，禁止消费者访问请使用路由规则)

   ```
   override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&disbaled=true
   ```

2. 调整权重：(通常用于容量评估，缺省权重为 100)

   ```
   override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&weight=200
   ```

3. 调整负载均衡策略：(缺省负载均衡策略为 random)

   ```
   override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&loadbalance=leastactive
   ```

4. 服务降级：(通常用于临时屏蔽某个出错的非关键服务)

   ```
   override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null
   ```

0.0.0.0表示所有的host

OverrideListener

```java
private class OverrideListener implements NotifyListener {

    private final URL subscribeUrl;
    private final Invoker originInvoker;

    public OverrideListener(URL subscribeUrl, Invoker originalInvoker) {
        this.subscribeUrl = subscribeUrl;
        this.originInvoker = originalInvoker;
    }

    /**
     * @param urls The list of registered information , is always not empty, The meaning is the same as the return value of {@link com.alibaba.dubbo.registry.RegistryService#lookup(URL)}.
     */
    public synchronized void notify(List<URL> urls) {
        logger.debug("original override urls: " + urls);
        //判断提供者消费者几个参数是否匹配，1、接口名 2、提供者消费者category是否一致 3、group 4、version 5、classifier 
        //拿到所有匹配的提供者url
        List<URL> matchedUrls = getMatchedUrls(urls, subscribeUrl);
        logger.debug("subscribe url: " + subscribeUrl + ", override urls: " + matchedUrls);
        // No matching results
        if (matchedUrls.isEmpty()) {
            return;
        }

        // override协议
        List<Configurator> configurators = RegistryDirectory.toConfigurators(matchedUrls);

        final Invoker<?> invoker;
        if (originInvoker instanceof InvokerDelegete) {
            invoker = ((InvokerDelegete<?>) originInvoker).getInvoker();
        } else {
            invoker = originInvoker;
        }
        //The origin invoker
        URL originUrl = RegistryProtocol.this.getProviderUrl(invoker);
        String key = getCacheKey(originInvoker);
        ExporterChangeableWrapper<?> exporter = bounds.get(key);
        if (exporter == null) {
            logger.warn(new IllegalStateException("error state, exporter should not be null"));
            return;
        }
        //The current, may have been merged many times
        URL currentUrl = exporter.getInvoker().getUrl();
        //合并configuratorUrls 中的属性   合并override协议和absent协议，
        URL newUrl = getConfigedInvokerUrl(configurators, originUrl);
        //暴露者url变更
        if (!currentUrl.equals(newUrl)) {
            RegistryProtocol.this.doChangeLocalExport(originInvoker, newUrl);
            logger.info("exported provider url changed, origin url: " + originUrl + ", old export url: " + currentUrl + ", new export url: " + newUrl);
        }
    }
```

合并override协议和absent协议过程：

List<Configurator> configurators = RegistryDirectory.toConfigurators(matchedUrls);    先获取到所有的Configurator

```java
public static List<Configurator> toConfigurators(List<URL> urls) {
    if (urls == null || urls.isEmpty()) {
        return Collections.emptyList();
    }

    List<Configurator> configurators = new ArrayList<Configurator>(urls.size());
    for (URL url : urls) {
        //包含empty协议，返回空
        if (Constants.EMPTY_PROTOCOL.equals(url.getProtocol())) {
            configurators.clear();
            break;
        }
        Map<String, String> override = new HashMap<String, String>(url.getParameters());
        // 去掉anyhost参数 override://ip:port...?anyhost=true
        override.remove(Constants.ANYHOST_KEY);
        if (override.size() == 0) {
            configurators.clear();
            continue;
        }
        //根据url协议生成 override->OverrideConfigurator absent->AbsentConfigurator
        configurators.add(configuratorFactory.getConfigurator(url));
    }
    //按host、priority排序
    Collections.sort(configurators);
    return configurators;
}
```



原始url和Configurators进行合并

```java
private URL getConfigedInvokerUrl(List<Configurator> configurators, URL url) {
    for (Configurator configurator : configurators) {
        url = configurator.configure(url);
    }
    return url;
}
```



```java
public URL configure(URL url) {
//判空
    if (configuratorUrl == null || configuratorUrl.getHost() == null
            || url == null || url.getHost() == null) {
        return url;
    }
    // 有端口号且相同configuratorUrl->url
    if (configuratorUrl.getPort() != 0) {
        if (url.getPort() == configuratorUrl.getPort()) {
            return configureIfMatch(url.getHost(), url);
        }
    } else {
        //如果是消费端
        if (url.getParameter(Constants.SIDE_KEY, Constants.PROVIDER).equals(Constants.CONSUMER)) {
            //本地host、url
            return configureIfMatch(NetUtils.getLocalHost(), url);
         //提供端
        } else if (url.getParameter(Constants.SIDE_KEY, Constants.CONSUMER).equals(Constants.PROVIDER)) {
            //0.0.0.0（影响所有提供端）、url
            return configureIfMatch(Constants.ANYHOST_VALUE, url);
        }
    }
    return url;
}
```



```java
private URL configureIfMatch(String host, URL url) {
    //配置为0.0.0.0或者host相同，意味着要合并url
    if (Constants.ANYHOST_VALUE.equals(configuratorUrl.getHost()) || host.equals(configuratorUrl.getHost())) {
        //获取override里面的application
        String configApplication = configuratorUrl.getParameter(Constants.APPLICATION_KEY,
                configuratorUrl.getUsername());
        //获取当前url的application
        String currentApplication = url.getParameter(Constants.APPLICATION_KEY, url.getUsername());
        //application一样
        if (configApplication == null || Constants.ANY_VALUE.equals(configApplication)
                || configApplication.equals(currentApplication)) {
            Set<String> condtionKeys = new HashSet<String>();
            //category、check、dynamic、enabled
            condtionKeys.add(Constants.CATEGORY_KEY);
            condtionKeys.add(Constants.CHECK_KEY);
            condtionKeys.add(Constants.DYNAMIC_KEY);
            condtionKeys.add(Constants.ENABLED_KEY);
            for (Map.Entry<String, String> entry : configuratorUrl.getParameters().entrySet()) {
                String key = entry.getKey();
                String value = entry.getValue();
                if (key.startsWith("~") || Constants.APPLICATION_KEY.equals(key) || Constants.SIDE_KEY.equals(key)) {
                    condtionKeys.add(key);
                    if (value != null && !Constants.ANY_VALUE.equals(value)
                            && !value.equals(url.getParameter(key.startsWith("~") ? key.substring(1) : key))) {
                        return url;
                    }
                }
            }
            //移除category、check、dynamic、enabled，如果是OverrideConfigurator就合并配置，AbsentConfigurator就只添加没有的配置
            return doConfigure(url, configuratorUrl.removeParameters(condtionKeys));
        }
    }
    return url;
}
```



url变更了，doChangeLocalExport重新暴露

```java
private <T> void doChangeLocalExport(final Invoker<T> originInvoker, URL newInvokerUrl) {
    String key = getCacheKey(originInvoker);
    final ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        logger.warn(new IllegalStateException("error state, exporter should not be null"));
    } else {
        final Invoker<T> invokerDelegete = new InvokerDelegete<T>(originInvoker, newInvokerUrl);
        exporter.setExporter(protocol.export(invokerDelegete));
    }
}
```

/Users/wangwenwei/.dubbo/dubbo-registry-demo-provider-localhost:2181.cache

当doSubscribe订阅失败的时候才会拿cache的内容

```java
private void saveProperties(URL url) {
    if (file == null) {
        return;
    }

    try {
        StringBuilder buf = new StringBuilder();
        Map<String, List<URL>> categoryNotified = notified.get(url);
        if (categoryNotified != null) {
            for (List<URL> us : categoryNotified.values()) {
                for (URL u : us) {
                    if (buf.length() > 0) {
                        buf.append(URL_SEPARATOR);
                    }
                    buf.append(u.toFullString());
                }
            }
        }
        properties.setProperty(url.getServiceKey(), buf.toString());
        long version = lastCacheChanged.incrementAndGet();
        //同步写入
        if (syncSaveFile) {
            doSaveProperties(version);
        } else {
            registryCacheExecutor.execute(new SaveProperties(version));
        }
    } catch (Throwable t) {
        logger.warn(t.getMessage(), t);
    }
}
```



```java
public void doSaveProperties(long version) {
    if (version < lastCacheChanged.get()) {
        return;
    }
    if (file == null) {
        return;
    }
    // Save
    try {
        //添加文件锁lock
        File lockfile = new File(file.getAbsolutePath() + ".lock");
        if (!lockfile.exists()) {
            lockfile.createNewFile();
        }
        RandomAccessFile raf = new RandomAccessFile(lockfile, "rw");
        try {
            FileChannel channel = raf.getChannel();
            try {
                FileLock lock = channel.tryLock();
                if (lock == null) {
                    throw new IOException("Can not lock the registry cache file " + file.getAbsolutePath() + ", ignore and retry later, maybe multi java process use the file, please config: dubbo.registry.file=xxx.properties");
                }
                // Save
                try {
                    if (!file.exists()) {
                        file.createNewFile();
                    }
                    FileOutputStream outputFile = new FileOutputStream(file);
                    try {
                        properties.store(outputFile, "Dubbo Registry Cache");
                    } finally {
                        outputFile.close();
                    }
                } finally {
                    lock.release();
                }
            } finally {
                channel.close();
            }
        } finally {
            raf.close();
        }
    } catch (Throwable e) {
        if (version < lastCacheChanged.get()) {
            return;
        } else {
            registryCacheExecutor.execute(new SaveProperties(lastCacheChanged.incrementAndGet()));
        }
        logger.warn("Failed to save registry store file, cause: " + e.getMessage(), e);
    }
}
```




## 注册中心扩展

### 扩展说明

负责服务的注册与发现。

### 扩展接口

- `com.alibaba.dubbo.registry.RegistryFactory`
- `com.alibaba.dubbo.registry.Registry`

### 扩展配置

```xml
<!-- 定义注册中心 -->
<dubbo:registry id="xxx1" address="xxx://ip:port" />
<!-- 引用注册中心，如果没有配置registry属性，将在ApplicationContext中自动扫描registry配置 -->
<dubbo:service registry="xxx1" />
<!-- 引用注册中心缺省值，当<dubbo:service>没有配置registry属性时，使用此配置 -->
<dubbo:provider registry="xxx1" />
```

### 扩展契约

RegistryFactory.java：

```java
public interface RegistryFactory {
    /**
     * 连接注册中心.
     * 
     * 连接注册中心需处理契约：<br>
     * 1. 当设置check=false时表示不检查连接，否则在连接不上时抛出异常。<br>
     * 2. 支持URL上的username:password权限认证。<br>
     * 3. 支持backup=10.20.153.10备选注册中心集群地址。<br>
     * 4. 支持file=registry.cache本地磁盘文件缓存。<br>
     * 5. 支持timeout=1000请求超时设置。<br>
     * 6. 支持session=60000会话超时或过期设置。<br>
     * 
     * @param url 注册中心地址，不允许为空
     * @return 注册中心引用，总不返回空
     */
    Registry getRegistry(URL url); 
}
```

RegistryService.java：

```java
public interface RegistryService { // Registry extends RegistryService 
    /**
     * 注册服务.
     * 
     * 注册需处理契约：<br>
     * 1. 当URL设置了check=false时，注册失败后不报错，在后台定时重试，否则抛出异常。<br>
     * 2. 当URL设置了dynamic=false参数，则需持久存储，否则，当注册者出现断电等情况异常退出时，需自动删除。<br>
     * 3. 当URL设置了category=overrides时，表示分类存储，缺省类别为providers，可按分类部分通知数据。<br>
     * 4. 当注册中心重启，网络抖动，不能丢失数据，包括断线自动删除数据。<br>
     * 5. 允许URI相同但参数不同的URL并存，不能覆盖。<br>
     * 
     * @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void register(URL url);

    /**
     * 取消注册服务.
     * 
     * 取消注册需处理契约：<br>
     * 1. 如果是dynamic=false的持久存储数据，找不到注册数据，则抛IllegalStateException，否则忽略。<br>
     * 2. 按全URL匹配取消注册。<br>
     * 
     * @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void unregister(URL url);

    /**
     * 订阅服务.
     * 
     * 订阅需处理契约：<br>
     * 1. 当URL设置了check=false时，订阅失败后不报错，在后台定时重试。<br>
     * 2. 当URL设置了category=overrides，只通知指定分类的数据，多个分类用逗号分隔，并允许星号通配，表示订阅所有分类数据。<br>
     * 3. 允许以interface,group,version,classifier作为条件查询，如：interface=com.alibaba.foo.BarService&version=1.0.0<br>
     * 4. 并且查询条件允许星号通配，订阅所有接口的所有分组的所有版本，或：interface=*&group=*&version=*&classifier=*<br>
     * 5. 当注册中心重启，网络抖动，需自动恢复订阅请求。<br>
     * 6. 允许URI相同但参数不同的URL并存，不能覆盖。<br>
     * 7. 必须阻塞订阅过程，等第一次通知完后再返回。<br>
     * 
     * @param url 订阅条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件监听器，不允许为空
     */
    void subscribe(URL url, NotifyListener listener);

    /**
     * 取消订阅服务.
     * 
     * 取消订阅需处理契约：<br>
     * 1. 如果没有订阅，直接忽略。<br>
     * 2. 按全URL匹配取消订阅。<br>
     * 
     * @param url 订阅条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件监听器，不允许为空
     */
    void unsubscribe(URL url, NotifyListener listener);

    /**
     * 查询注册列表，与订阅的推模式相对应，这里为拉模式，只返回一次结果。
     * 
     * @see com.alibaba.dubbo.registry.NotifyListener#notify(List)
     * @param url 查询条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @return 已注册信息列表，可能为空，含义同{@link com.alibaba.dubbo.registry.NotifyListener#notify(List<URL>)}的参数。
     */
    List<URL> lookup(URL url);

}
```

NotifyListener.java：

```java
public interface NotifyListener { 
    /**
     * 当收到服务变更通知时触发。
     * 
     * 通知需处理契约：<br>
     * 1. 总是以服务接口和数据类型为维度全量通知，即不会通知一个服务的同类型的部分数据，用户不需要对比上一次通知结果。<br>
     * 2. 订阅时的第一次通知，必须是一个服务的所有类型数据的全量通知。<br>
     * 3. 中途变更时，允许不同类型的数据分开通知，比如：providers, consumers, routes, overrides，允许只通知其中一种类型，但该类型的数据必须是全量的，不是增量的。<br>
     * 4. 如果一种类型的数据为空，需通知一个empty协议并带category参数的标识性URL数据。<br>
     * 5. 通知者(即注册中心实现)需保证通知的顺序，比如：单线程推送，队列串行化，带版本对比。<br>
     * 
     * @param urls 已注册信息列表，总不为空，含义同{@link com.alibaba.dubbo.registry.RegistryService#lookup(URL)}的返回值。
     */
    void notify(List<URL> urls);

}
```

### 已知扩展

`com.alibaba.dubbo.registry.support.dubbo.DubboRegistryFactory`

#### 扩展示例

Maven 项目结构：

```
src
 |-main
    |-java
        |-com
            |-xxx
                |-XxxRegistryFactory.java (实现RegistryFactory接口)
                |-XxxRegistry.java (实现Registry接口)
    |-resources
        |-META-INF
            |-dubbo
                |-com.alibaba.dubbo.registry.RegistryFactory (纯文本文件，内容为：xxx=com.xxx.XxxRegistryFactory)
```

XxxRegistryFactory.java：

```java
package com.xxx;

import com.alibaba.dubbo.registry.RegistryFactory;
import com.alibaba.dubbo.registry.Registry;
import com.alibaba.dubbo.common.URL;

public class XxxRegistryFactory implements RegistryFactory {
    public Registry getRegistry(URL url) {
        return new XxxRegistry(url);
    }
}
```

XxxRegistry.java：

```java
package com.xxx;

import com.alibaba.dubbo.registry.Registry;
import com.alibaba.dubbo.registry.NotifyListener;
import com.alibaba.dubbo.common.URL;

public class XxxRegistry implements Registry {
    public void register(URL url) {
        // ...
    }
    public void unregister(URL url) {
        // ...
    }
    public void subscribe(URL url, NotifyListener listener) {
        // ...
    }
    public void unsubscribe(URL url, NotifyListener listener) {
        // ...
    }
}
```

META-INF/dubbo/com.alibaba.dubbo.registry.RegistryFactory：

```properties
xxx=com.xxx.XxxRegistryFactory
```




比如dubbo官方给出的基于redis的注册方式

```properties
redis=com.alibaba.dubbo.registry.redis.RedisRegistryFactory
```

```java
public class RedisRegistryFactory extends AbstractRegistryFactory {
    @Override
    protected Registry createRegistry(URL url) {
        return new RedisRegistry(url);
    }
}
```

RedisRegistry就不具体分析了，注册的时候主要使用hset放入所有url、publish通知订阅者要订阅，订阅的时候使用hgetAll获取所有url、psubscribe进行订阅通知