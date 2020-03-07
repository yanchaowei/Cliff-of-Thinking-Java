# Spring核心接口之InitializingBean

## 概述

`InitializingBean`接口为bean提供了属性初始化后的处理方法，它只包括`afterPropertiesSet()`方法，凡是继承该接口的类，在bean的属性初始化后都会执行该方法。

```java
package org.springframework.beans.factory;

public interface InitializingBean {
    void afterPropertiesSet() throws Exception;
}
```

该方法是在属性设置后才调用的，从方法名`afterPropertiesSet()`也可以清楚的理解。

## Bean的加载

通过查看spring的加载bean的源码类`AbstractAutowireCapableBeanFactory`可以看到

```java
protected void invokeInitMethods(String beanName, Object bean, RootBeanDefinition mbd) throws Throwable {
    boolean isInitializingBean = bean instanceof InitializingBean;
	// //判断该bean是否实现了实现了InitializingBean接口，如果实现了InitializingBean接口，则调用bean的afterPropertiesSet方法
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
		// 调用afterPropertiesSet
        ((InitializingBean)bean).afterPropertiesSet();
    }

    // 判断是否指定了init-method方法，如果指定了init-method方法，则再调用制定的init-method
    String initMethodName = mbd != null ? mbd.getInitMethodName() : null;
    if (initMethodName != null && (!isInitializingBean || !"afterPropertiesSet".equals(initMethodName)) && !mbd.isExternallyManagedInitMethod(initMethodName)) {
        // 反射调用init-method方法
        this.invokeCustomInitMethod(beanName, bean, initMethodName, mbd.isEnforceInitMethod());
    }

}
```

分析代码可以了解：

+ spring为bean提供了两种初始化bean的方式，实现`InitializingBean`接口，实现`afterPropertiesSet()`方法，或者在配置文件中同过`init-method`指定，两种方式可以同时使用
+ 实现`InitializingBean`接口是直接调用`afterPropertiesSet()`方法，比通过反射调用`init-method`指定的方法效率相对来说要高点。但是`init-method`方式消除了对spring的依赖
+ 如果调用`afterPropertiesSet`方法时出错，则不调用`init-method`指定的方法。