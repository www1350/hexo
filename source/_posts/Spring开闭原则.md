---
title: Spring开闭原则
date: 2018-04-03 22:55:43
tags:
categories:
---

BeanPostProcessor的执行顺序
1、如果使用BeanFactory实现，非ApplicationContext实现，BeanPostProcessor执行顺序就是添加顺序。

2、如果使用的是AbstractApplicationContext（实现了ApplicationContext）的实现，则通过如下规则指定顺序。
2.1、PriorityOrdered（继承了Ordered），实现了该接口的BeanPostProcessor会在第一个顺序注册，标识高优先级顺序，即比实现Ordered的具有更高的优先级；
2.2、Ordered，实现了该接口的BeanPostProcessor会第二个顺序注册；

int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;//最高优先级
int LOWEST_PRECEDENCE = Integer.MAX_VALUE;//最低优先级

即数字越小优先级越高，数字越大优先级越低，如0（高优先级）——1000（低优先级）

2.3、无序的，没有实现Ordered/ PriorityOrdered的会在第三个顺序注册；
2.4、内部Bean后处理器，实现了MergedBeanDefinitionPostProcessor接口的是内部Bean PostProcessor，将在最后且无序注册。


3、接下来我们看看内置的BeanPostProcessor执行顺序

1.注册实现了PriorityOrdered接口的BeanPostProcessor

2.注册实现了Ordered接口的BeanPostProcessor
AbstractAutoProxyCreator              实现了Ordered，order = Ordered.LOWEST_PRECEDENCE
MethodValidationPostProcessor          实现了Ordered，LOWEST_PRECEDENCE
ScheduledAnnotationBeanPostProcessor   实现了Ordered，LOWEST_PRECEDENCE
AsyncAnnotationBeanPostProcessor      实现了Ordered，order = Ordered.LOWEST_PRECEDENCE

3.注册无实现任何接口的BeanPostProcessor
BeanValidationPostProcessor            无序
ApplicationContextAwareProcessor       无序
ServletContextAwareProcessor          无序

4. 注册实现了MergedBeanDefinitionPostProcessor接口的BeanPostProcessor，且按照实现了Ordered的顺序进行注册，没有实现Ordered的默认为Ordered.LOWEST_PRECEDENCE。
  PersistenceAnnotationBeanPostProcessor  实现了PriorityOrdered，Ordered.LOWEST_PRECEDENCE 5.
  AutowiredAnnotationBeanPostProcessor   实现了PriorityOrdered，order = Ordered.LOWEST_PRECEDENCE - 2
  RequiredAnnotationBeanPostProcessor    实现了PriorityOrdered，order = Ordered.LOWEST_PRECEDENCE - 1
  CommonAnnotationBeanPostProcessor    实现了PriorityOrdered，Ordered.LOWEST_PRECEDENCE

从上到下顺序执行，如果order相同则我们应该认为同序（谁先执行不确定，其执行顺序根据注册顺序决定）。





看源码毕竟清晰
AbstractApplicationContext
```
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```


registerBeanPostProcessors(beanFactory); 这个是关键所在，对吧

```
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
```


注册顺序PriorityOrdered（priorityOrderedPostProcessors里的，按order级别）－》Ordered（orderedPostProcessorNames里的，按order级别）－》既不是PriorityOrdered也不是Ordered（nonOrderedPostProcessors）－》实现MergedBeanDefinitionPostProcessor（internalPostProcessors，相当于重新注册，按ordered顺序注册）

priorityOrderedPostProcessors：PriorityOrdered
internalPostProcessors：MergedBeanDefinitionPostProcessor
orderedPostProcessorNames：Ordered
nonOrderedPostProcessorNames：（非PriorityOrdered，Ordered）
```
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(beanFactory, orderedPostProcessors);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(beanFactory, internalPostProcessors);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}

```


所以如果有顺序的话。就是
注册实现了PriorityOrdered接口的BeanPostProcessor［PersistenceAnnotationBeanPostProcessor  （PriorityOrdered，Ordered.LOWEST_PRECEDENCE－5）－》
AutowiredAnnotationBeanPostProcessor （PriorityOrdered，order = Ordered.LOWEST_PRECEDENCE - 2）－》
RequiredAnnotationBeanPostProcessor  （PriorityOrdered，order = Ordered.LOWEST_PRECEDENCE - 1）－》
CommonAnnotationBeanPostProcessor  （PriorityOrdered，Ordered.LOWEST_PRECEDENCE）］
 －》注册实现了Ordered接口的BeanPostProcessor－》AbstractAutoProxyCreator、MethodValidationPostProcessor、ScheduledAnnotationBeanPostProcessor、AsyncAnnotationBeanPostProcessor(Ordered，LOWEST_PRECEDENCE，互相无序)    －》注册无实现任何接口的BeanPostProcessor
（BeanValidationPostProcessor 、ApplicationContextAwareProcessor   、ServletContextAwareProcessor  无序）－》注册实现了MergedBeanDefinitionPostProcessor接口的BeanPostProcessor，（Ordered的顺序进行注册，没有实现Ordered的默认为Ordered.LOWEST_PRECEDENCE）
