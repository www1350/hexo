---
title: Tomcat 源码解析（四）--Lifecycle 机制和实现原理
abbrlink: bf6044ee
date: 2018-04-03 22:43:10
tags:
categories:
---

# Lifecycle接口
 ```
/**
     * Add a LifecycleEvent listener to this component.
     *
     * @param listener The listener to add
     */
    public void addLifecycleListener(LifecycleListener listener);//给该组将添加一个监听器


    /**
     * Get the life cycle listeners associated with this life cycle. If this
     * component has no listeners registered, a zero-length array is returned.
     */
    public LifecycleListener[] findLifecycleListeners();//获取该组件所有已注册的监听器


    /**
     * Remove a LifecycleEvent listener from this component.
     *
     * @param listener The listener to remove
     */
    public void removeLifecycleListener(LifecycleListener listener);//删除该组件中的一个监听器

 ```

LifecycleListener接口
```
public interface LifecycleListener {
    /**
     * Acknowledge the occurrence of the specified event.
     *
     * @param event LifecycleEvent that has occurred
     */
    public void lifecycleEvent(LifecycleEvent event);
}

```


LifecycleEvent接口（继承jdk 内置的 java.util.EventObject）
```
public final class LifecycleEvent extends EventObject {

    private static final long serialVersionUID = 1L;


    // ----------------------------------------------------------- Constructors

    /**
     * Construct a new LifecycleEvent with the specified parameters.
     *
     * @param lifecycle Component on which this event occurred
     * @param type Event type (required)
     * @param data Event data (if any)
     */
    public LifecycleEvent(Lifecycle lifecycle, String type, Object data) {

        super(lifecycle);
        this.type = type;
        this.data = data;
    }


    // ----------------------------------------------------- Instance Variables


    /**
     * The event data associated with this event.
     */
    private final Object data;


    /**
     * The event type this instance represents.
     */
    private final String type;


    // ------------------------------------------------------------- Properties


    /**
     * Return the event data of this event.
     */
    public Object getData() {

        return (this.data);

    }


    /**
     * Return the Lifecycle on which this event occurred.
     */
    public Lifecycle getLifecycle() {

        return (Lifecycle) getSource();

    }


    /**
     * Return the event type of this event.
     */
    public String getType() {

        return (this.type);

    }


}

```


LifecycleSupport的fireLifecycleEvent，向注册到组件中的所有监听器发布这个新构造的事件对象。

在 LifecycleBase类里LifecycleSupport是这么初始化的：`private final LifecycleSupport lifecycle = new LifecycleSupport(this);`，就是把StandardServer(LifecycleBase)的实例作为Lifecycle
```
    public void fireLifecycleEvent(String type, Object data) {

        LifecycleEvent event = new LifecycleEvent(lifecycle, type, data);
        LifecycleListener interested[] = listeners;
        for (int i = 0; i < interested.length; i++)
            interested[i].lifecycleEvent(event);

    }
```




org.apache.catalina.startup.Catalina 类的 createStartDigester 方法有这么一段代码：

```
        digester.addObjectCreate("Server/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Listener");
        digester.addSetNext("Server/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");
```



LifecycleSupport
```
    public void addLifecycleListener(LifecycleListener listener) {

      synchronized (listenersLock) {
          LifecycleListener results[] =
            new LifecycleListener[listeners.length + 1];
          for (int i = 0; i < listeners.length; i++)
              results[i] = listeners[i];
          results[listeners.length] = listener;
          listeners = results;
      }

    }

private LifecycleListener listeners[] = new LifecycleListener[0];  
```



如果对设计模式比较熟悉的话会发现 Tomcat 的 Lifecycle 使用的是观察者模式：LifecycleListener 代表的是抽象观察者，它定义一个 lifecycleEvent 方法，而实现该接口的监听器是作为具体的观察者。Lifecycle  接口代表的是抽象主题，它定义了管理观察者的方法和它要所做的其它方法。而各组件代表的是具体主题，它实现了抽象主题的所有方法。通常会由具体主题保存对具体观察者对象有用的内部状态；在这种内部状态改变时给其观察者发出一个通知。Tomcat 对这种模式做了改进，增加了另外两个工具类：LifecycleSupport、LifecycleEvent ，它们作为辅助类扩展了观察者的功能。LifecycleEvent 中定义了事件类别，不同的事件在具体观察者中可区别处理，更加灵活。LifecycleSupport 类代理了所有具体主题对观察者的管理，将这个管理抽出来统一实现，以后如果修改只要修改 LifecycleSupport 类就可以了，不需要去修改所有具体主题，因为所有具体主题的对观察者的操作都被代理给 LifecycleSupport 类了。
事件的发布使用的是推模式，即每发布一个事件都会通知主题的所有具体观察者，由各观察者再来决定是否需要对该事件进行后续处理。