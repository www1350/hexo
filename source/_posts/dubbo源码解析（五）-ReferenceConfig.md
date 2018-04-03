---
title: dubbo源码解析（五） ReferenceConfig
abbrlink: 26e556d8
date: 2018-04-03 22:44:25
tags:
categories:
---

分析完了服务提供者，紧接着分析消费者。

从ReferenceBean入手

## Config层

### ReferenceBean

```java
public void afterPropertiesSet() throws Exception {
   //init consumerConfig
    //...
   
    //init applicationConfig
    //init moduleConfig
    //init registryConfigs
    //init monitorConfig
    Boolean b = isInit();
    if (b == null && getConsumer() != null) {
        b = getConsumer().isInit();
    }
    if (b != null && b.booleanValue()) {
        //referenceConfig#get
        //init
        getObject();
    }
}
```

核心点在于getObject，其他都是一些config的初始化

```java
private void init() {
   //...
    checkApplication();
    checkStubAndMock(interfaceClass);
    Map<String, String> map = new HashMap<String, String>();
    Map<Object, Object> attributes = new HashMap<Object, Object>();
    map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }
    if (!isGeneric()) {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("NO method found in service interface " + interfaceClass.getName());
            map.put("methods", Constants.ANY_VALUE);
        } else {
            map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
    map.put(Constants.INTERFACE_KEY, interfaceName);
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, consumer, Constants.DEFAULT_KEY);
    appendParameters(map, this);
    String prefix = StringUtils.getServiceKey(map);
    if (methods != null && !methods.isEmpty()) {
        for (MethodConfig method : methods) {
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            appendAttributes(attributes, method, prefix + "." + method.getName());
            checkAndConvertImplicitConfig(method, map, attributes);
        }
    }

    String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
    if (hostToRegistry == null || hostToRegistry.length() == 0) {
        hostToRegistry = NetUtils.getLocalHost();
    } else if (isInvalidLocalHost(hostToRegistry)) {
        throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
    }
    map.put(Constants.REGISTER_IP_KEY, hostToRegistry);

    //attributes are stored by system context.
    StaticContext.getSystemContext().putAll(attributes);
    //创建代理类
    //FailoverClusterInvoker
    ref = createProxy(map);
    ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
    ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
}
```

核心点在于createProxy

```java
private T createProxy(Map<String, String> map) {
    URL tmpUrl = new URL("temp", "localhost", 0, map);
    //处理isJvmRefer
    //...

    if (isJvmRefer) {
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        invoker = refprotocol.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
            String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (url.getPath() == null || url.getPath().length() == 0) {
                        url = url.setPath(interfaceName);
                    }
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else { // assemble URL from register center's configuration
            //RegistryConfig读取配置转化为url
            // registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=10852&qos.port=33333&refer=application=demo-consumer&check=false&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=10852&qos.port=33333&register.ip=172.17.8.254&side=consumer&timestamp=1522667530807&registry=zookeeper&timestamp=1522667559837
            List<URL> us = loadRegistries(false);
            if (us != null && !us.isEmpty()) {
                for (URL u : us) {
                    URL monitorUrl = loadMonitor(u);
                    if (monitorUrl != null) {
                        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                    }
                    //   zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=10852&qos.port=33333&refer=application=demo-consumer&check=false&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=10852&qos.port=33333&register.ip=172.17.8.254&side=consumer&timestamp=1522667530807&timestamp=1522667559837
                    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                }
            }
            if (urls == null || urls.isEmpty()) {
                throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
            }
        }
//如果只有一服务提供者，直接协议引用获取invoker
        if (urls.size() == 1) {
            //ProtocolFilterWrapper->ProtocolListenerWrapper->RegistryProtocol#refer
            //返回FailoverClusterInvoker
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
        } else {
            //有多个需要使用负载算法负载出相应的invoker
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            for (URL url : urls) {
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url; // use last registry url
                }
            }
            if (registryURL != null) { // registry url is available
                // use AvailableCluster only when register's cluster is available
                URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                //AvailableCluster 
                invoker = cluster.join(new StaticDirectory(u, invokers));
            } else { // not a registry url
                //FailoverCluster
                invoker = cluster.join(new StaticDirectory(invokers));
            }
        }
    }

    Boolean c = check;
    if (c == null && consumer != null) {
        c = consumer.isCheck();
    }
    if (c == null) {
        c = true; // default true
    }
    if (c && !invoker.isAvailable()) {
        throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
    }
    if (logger.isInfoEnabled()) {
        logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
    }
    // create service proxy
    //JavassistProxyFactory
    return (T) proxyFactory.getProxy(invoker);
}
```

## protocol层

RegistryProtocol

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    //ZookeeperRegistry
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // group="a,b" or group="*"
    //分组
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
                || "*".equals(group)) {
            return doRefer(getMergeableCluster(), registry, type, url);
        }
    }
    return doRefer(cluster, registry, type, url);
}
```



```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    //type->com.alibaba.dubbo.demo.DemoService
    //url->zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=10852&qos.port=33333&refer=application=demo-consumer&check=false&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=10852&qos.port=33333&register.ip=172.17.8.254&side=consumer&timestamp=1522667530807&timestamp=1522667559837
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    //设置DubboProtocol
    directory.setProtocol(protocol);
    // all attributes of REFER_KEY
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }
    //订阅RegistryDirectory#subscrib
    //调用ZookeeperRegistry(FailbackRegistry)#subscribe->doSubscribe
    //上一文已经解析过,但是注意的是，这里notify是调用RegistryDirectory的
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
            Constants.PROVIDERS_CATEGORY
                    + "," + Constants.CONFIGURATORS_CATEGORY
                    + "," + Constants.ROUTERS_CATEGORY));

    //负载
    //(MockClusterWrapper)->FailoverCluster
    //返回(MockClusterInvoker)->FailoverClusterInvoker
    Invoker invoker = cluster.join(directory);
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```



zk事件变化的通知

RegistryDirectory

```java
public synchronized void notify(List<URL> urls) {
    List<URL> invokerUrls = new ArrayList<URL>();
    List<URL> routerUrls = new ArrayList<URL>();
    List<URL> configuratorUrls = new ArrayList<URL>();
    for (URL url : urls) {
        String protocol = url.getProtocol();
        String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
        if (Constants.ROUTERS_CATEGORY.equals(category)
                || Constants.ROUTE_PROTOCOL.equals(protocol)) {
            routerUrls.add(url);
        } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
            configuratorUrls.add(url);
        } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
            invokerUrls.add(url);
        } else {
            logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
        }
    }
    // configurators
    if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
        this.configurators = toConfigurators(configuratorUrls);
    }
    // routers
    if (routerUrls != null && !routerUrls.isEmpty()) {
        List<Router> routers = toRouters(routerUrls);
        if (routers != null) { // null - do nothing
            setRouters(routers);
        }
    }
    List<Configurator> localConfigurators = this.configurators; // local reference
    // merge override parameters
    this.overrideDirectoryUrl = directoryUrl;
    if (localConfigurators != null && !localConfigurators.isEmpty()) {
        for (Configurator configurator : localConfigurators) {
            this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
        }
    }
    // 主要功能是缓存起来methodInvokerMap->RegistryDirectory$InvokerDelegate(DubboInvoker)
    refreshInvoker(invokerUrls);
}
```

refreshInvoker里面包括toInvokers和toMethodInvokers两个方法，使用refreshInvoker方法后将把url对应的服务提供者的invoker全都缓存到RegistryDirectory类的methodInvokerMap，后续当使用FailoverCluster失效转移的时候将会使用这个map并使用相应负载算法负载出一个invoker。

```java
private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
    Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<String, Invoker<T>>();
    if (urls == null || urls.isEmpty()) {
        return newUrlInvokerMap;
    }
    Set<String> keys = new HashSet<String>();
    String queryProtocols = this.queryMap.get(Constants.PROTOCOL_KEY);
    for (URL providerUrl : urls) {
        // If protocol is configured at the reference side, only the matching protocol is selected
        if (queryProtocols != null && queryProtocols.length() > 0) {
            boolean accept = false;
            String[] acceptProtocols = queryProtocols.split(",");
            for (String acceptProtocol : acceptProtocols) {
                if (providerUrl.getProtocol().equals(acceptProtocol)) {
                    accept = true;
                    break;
                }
            }
            if (!accept) {
                continue;
            }
        }
        if (Constants.EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
            continue;
        }
        if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
            logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() + " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost()
                    + ", supported protocol: " + ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
            continue;
        }
        URL url = mergeUrl(providerUrl);

        String key = url.toFullString(); // The parameter urls are sorted
        if (keys.contains(key)) { // Repeated url
            continue;
        }
        keys.add(key);
        // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
        Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
        Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
        if (invoker == null) { // Not in the cache, refer again
            try {
                boolean enabled = true;
                if (url.hasParameter(Constants.DISABLED_KEY)) {
                    enabled = !url.getParameter(Constants.DISABLED_KEY, false);
                } else {
                    enabled = url.getParameter(Constants.ENABLED_KEY, true);
                }
                if (enabled) {
                    //ProtocolFilterWrapper->ProtocolListenerWrapper->DubboProtocal#refer
                    //返回DubboInvoker
                    //RegistryDirectory$InvokerDelegate
                    invoker = new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);
                }
            } catch (Throwable t) {
                logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
            }
            if (invoker != null) { // Put new invoker in cache
                newUrlInvokerMap.put(key, invoker);
            }
        } else {
            newUrlInvokerMap.put(key, invoker);
        }
    }
    keys.clear();
    return newUrlInvokerMap;
}
```



```java
private Map<String, List<Invoker<T>>> toMethodInvokers(Map<String, Invoker<T>> invokersMap) {
    Map<String, List<Invoker<T>>> newMethodInvokerMap = new HashMap<String, List<Invoker<T>>>();
    // According to the methods classification declared by the provider URL, the methods is compatible with the registry to execute the filtered methods
    List<Invoker<T>> invokersList = new ArrayList<Invoker<T>>();
    if (invokersMap != null && invokersMap.size() > 0) {
        for (Invoker<T> invoker : invokersMap.values()) {
            String parameter = invoker.getUrl().getParameter(Constants.METHODS_KEY);
            if (parameter != null && parameter.length() > 0) {
                String[] methods = Constants.COMMA_SPLIT_PATTERN.split(parameter);
                if (methods != null && methods.length > 0) {
                    for (String method : methods) {
                        if (method != null && method.length() > 0
                                && !Constants.ANY_VALUE.equals(method)) {
                            List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
                            if (methodInvokers == null) {
                                methodInvokers = new ArrayList<Invoker<T>>();
                                newMethodInvokerMap.put(method, methodInvokers);
                            }
                            methodInvokers.add(invoker);
                        }
                    }
                }
            }
            invokersList.add(invoker);
        }
    }
    //通过MockInvokersSelector
    List<Invoker<T>> newInvokersList = route(invokersList, null);
    newMethodInvokerMap.put(Constants.ANY_VALUE, newInvokersList);
    if (serviceMethods != null && serviceMethods.length > 0) {
        for (String method : serviceMethods) {
            List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
            if (methodInvokers == null || methodInvokers.isEmpty()) {
                methodInvokers = newInvokersList;
            }
            newMethodInvokerMap.put(method, route(methodInvokers, method));
        }
    }
    // sort and unmodifiable
    for (String method : new HashSet<String>(newMethodInvokerMap.keySet())) {
        List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
        Collections.sort(methodInvokers, InvokerComparator.getComparator());
        newMethodInvokerMap.put(method, Collections.unmodifiableList(methodInvokers));
    }
    return Collections.unmodifiableMap(newMethodInvokerMap);
}
```

## cluster层

```java
@SPI(FailoverCluster.NAME)
public interface Cluster {

    /**
     * Merge the directory invokers to a virtual invoker.
     *
     * @param <T>
     * @param directory
     * @return cluster invoker
     * @throws RpcException
     */
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;

}
```

FailoverCluster把RegistryDirectory引用放进去

```java
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }

}
```