---
title: Spring声明式事务Transactional原理只看这一篇就够了
tags:
  - spring
  - transaction
categories: uncategorized
cover: /upload/homePage/20210928152000.jpg
abbrlink: '204726e2'
date: 2021-09-28 15:18:34
---
## 问题
Spring声明式事务@Transactional在同一个类中的两个方法之间调用失效是个老生常谈的问题，之前恰巧又遇到有这么写的同学，今天就翻一翻源码，总结一下其背后的原因。

## 初步分析
由于Spring声明式事务采用动态代理（AOP）实现，它会在代码运行时为类生成一个代理对象，实际会使用代理对象调用业务方法，并在方法调用之前决定是否开启一个事务，在方法执行之后决定事务的提交和回滚。

其相当于下面这段代码：
```
// 业务方法 
main {
    bizMethod();
}

// 经过AOP事务增强后
main {
  Connection con = null; 
  try{ 
      con = getConnection(); 
      con.setAutoCommit(false); 

      //方法调用
      bizMethod(); 

      //提交事务
      con.commit(); 
  } catch(RuntimeException ex) { 
      //回滚事务
      con.rollback();   
  } finally { 
      //释放资源
      closeCon(); 
  }
}
```

像这类使用代理对象调用方法的实现，我们直接在业务代码里加断点是很难debug到目标代码的，这里教大家一种方式能够快速的定位目标代码位置。

首先我们需要构造一个简单的Spring Jdbc环境，相关代码我会传到的[Github aop-transactional-test](http://github.com/hf-hf/aop-transactional-test)，在TestMapper中添加一条简单的插入语句，注意这里要故意造成报错，以INSERT语句为例，我们可以这样写INSERT INTO t_test(name)，然后将TestMapper注入TestService，此时我们在TestSerivce的方法testSave中调用TestMapper插入数据的方法save，并通过控制该方法上的@Transactional注解，分别以有无注解的方式各运行一次，我们会得到两段报错信息。

有/无@Transactional运行的报错信息，差异为'-'标记部分。
```diff
### Error updating database.  Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1
### The error may exist in file [H:\github\aop-test\target\classes\mapper\TestMapper.xml]
### The error may involve defaultParameterMap
### The error occurred while setting parameters
### SQL: INSERT INTO t_test(name)
### Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1
; bad SQL grammar []; nested exception is com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1

	at org.springframework.jdbc.support.SQLErrorCodeSQLExceptionTranslator.doTranslate(SQLErrorCodeSQLExceptionTranslator.java:239)
	at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:70)
	at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:88)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:440)
	at com.sun.proxy.$Proxy54.insert(Unknown Source)
	at org.mybatis.spring.SqlSessionTemplate.insert(SqlSessionTemplate.java:271)
	at com.baomidou.mybatisplus.core.override.MybatisMapperMethod.execute(MybatisMapperMethod.java:60)
	at com.baomidou.mybatisplus.core.override.MybatisMapperProxy.invoke(MybatisMapperProxy.java:96)
	at com.sun.proxy.$Proxy55.save(Unknown Source)
	at com.test.demo.service.TestService.testSave(TestService.java:20)
	- at com.test.demo.service.TestService$$FastClassBySpringCGLIB$$28e09d8e.invoke(<generated>)
	- at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
	- at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:779)
	- at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
	- at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750)
	- at org.springframework.transaction.interceptor.TransactionInterceptor$1.proceedWithInvocation(TransactionInterceptor.java:123)
	- at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:388)
	- at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
	- at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	- at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750)
	- at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:692)
	- at com.test.demo.service.TestService$$EnhancerBySpringCGLIB$$6cc1a484.testSave(<generated>)
	at com.test.AopTestApplicationTests.testServiceSave(AopTestApplicationTests.java:35)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:497)
	at org.junit.platform.commons.util.ReflectionUtils.invokeMethod(ReflectionUtils.java:688)
```

通过比对两个调用堆栈信息可以看出，应用了声明式事务的运行堆栈信息中出现了CglibAopProxy、invoke、TransactionInterceptor等信息，这也可以证实Spring的声明式事务确实是通过AOP的方式实现的，那么接下来我们就可以从该堆栈信息提示的位置入手开始进行debug调试了。

## 源码分析
我们根据有/无事务注解输出的堆栈信息的差异，可以分析出只有进入TransactionInterceptor，才会被加入Spring的事务管理，因此点击TransactionInterceptor$1.proceedWithInvocation定位到代码位置，再结合上下两条堆栈信息CglibAopProxy、TransactionAspectSupport，可以看出这是代理对象执行方法被拦截后，在方法执行前后追加逻辑操作的位置。

org.springframework.transaction.interceptor.TransactionInterceptor#invoke
```
public Object invoke(MethodInvocation invocation) throws Throwable {
  // Work out the target class: may be {@code null}.
  // The TransactionAttributeSource should be passed the target class
  // as well as the method, which may be from an interface.
  Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

  // Adapt to TransactionAspectSupport's invokeWithinTransaction...
  return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
    @Override
    @Nullable
    public Object proceedWithInvocation() throws Throwable {
      return invocation.proceed();
    }
    @Override
    public Object getTarget() {
      return invocation.getThis();
    }
    @Override
    public Object[] getArguments() {
      return invocation.getArguments();
    }
  });
}
```

org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction
```
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {
  ...
  PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
  final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

  if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
    // Standard transaction demarcation with getTransaction and commit/rollback calls.
    // 创建事务
    TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

    Object retVal;
    try {
      // This is an around advice: Invoke the next interceptor in the chain.
      // This will normally result in a target object being invoked.
      // 调用原方法
      retVal = invocation.proceedWithInvocation();
    }
    catch (Throwable ex) {
      // target invocation exception
      // 提交/回滚事务
      completeTransactionAfterThrowing(txInfo, ex);
      throw ex;
    }
    finally {
      // 清理、释放资源
      cleanupTransactionInfo(txInfo);
    }

    if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
      // Set rollback-only in case of Vavr failure matching our rollback rules...
      TransactionStatus status = txInfo.getTransactionStatus();
      if (status != null && txAttr != null) {
        retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
      }
    }

    commitTransactionAfterReturning(txInfo);
    return retVal;
  }
  ...
}
```

我们可以在TransactionAspectSupport#invokeWithinTransaction中看到，代理方法执行前会创建事务，在catch中会提交或者回滚事务，最后执行清理释放的过程，这和问题分析中所示经过AOP事务增强后的代码是一致的。

另外在源码CglibAopProxy#getProxy中，可以看到Spring使用cglib创建代理对象，并将ProxyCallbackFilter添加到代理对象的CallbackFilter中。

org.springframework.aop.framework.CglibAopProxy
```
public Object getProxy(@Nullable ClassLoader classLoader) {
  ...
  Callback[] callbacks = getCallbacks(rootClass);
  Class<?>[] types = new Class<?>[callbacks.length];
  for (int x = 0; x < types.length; x++) {
    types[x] = callbacks[x].getClass();
  }
  // fixedInterceptorMap only populated at this point, after getCallbacks call above
  enhancer.setCallbackFilter(new ProxyCallbackFilter(
      this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
  enhancer.setCallbackTypes(types);

  // Generate the proxy class and create a proxy instance.
  return createProxyClassAndInstance(enhancer, callbacks);
  ...
}
```

接着在CglibAopProxy.ProxyCallbackFilter#accept代理回调中会调用org.springframework.aop.framework.AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice，该方法会从调用方法中获取@Transactional注解，如果有该注解，则将TransactionInterceptor加入代理对象的拦截器链中，若加入了该拦截器，即表示启用事务，否则不会启用事务。

org.springframework.aop.framework.CglibAopProxy.ProxyCallbackFilter#accept
```
public int accept(Method method) {
  ...
  Class<?> targetClass = this.advised.getTargetClass();
  // Proxy is not yet available, but that shouldn't matter.
  List<?> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
  boolean haveAdvice = !chain.isEmpty();
  boolean exposeProxy = this.advised.isExposeProxy();
  boolean isStatic = this.advised.getTargetSource().isStatic();
  boolean isFrozen = this.advised.isFrozen();
  if (haveAdvice || !isFrozen) {
  ...
}
```

详细代码调用嵌套比较深，这里贴出调用的顺序。
```
org.springframework.aop.framework.CglibAopProxy.ProxyCallbackFilter#accept
  org.springframework.aop.framework.DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice
    org.springframework.transaction.interceptor.TransactionAttributeSourcePointcut#matches
      org.springframework.transaction.interceptor.AbstractFallbackTransactionAttributeSource#getTransactionAttribute
        org.springframework.transaction.interceptor.AbstractFallbackTransactionAttributeSource#computeTransactionAttribute
          org.springframework.transaction.annotation.AnnotationTransactionAttributeSource#findTransactionAttribute(java.lang.reflect.Method)
            org.springframework.transaction.annotation.AnnotationTransactionAttributeSource#determineTransactionAttribute
              org.springframework.transaction.annotation.SpringTransactionAnnotationParser#parseTransactionAnnotation(java.lang.reflect.AnnotatedElement)
```

最终会在parseTransactionAnnotation中获取@Transactional注解并解析事务配置信息。

org.springframework.transaction.annotation.SpringTransactionAnnotationParser
```
public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
  AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
      element, Transactional.class, false, false);
  if (attributes != null) {
    return parseTransactionAnnotation(attributes);
  }
  else {
    return null;
  }
}
```

若是启用了事务，那么在代理对象调用方法时，就会被TransactionInterceptor拦截，由TransactionAspectSupport进行事务增强，在invokeWithinTransaction方法中创建事务、执行原业务方法、提交/回滚事务，也就是上面的TransactionAspectSupport#invokeWithinTransaction源码内容。

## 产生原因
那么我们现在只剩下一个问题还没有理清，也就是代理对象是什么时候创建的呢？

其实Spring项目启动时会加载类(Bean)，如果类中存在添加了@Transactional事务注解的方法，则会为Bean生成代理对象，createProxy生成代理对象时会调用CglibAopProxy.getProxy，这会将TransactionInterceptor加入代理对象的拦截器链中，这样就接上了之前说明的内容，详情见下方代码：

org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
```
public Object getEarlyBeanReference(Object bean, String beanName) {
  Object cacheKey = getCacheKey(bean.getClass(), beanName);
  // 保存原始对象信息
  this.earlyProxyReferences.put(cacheKey, bean);
  return wrapIfNecessary(bean, beanName, cacheKey);
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
    return bean;
  }
  if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
    return bean;
  }
  if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
  }

  // Create proxy if we have advice.
  // 判断@Transactional注解，返回TransactionInterceptor
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  if (specificInterceptors != DO_NOT_PROXY) {
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    // 创建代理对象，这里createProxy会调用到CglibAopProxy.getProxy
    Object proxy = createProxy(
        bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }

  this.advisedBeans.put(cacheKey, Boolean.FALSE);
  return bean;
}
```

以示例Demo为例，earlyProxyReferences成员变量中保存了TestService原始对象信息，然后通过wrapIfNecessary方法生成了TestService的代理对象返回，此时我们在代码中使用@Autowired注入TestService时，实际注入的是这个代理对象。

因此我们如果在不同的类之间调用带事务注解的方法，即在OtherService中调用TestService#testSave，因在OtherService中注入了TestService，所以testSave实际是通过TestService的代理对象调用的，由于TransactionInterceptor实现了MethodInterceptor，因此会在代理对象方法调用时进入拦截器，而Spring在同类内的两个方法间互相调用是不会使用代理对象的，因此不会进入此拦截器，这也是声明式事务失效的原因。

## 总结
经过一顿折腾，终于调查清楚了Spring声明式事务的执行原理以及处理流程，建议有心的同学最好自己先试着跟踪代码一步一步走下去，再去看理解会更深入一些。

在Spring官网的介绍中也有说明这个问题，详见[transaction-declarative-annotations](https://docs.spring.io/spring-framework/docs/4.3.13.RELEASE/spring-framework-reference/htmlsingle/#transaction-declarative-annotations) 17.5.6 Using @Transactional部分，见下图。

![Spring官网说明节选](/upload/Spring声明式事务失效/1.png)

## 参考资料
[spring事务@Transactional在同一个类中的方法调用不生效](https://blog.csdn.net/qq_30336433/article/details/83338835)
[Spring事务管理（详解+实例）](https://blog.csdn.net/ligeforrent/article/details/79223673)
[Spring Framework Reference Documentation](https://docs.spring.io/spring-framework/docs/4.3.13.RELEASE/spring-framework-reference/htmlsingle)
[pring中autowired注入自己的代理后，最后容器中的对象是原对象还是代理对象](https://blog.csdn.net/qq_39002724/article/details/113609903)
