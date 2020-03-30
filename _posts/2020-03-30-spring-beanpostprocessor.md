---
title: Spring的Bean后置处理器与Bean创建流程
tags:
    - Spring
    - Bean后置处理器
    - BeanPostProcessor
---

## BeanPostProcessor

BeanPostProcessor使所有Bean后置处理器的子接口，包含两个方法：
`postProcessBeforeInitialization`与`postProcessAfterInitialization`。

在Spring创建Bean的过程中，依次调用了如下的Bean后置处理器：

## 1.InstantiationAwareBeanPostProcessor

`InstantiationAwareBeanPostProcessor`接口继承`BeanPostProcessor`接口，它内部提供了3个方法，再加上`BeanPostProcessor`接口的2个方法，所以实现这个接口需要实现5个方法。`InstantiationAwareBeanPostProcessor`接口的主要作用在于目标对象的`Instantiation`(实例化)过程中需要处理的事情，包括实例化对象的前后过程以及实例的属性设置。

### postProcessBeforeInstantiation

在`AbstractAutowireCapableBeanFactory#createBean()`方法中执行了这个后置处理器。

在目标对象实例化之前调用，方法的返回值类型是Object，我们可以注册一个`InstantiationAwareBeanPostProcessor`并重写`postProcessBeforeInstantiation`方法返回任何类型的值。由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(一般都是代理对象)。如果该方法的返回值代替原本该生成的目标对象，后续只会调用`applyBeanPostProcessorsAfterInitialization`执行所有后置处理器的`postProcessAfterInitialization`方法后直接返回，不再继续执行Bean的正常创建流程。

比如AOP就是在`AbstractAutoProxyCreator`这个后置处理器中，调用`postProcessBeforeInstantiation`完成Bean类的判断，判断是否不需要代理。分为两种情况：

1. 不需代理的类：Infrastructure类（比如加了`@Advice`、`@Pointcut`等注解的类，这些类也不需要被代理）或者shouldSkip的类，加入黑名单`this.advisedBeans.put(cacheKey, Boolean.FALSE)`。然后返回null表示继续执行Bean的正常创建流程。注意在后面调用`postProcessAfterInitialization`时(后文有提到)，这些处于`advisedBeans`不需要代理的类将直接返回目标对象而不创建代理对象。
2. 已经被代理，则返回一个代理对象的代理对象，由于返回值不为null，因此不再继续执行Bean的正常创建流程。


## 2.SmartInstantiationAwareBeanPostProcessor

`SmartInstantiationAwareBeanPostProcessor`继承了`InstantiationAwareBeanPostProcessor`

### determineCandidateConstructors

`AbstractAutowireCapableBeanFactory#determineConstructorsFromBeanPostProcessors`中调用,可以检测出Bean的多个候选构造器。

## 3.MergedBeanDefinitionPostProcessor

用于处理`Merged Bean` 的 `Definition`(merged代表bean会继承父bean的definition)

### postProcessMergedBeanDefinition

`AbstractAutowireCapableBeanFactory#doCreateBean`中调用。

缓存bean的注解信息的后置处理器，仅仅是缓存或者干脆叫做查找更加合适，没有完成注入，注入是另外一个后置处理器的作用

## 4.SmartInstantiationAwareBeanPostProcessor

### getEarlyBeanReference

在`AbstractAutowireCapableBeanFactory#getEarlyBeanReference中调用`。

得到获得提前暴露的对象引用(EarlyBean)。主要用于解决循环引用的问题，只有单例对象才会调用此方法。

## 5.InstantiationAwareBeanPostProcessor

### postProcessAfterInstantiation

在`AbstractAutowireCapableBeanFactory#populateBean`中调用。判断你的bean需不需要完成属性填充(true/false)。

## 6.InstantiationAwareBeanPostProcessor

### postProcessProperties

在`AbstractAutowireCapableBeanFactory#populateBean`中调用。

### postProcessPropertyValues

在`AbstractAutowireCapableBeanFactory#populateBean`中调用。完成属性填充(自动注入)。

## 7.BeanPostProcessor

### postProcessBeforeInitialization

在`AbstractAutowireCapableBeanFactory#initializeBean#applyBeanPostProcessorsBeforeInitialization`中调用。

## 中间调用了invokeInitMethods执行生命周期相关函数

## 8.BeanPostProcessor

### postProcessAfterInitialization

`AbstractAutowireCapableBeanFactory#initializeBean#applyBeanPostProcessorsAfterInitialization`中调用。

比如AOP就是在`AbstractAutoProxyCreator`这个后置处理器中，调用`postProcessAfterInitialization`完成代理对象的创建的。

## 9.InitDestroyAnnotationBeanPostProcessor

...
