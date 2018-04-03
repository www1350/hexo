---
title: dubbo源码解析（四）registry
abbrlink: ee087dca
date: 2018-04-03 22:44:16
tags:
categories:
---

registry层

通过dubbo解析一我们分析了整个ServiceConfig，留下了RegistryProtocol这个没有分析，接下来我们来分析分析RegistryProtocol

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

    //to judge to delay publish whether or not
    boolean register = registedProviderUrl.getParameter("register", true);

    //"com.alibaba.dubbo.demo.DemoService" -> ConcurrentHashSet<ConsumerInvokerWrapper>
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

    if (register) {
        //注册
        //ZookeeperRegistry(FailbackRegistry)
        // registry.register(registedProviderUrl);
        //registry.doRegister 下面讲
        register(registryUrl, registedProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
    // provider://172.17.8.254:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=10350&side=provider&timestamp=1522656496788
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    //订阅，最后调用overrideSubscribeListener#notify
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

注册

```java
protected void doRegister(URL url) {
    try {
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

订阅


```java
protected void doSubscribe(final URL url, final NotifyListener listener) {
    try {
        if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
            String root = toRootPath();
            ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
            if (listeners == null) {
                zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                listeners = zkListeners.get(url);
            }
            ChildListener zkListener = listeners.get(listener);
            if (zkListener == null) {
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
            zkClient.create(root, false);
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
        List<URL> matchedUrls = getMatchedUrls(urls, subscribeUrl);
        logger.debug("subscribe url: " + subscribeUrl + ", override urls: " + matchedUrls);
        // No matching results
        if (matchedUrls.isEmpty()) {
            return;
        }

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
        //Merged with this configuration
        URL newUrl = getConfigedInvokerUrl(configurators, originUrl);
        //暴露者url变更
        if (!currentUrl.equals(newUrl)) {
            RegistryProtocol.this.doChangeLocalExport(originInvoker, newUrl);
            logger.info("exported provider url changed, origin url: " + originUrl + ", old export url: " + currentUrl + ", new export url: " + newUrl);
        }
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