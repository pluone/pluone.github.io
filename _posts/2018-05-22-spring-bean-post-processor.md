---
layout: post
title:  "Spring框架BeanPostProcessor解读"
date:   2018-05-22 18:00:00 +0800
categories: 框架
---

按照Spring bean的生命周期，先读取BeanDefinition，然后是实例化，之后是初始化，而`BeanPostProcessor`就是作用在实例化阶段之后，围绕着初始化阶段。

```java
public interface BeanPostProcessor {
    //在初始化阶段开始前执行
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    //在初始化阶段之后执行
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

Spring bean生命周期的图片如下,BPP环绕在Initializer周围.  
<img src="/assets/bean-lifecycle.jpg" alt="Spring bean生命周期" style="width: 500px;"/>

## 源码解读

我们从`BeanFactory`的`getBean`方法开始分析源码，之后执行到`AbstractAutowireCapableBeanFactory`的`doCreateBean`方法  
`doCreateBean`代码片段摘录如下

```java
Object exposedObject = bean;
try {
    populateBean(beanName, mbd, instanceWrapper);
    if (exposedObject != null) {
        //在这里执行bean的初始化
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
}
```

```java
//bean初始化代码整段摘录,可以只看有注释的地方
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
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
        //这里执行了BeanPostProcessor的postProcessBeforeInitialization方法
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        //这里执行bean的初始化
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }

    if (mbd == null || !mbd.isSynthetic()) {
        //初始化执行继续执行BeanPostProcessor的postProcessAfterInitialization方法
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

参考:  

1. 《Spring技术内幕第二版》
2. https://stackoverflow.com/questions/29743320/how-exactly-works-the-spring-bean-post-processor