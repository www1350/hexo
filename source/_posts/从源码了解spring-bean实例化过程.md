---
title: 从源码了解spring bean实例化过程
date: 2018-04-03 22:39:53
tags:
categories:
---

现在都在简书写了：http://www.jianshu.com/p/0a9c964dd28a

我们先来看下spring如何手动初始化一个对象

```
ClassPathResource res = new ClassPathResource("beans.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
User user=(User) factory.getBean("user");
```


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/3095882-5ff18c4b78b18301.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


所以我们先从DefaultListableBeanFactory开始了解
AbstractBeanFactory

```
public <T> T getBean(String name, Class<T> requiredType, Object... args) throws BeansException {
		return doGetBean(name, requiredType, args, false);
	}
```

```
	protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

```

1.去掉&

```
		final String beanName = transformedBeanName(name);
```
```
	protected String transformedBeanName(String name) {
		return canonicalName(BeanFactoryUtils.transformedBeanName(name));
	}
```

2.如果是单例且存在，就直接取过来

```
		Object bean;
		// 
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			....
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
```

3.如果已经创建，抛出异常

```
		else {
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
```

4.在父BeanFactory查找

```
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
```

5.创建标记，（用于清除）

```
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}
```

6.寻找bean定义信息，并针对bean定义进行验证

```
			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);
```
7.处理依赖信息，这里会针对xml定义中的depends-on进行处理

```
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dependsOnBean : dependsOn) {
						if (isDependent(beanName, dependsOnBean)) {
						....
					registerDependentBean(dependsOnBean, beanName);
						getBean(dependsOnBean);
					}
				}
```
具体通过这个私有方法来解决依赖的
假设A依赖于B，那么在创建A之前，必须保证B先被创建。
在创建了B之后，这里会进行依赖信息存储。后面在递归调用一下，不过依赖关系反过来。
dependentBeanMap放 A->［B,C,D］ （A所依赖的B,C,D）
```
private boolean isDependent(String beanName, String dependentBeanName, Set<String> alreadySeen) {
		if (alreadySeen != null && alreadySeen.contains(beanName)) {
			return false;
		}
		String canonicalName = canonicalName(beanName);
		Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
		if (dependentBeans == null) {
			return false;
		}
		if (dependentBeans.contains(dependentBeanName)) {
			return true;
		}
		for (String transitiveDependency : dependentBeans) {
			if (alreadySeen == null) {
				alreadySeen = new HashSet<String>();
			}
			alreadySeen.add(beanName);
			if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
				return true;
			}
		}
		return false;
	}
```
单例模式就调用getSingleton(String beanName, ObjectFactory<?> singletonFactory)

```

				// 单例
				if (mbd.isSingleton()) {
					....
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
```

```
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
              ....
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
				....
				beforeSingletonCreation(beanName);
				....
					singletonObject = singletonFactory.getObject();
				....
					addSingleton(beanName, singletonObject);
				
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}
```
原型模式
```
				else if (mbd.isPrototype()) {
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
```

其他模式，比如scope=request

```
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					....
					try {
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					....
				}
			}
			....
		}
```

8.无论哪种模式都会进入AbstractAutowireCapableBeanFactory的createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)

```
                ....
				RootBeanDefinition mbdToUse = mbd;

		// 解析beanDefinition，以确保bean定义中的class可以被正确解析
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		....

		try {
			// 如果实现了InstantiationAwareBeanPostProcessor就先postProcessBeforeInstantiation(Class<?> beanClass, String beanName)->postProcessAfterInitialization(Object bean, String beanName)
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		....

		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		....
		}
		return beanInstance;
```

9.Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)

```
                         ....
			if (exposedObject != null) {
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		       ....

```

10.Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)

```
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
                       //setBeanName#BeanNameAware
                       //->setBeanClassLoader#BeanClassLoaderAware
                       //->setBeanFactory#BeanFactoryAware
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
                    //->postProcessBeforeInitialization#BeanPostProcessor
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
                        //->afterPropertiesSet#InitializingBean
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		....

		if (mbd == null || !mbd.isSynthetic()) {
                    //->postProcessAfterInitialization#BeanPostProcessor
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
```

```
	private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
```

```
	protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			....
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						@Override
						public Object run() throws Exception {
							((InitializingBean) bean).afterPropertiesSet();
							return null;
						}
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null) {
			String initMethodName = mbd.getInitMethodName();
			if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```

```
postProcessBeforeInstantiation(Class<?> beanClass, String beanName)#InstantiationAwareBeanPostProcessor
->postProcessAfterInitialization(Object bean, String beanName)#InstantiationAwareBeanPostProcessor
->setBeanName#BeanNameAware
->setBeanClassLoader#BeanClassLoaderAware
->setBeanFactory#BeanFactoryAware
->postProcessBeforeInitialization#BeanPostProcessor
->afterPropertiesSet#InitializingBean
->postProcessAfterInitialization#BeanPostProcessor
```


```
public class InitAndDestroySeqBean implements InitializingBean,DisposableBean,BeanPostProcessor,InstantiationAwareBeanPostProcessor {

    public InitAndDestroySeqBean() {
        System.out.println("构造方法");
    }

    @PostConstruct
    public void postConstruct(){
        System.out.println("@PostConstruct");
    }

    @Override public void afterPropertiesSet() throws Exception {
        System.out.println("afterPropertiesSet()#InitializingBean");

    }

    public void initMethod(){
        System.out.println("init-method");

    }

    @PreDestroy
    public void preDestroy(){
        System.out.println("@PreDestroy");
    }


    @Override public void destroy() throws Exception {
        System.out.println("destroy()#DisposableBean");
    }


    public void destroyMethod(){
        System.out.println("destroy-method");
    }


    @Override public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("postProcessBeforeInitialization(Object bean, String beanName)#BeanPostProcessor");
        return bean;
    }


    @Override public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("postProcessAfterInitialization(Object bean, String beanName)#BeanPostProcessor");
        return bean;
    }


    @Override public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName)
            throws BeansException {
        System.out.println("postProcessBeforeInstantiation(Class<?> beanClass, String beanName)#InstantiationAwareBeanPostProcessor");
        return null;
    }


    @Override public boolean postProcessAfterInstantiation(Object bean, String beanName)
            throws BeansException {
        System.out.println("postProcessAfterInstantiation(Object bean, String beanName)#InstantiationAwareBeanPostProcessor");

        return false;
    }


    @Override public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds,
            Object bean, String beanName) throws BeansException {
        System.out.println("postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds,\n"
                + "            Object bean, String beanName)#InstantiationAwareBeanPostProcessor");

        return null;
    }
}

```

```
20170308:11:34:52.866 [main] INFO   Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@49c43f4e, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@290dbf45, org.springframework.test.context.support.DependencyInjectionTestExecutionListener@12028586, org.springframework.test.context.support.DirtiesContextTestExecutionListener@17776a8, org.springframework.test.context.transaction.TransactionalTestExecutionListener@69a10787, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@2d127a61]20170308:11:34:52.972 [main] INFO   Loading XML bean definitions from URL [file:/Users/dsc/IdeaProjects/firDemo/firdemo/web/target/classes/init.xml]
20170308:11:34:53.105 [main] INFO   Refreshing org.springframework.context.support.GenericApplicationContext@2b4a2ec7: startup date [Wed Mar 08 11:34:53 CST 2017]; root of context hierarchy
构造方法
@PostConstruct
afterPropertiesSet()#InitializingBean
init-method
beanName)#InstantiationAwareBeanPostProcessor**guavaCache
postProcessAfterInstantiation(Object bean, String beanName)#InstantiationAwareBeanPostProcessor**guavaCache
postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds,
            Object bean, String beanName)#InstantiationAwareBeanPostProcessor**guavaCache
postProcessBeforeInitialization(Object bean, String beanName)#BeanPostProcessor**guavaCache
afterPropertiesSet#InitializingBean**guavaCache
postProcessAfterInitialization(Object bean, String beanName)#BeanPostProcessor**guavaCache
20170308:11:34:53.286 [Thread-0] INFO   Closing org.springframework.context.support.GenericApplicationContext@2b4a2ec7: startup date [Wed Mar 08 11:34:53 CST 2017]; root of context hierarchy
@PreDestroy
destroy()#DisposableBean
destroy-method
  
  
```



刚才反复提到了几个接口，我们来单独看下他们的使用：

BeanPostProcessor接口
```
public interface BeanPostProcessor {

	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

InstantiationAwareBeanPostProcessor接口
```
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

	Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

	boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

	PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;
}
```

InitializingBean接口
```
public interface InitializingBean {
        //本bean
	void afterPropertiesSet() throws Exception;
}
```

DisposableBean接口
```
public interface DisposableBean {
	void destroy() throws Exception;
}

```
