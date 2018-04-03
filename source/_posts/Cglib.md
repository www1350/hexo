---
title: Cglib
abbrlink: 29cef8e0
date: 2018-04-03 22:40:21
tags:
categories:
---

cglib（字节码生成库）是一个生成和转化Java字节码的高级api。被使用在AOP上。在实现内部，CGLIB库使用了ASM这一个轻量但高性能的字节码操作框架来转化字节码，产生新类

Enhancer是cglib一个很重要的类。Enhancer动态创建一个子类。

Enhancer只能在java字节码级别构造方法，但是不能构造static或者final类。


```
public Object createProxy(Class targetClass) {
     Enhancer enhancer = new Enhancer();
     enhancer.setSuperclass(targetClass);
     enhancer.setCallback(NoOp.INSTANCE);
     return enhancer.create();
}
```
在例子中，默认的无参构造方法被使用来创建目标对象。如果你希望CGLIB创建一个有参数的实例，你应该使用net.sf.cglib.proxy.Enhancer.create(Class[], Object[])。NoOp是内置的一个类，可以看下源码

```
public interface NoOp extends Callback {
    NoOp INSTANCE = new NoOp() {
    };
}
```




```
public class SampleClass {
  public String test(String input) {
    return "Hello world!";
  }
}
```

```
@Test
public void testFixedValue() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new FixedValue() {
    @Override
    public Object loadObject() throws Exception {
      return "Hello cglib!";
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
}
```

上面的方式，任何方法都会被代理。比如proxy.toString()会返回"Hello cglib!" ，proxy.hashCode()则会抛出ClassCastException异常因为hashcode需要整数。此外final方法不会被拦截。Object#getClass 会返回类似 "SampleClass$$EnhancerByCGLIB$$e277c63c"





```
@Test
public void testInvocationHandler() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new InvocationHandler() {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable {
      if(method.getDeclaringClass() != Object.class && method.getReturnType() == String.class) {
        return "Hello cglib!";
      } else {
        throw new RuntimeException("Do not know what to do.");
      }
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
}
```

所有调用方法将会分发到相同的InvocationHandler可能会导致死循环.


```
@Test
public void testMethodInterceptor() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)
        throws Throwable {
      if(method.getDeclaringClass() != Object.class && method.getReturnType() == String.class) {
        return "Hello cglib!";
      } else {
        return proxy.invokeSuper(obj, args);
      }
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
  proxy.hashCode(); // Does not throw an exception or result in an endless loop.
}
```

MethodInterceptor的创建和链接需要生成不同类型的字节码和一些不需要Invocationhandler就能产生的运行对象

> LazyLoader: Even though the LazyLoader's only method has the same method signature as FixedValue, the LazyLoader is fundamentally different to the FixedValue interceptor. The LazyLoader is actually supposed to return an instance of a subclass of the enhanced class. This instance is requested only when a method is called on the enhanced object and then stored for future invocations of the generated proxy. This makes sense if your object is expensive in its creation without knowing if the object will ever be used. Be aware that some constructor of the enhanced class must be called both for the proxy object and for the lazily loaded object. Thus, make sure that there is another cheap (maybe protected) constructor available or use an interface type for the proxy. You can choose the invoked constructed by supplying arguments to Enhancer#create(Object...).在被代理对象需要懒加载场景下非常有用，如果被代理对象加载完成，那么在以后的代理调用时会重复使用。
>
> Dispatcher: The Dispatcher is like the LazyLoader but will be invoked on every method call without storing the loaded object. This allows to change the implementation of a class without changing the reference to it. Again, be aware that some constructor must be called for both the proxy and the generated objects.与net.sf.cglib.proxy.LazyLoader差不多，但每次调用代理方法时都会调用loadObject方法来加载被代理对象。
>
> ProxyRefDispatcher: This class carries a reference to the proxy object it is invoked from in its signature. This allows for example to delegate method calls to another method of this proxy. Be aware that this can easily cause an endless loop and will always cause an endless loop if the same method is called from within ProxyRefDispatcher#loadObject(Object).与Dispatcher相同，但它的loadObject方法支持传入代理对象。
>
> NoOp: The NoOp class does not what its name suggests. Instead, it delegates each method call to the enhanced class's method implementation.




net.sf.cglib.proxy.CallbackFilter允许你在方法级别设置回调。

```
public class PersistenceServiceImpl implements PersistenceService {

    public void save(long id, String data) {
        System.out.println(data + " has been saved successfully.");
    }

    public String load(long id) {
        return "Jason Zhicheng Li";
    }
}
```
accept方法将代理方法映射到回调。方法返回值是一个回调对象数组中的下标
```
public class PersistenceServiceCallbackFilter implements CallbackFilter {

    //callback index for save method
    private static final int SAVE = 0;

    //callback index for load method
    private static final int LOAD = 1;

    /**
     * Specify which callback to use for the method being invoked.
     * @method the method being invoked.
     * @return the callback index in the callback array for this method
     */
    public int accept(Method method) {
        String name = method.getName();
        if ("save".equals(name)) {
            return SAVE;
        }
        // for other methods, including the load method, use the
        // second callback
        return LOAD;
    }
}
```

```
...
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(PersistenceServiceImpl.class);

CallbackFilter callbackFilter = new PersistenceServiceCallbackFilter();
enhancer.setCallbackFilter(callbackFilter);

AuthorizationService authorizationService = ...
Callback saveCallback = new AuthorizationInterceptor(authorizationService);
Callback loadCallback = NoOp.INSTANCE;
Callback[] callbacks = new Callback[]{saveCallback, loadCallback };
enhancer.setCallbacks(callbacks);
...
return (PersistenceServiceImpl)enhancer.create();
```



```
@Test
public void testCallbackFilter() throws Exception {
  Enhancer enhancer = new Enhancer();
  CallbackHelper callbackHelper = new CallbackHelper(SampleClass.class, new Class[0]) {
    @Override
    protected Object getCallback(Method method) {
      if(method.getDeclaringClass() != Object.class && method.getReturnType() == String.class) {
        return new FixedValue() {
          @Override
          public Object loadObject() throws Exception {
            return "Hello cglib!";
          };
        }
      } else {
        return NoOp.INSTANCE; // A singleton provided by NoOp.
      }
    }
  };
  enhancer.setSuperclass(MyClass.class);
  enhancer.setCallbackFilter(callbackHelper);
  enhancer.setCallbacks(callbackHelper.getCallbacks());
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
  proxy.hashCode(); // Does not throw an exception or result in an endless loop.
}
```



Bean generator

```
@Test
public void testBeanGenerator() throws Exception {
  BeanGenerator beanGenerator = new BeanGenerator();
  beanGenerator.addProperty("value", String.class);
  Object myBean = beanGenerator.create();
   
  Method setter = myBean.getClass().getMethod("setValue", String.class);
  setter.invoke(myBean, "Hello cglib!");
  Method getter = myBean.getClass().getMethod("getValue");
  assertEquals("Hello cglib!", getter.invoke(myBean));
}
```


Bean copier

```
public class OtherSampleBean {
  private String value;
  public String getValue() {
    return value;
  }
  public void setValue(String value) {
    this.value = value;
  }
}


@Test
public void testBeanCopier() throws Exception {
  BeanCopier copier = BeanCopier.create(SampleBean.class, OtherSampleBean.class, false);
  SampleBean bean = new SampleBean();
  bean.setValue("Hello cglib!");
  OtherSampleBean otherBean = new OtherSampleBean();
  copier.copy(bean, otherBean, null);
  assertEquals("Hello cglib!", otherBean.getValue()); 
}
```


Bulk bean


```
@Test
public void testBulkBean() throws Exception {
  BulkBean bulkBean = BulkBean.create(SampleBean.class,
      new String[]{"getValue"},
      new String[]{"setValue"},
      new Class[]{String.class});
  SampleBean bean = new SampleBean();
  bean.setValue("Hello world!");
  assertEquals(1, bulkBean.getPropertyValues(bean).length);
  assertEquals("Hello world!", bulkBean.getPropertyValues(bean)[0]);
  bulkBean.setPropertyValues(bean, new Object[] {"Hello cglib!"});
  assertEquals("Hello cglib!", bean.getValue());
}
```

Bean map

```
@Test
public void testBeanGenerator() throws Exception {
  SampleBean bean = new SampleBean();
  BeanMap map = BeanMap.create(bean);
  bean.setValue("Hello cglib!");
  assertEquals("Hello cglib", map.get("value"));
}
```


https://github.com/cglib/cglib/wiki/Tutorial


AOP源码如下～
```
// Configure CGLIB Enhancer...
Enhancer enhancer = createEnhancer();
if (classLoader != null) {
	enhancer.setClassLoader(classLoader);
	if (classLoader instanceof SmartClassLoader &&
	((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
		enhancer.setUseCache(false);
	}
}
enhancer.setSuperclass(proxySuperClass);
enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

Callback[] callbacks = getCallbacks(rootClass);
Class<?>[] types = new Class<?>[callbacks.length];
for (int x = 0; x < types.length; x++) {
	types[x] = callbacks[x].getClass();
}
// fixedInterceptorMap only populated at this point, after getCallbacks call above
enhancer.setCallbackFilter(new ProxyCallbackFilter(
	this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
enhancer.setCallbackTypes(types);

protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
	enhancer.setInterceptDuringConstruction(false);
	enhancer.setCallbacks(callbacks);
	return (this.constructorArgs != null ?
			enhancer.create(this.constructorArgTypes, this.constructorArgs) :
			enhancer.create());
}

```