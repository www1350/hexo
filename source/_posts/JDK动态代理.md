---
title: JDK动态代理
abbrlink: dfdc2005
date: 2017-08-31 22:40:11
tags: [java,代理]
categories: 基础
---

使用动态代理的五大步骤
1.通过实现InvocationHandler接口来自定义自己的InvocationHandler;

2.通过Proxy.getProxyClass获得动态代理类

3.通过反射机制获得代理类的构造方法，方法签名为getConstructor(InvocationHandler.class)

4.通过构造函数获得代理对象并将自定义的InvocationHandler实例对象传为参数传入

5.通过代理对象调用目标方法

```
public interface HelloWorld {  
    void sayHello(String name);  
} 


public class HelloWorldImpl implements HelloWorld {  
    @Override  
    public void sayHello(String name) {  
        System.out.println("Hello " + name);  
    }  
}  


public class CustomInvocationHandler implements InvocationHandler {  
    private Object target;  
  
    public CustomInvocationHandler(Object target) {  
        this.target = target;  
    }  
  
    @Override  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        System.out.println("Before invocation");  
        Object retVal = method.invoke(target, args);  
        System.out.println("After invocation");  
        return retVal;  
    }  
}  

```


方法1
```
public class ProxyTest {  
    public static void main(String[] args) throws Exception {  
//生成$Proxy0的class文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");  
 //获取动态代理类
        Class proxyClazz = Proxy.getProxyClass(HelloWorld.class.getClassLoader(),HelloWorld.class);

  //获得代理类的构造函数，并传入参数类型InvocationHandler.class
        Constructor constructor = proxyClazz.getConstructor(InvocationHandler.class);
        //通过构造函数来创建动态代理对象，将自定义的InvocationHandler实例传入
        HelloWorld proxy = (HelloWorld) constructor.newInstance(new CustomInvocationHandler(new HelloWorldImpl()));
        proxy.sayHello("Mikan");  
    }  
} 
```




方法2
```
public class ProxyTest {  
    public static void main(String[] args) throws Exception {  
//生成$Proxy0的class文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");  
        CustomInvocationHandler handler = new CustomInvocationHandler(new HelloWorldImpl());  
        HelloWorld proxy = (HelloWorld) Proxy.newProxyInstance(  
                ProxyTest.class.getClassLoader(),  
                new Class[]{HelloWorld.class},  
                handler);  
        proxy.sayHello("Mikan");  
    }  
} 
```


http://www.cnblogs.com/MOBIN/p/5597215.html