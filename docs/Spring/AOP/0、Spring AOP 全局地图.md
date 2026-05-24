# 0、Spring AOP 全局地图

## 先建立边界：AOP 学习先看一条主线

> [!note]
> 这篇只做一件事：把 AOP 从“开启”到“方法调用增强”的主线串起来。
> 
> 三篇现有文档不再单独列清单，而是直接挂到主线的对应节点上。这样看地图时，就能顺手知道下一步该读哪篇。

## 完整生命周期地图：从开启 AOP 到方法调用

- [00] 代理基础：代理对象为什么能接住方法调用
  - 对应文档：[[JDK动态代理的实现原理解析]]
  - 解决的问题：代理对象为什么能接住方法调用。
  - 核心直觉：`$Proxy0.method()` 不是直接调用目标对象，而是回调 `InvocationHandler.invoke(...)`。
  - 关键对象：
    - `Proxy`
    - `InvocationHandler`
    - `$Proxy0`
  - 关键问题：
    - 为什么调用代理对象的方法，会进入 `InvocationHandler.invoke(...)`
    - Spring AOP 里的 `JdkDynamicAopProxy.invoke(...)` 后面为什么能接住方法调用
  - 注意：这篇不直接等于 Spring AOP，但它是理解 `JdkDynamicAopProxy.invoke(...)` 的前置知识。

- [01] AOP 入口：把 AutoProxyCreator 注册进容器
  - 对应文档：[[Spring-Aop如何生效]]
  - 现有文档重点：XML 入口 `<aop:aspectj-autoproxy>`。
  - 入口不止一种：
    - 注解入口：`@EnableAspectJAutoProxy`
      - `@EnableAspectJAutoProxy`
        - 作用：打开基于注解的 Spring AOP 自动代理能力。
        - 关键点：它本身不是直接创建代理，而是通过 `@Import` 导入注册器。
        - 源码位置：`org.springframework.context.annotation.EnableAspectJAutoProxy`
        - 下一步：导入 `AspectJAutoProxyRegistrar`
          - `AspectJAutoProxyRegistrar`
            - 关系：实现 `ImportBeanDefinitionRegistrar`。
            - 作用：在配置类解析阶段，向当前 `BeanDefinitionRegistry` 注册 AOP 自动代理创建器。
            - 源码位置：`org.springframework.context.annotation.AspectJAutoProxyRegistrar`
            - 核心方法：`registerBeanDefinitions(importingClassMetadata, registry)`
              - 调用：`AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry)`
                - `AopConfigUtils`
                  - 关系：AOP 自动代理创建器的注册工具类。
                  - 作用：把 `AnnotationAwareAspectJAutoProxyCreator` 注册成一个基础设施 BeanDefinition。
                  - 源码位置：`org.springframework.aop.config.AopConfigUtils`
                  - 核心方法：`registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`
                    - 调用：`registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source)`
                      - `AnnotationAwareAspectJAutoProxyCreator`
                        - 关系：注解 AOP 的核心自动代理创建器。
                        - 作用：后面作为 BeanPostProcessor 参与 Bean 创建，判断 Bean 是否需要被 AOP 代理。
                        - 源码位置：`org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator`
                      - `AUTO_PROXY_CREATOR_BEAN_NAME`
                        - 作用：Spring 内部固定的自动代理创建器 beanName。
                        - 说明：如果容器里已经有自动代理创建器，Spring 会按优先级升级，而不是重复注册多个同类创建器。
                      - `BeanDefinitionRegistry.registerBeanDefinition(...)`
                        - 结果：容器里多了一个 `AnnotationAwareAspectJAutoProxyCreator` 的 BeanDefinition。
    - XML 入口：`<aop:aspectj-autoproxy>`
      - `AspectJAutoProxyBeanDefinitionParser`
        - `AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`
          - `AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`
      - 对应现有笔记：[[Spring-Aop如何生效]]
    - Spring Boot 入口：`AopAutoConfiguration`
      - 作用：在满足条件时自动引入 `@EnableAspectJAutoProxy`。
      - 常见触发：引入 `spring-boot-starter-aop`。
    - 事务入口：`@EnableTransactionManagement`
      - 说明：这不是 `@Aspect` 注解切面入口，但也是通过代理 / Advisor / Interceptor 思路实现事务增强。
      - 常见结果：事务 Advisor + `TransactionInterceptor` + AutoProxyCreator。
  - 项目对应：
    - `biz`：`@ImportResource(...)` 导入 XML，XML 里使用 `<aop:aspectj-autoproxy>`。
    - `slt`：引入 `spring-boot-starter-aop`，Spring Boot 通过 `AopAutoConfiguration` 间接启用 AOP。
    - `biz_boot`：大量 `@Transactional`，主要看 `@EnableTransactionManagement` 的事务代理路径。
  - 汇合点：
    - 不同入口最终都是为了注册某种 AutoProxyCreator。
    - AutoProxyCreator 后面会作为 BeanPostProcessor 参与 Bean 创建。

- [02] AutoProxyCreator 进入 BeanPostProcessor 体系
  - 对应 IoC 文档：[[BeanPostProcessor]]
  - 关键对象：`AnnotationAwareAspectJAutoProxyCreator`
  - 继承主线：
    - `AnnotationAwareAspectJAutoProxyCreator`
      - `AspectJAwareAdvisorAutoProxyCreator`
        - `AbstractAdvisorAutoProxyCreator`
          - `AbstractAutoProxyCreator`
  - 关键点：
    - 它不是普通业务 Bean。
    - 它会参与其他 Bean 的创建过程。
    - 它的重点方法在 `AbstractAutoProxyCreator` 里。

- [03] Bean 创建阶段：判断当前 Bean 要不要被代理
  - 对应 IoC 文档：
    - [[4、依赖注入(DI)]]
    - [[三级缓存为什么和 AOP 代理有关]]
  - 普通非循环依赖：
    - `initializeBean(...)`
      - `applyBeanPostProcessorsAfterInitialization(...)`
        - `AbstractAutoProxyCreator.postProcessAfterInitialization(...)`
          - `wrapIfNecessary(...)`
            - 查找 Advisor / Advice
            - 判断 Pointcut 是否匹配当前 Bean
            - 匹配则创建代理
            - 不匹配则返回原始 Bean
  - 循环依赖提前暴露：
    - `addSingletonFactory(...)`
      - `ObjectFactory.getObject()`
        - `AbstractAutoProxyCreator.getEarlyBeanReference(...)`
          - `wrapIfNecessary(...)`
            - 如果最终应该被代理，就提前返回代理对象
            - 如果不需要代理，就返回原始对象
  - 关键点：
    - `postProcessAfterInitialization(...)` 是普通路径。
    - `getEarlyBeanReference(...)` 是循环依赖提前暴露路径。
    - 两条路径都会进入 `wrapIfNecessary(...)`，只是触发时机不同。

- [04] 代理创建阶段：把 raw bean 包成 proxy bean
  - 对应文档：[[AOP源码实现及分析#2 Spring AOP 的设计与实现]]
  - 现有文档重点：`ProxyFactoryBean`、`ProxyFactory`、JDK 代理、CGLIB 代理。
  - 关键对象：
    - `ProxyFactory`
    - `AdvisedSupport`
    - `DefaultAopProxyFactory`
    - `JdkDynamicAopProxy`
    - `CglibAopProxy`
  - 关键问题：
    - 当前 Bean 命中了哪些 Advisor
    - 使用 JDK 代理还是 CGLIB 代理
    - 代理对象内部怎么持有目标对象和增强信息
  - 具体对应：
    - [[AOP源码实现及分析#2.2 为配置的 target 生成 AopProxy 代理对象]]
    - [[AOP源码实现及分析#2.5 JDK 动态代理 生成 AopProxy 代理对象]]
    - [[AOP源码实现及分析#2.6 CGLIB 生成 AopProxy 代理对象]]
  - 注意：[[AOP源码实现及分析]] 有些内容从 `ProxyFactoryBean` 这种老式入口讲起，不要把入口形式和 AOP 主线混为一谈。真正要抓的是“命中 Advisor 后创建代理”这条主线。

- [05] 方法调用阶段：调用 proxy.method() 后进入增强链
  - 对应文档：[[AOP源码实现及分析#3 Spring AOP 拦截器调用的实现]]
  - JDK 代理：
    - `proxy.method()`
      - `$Proxy0.method()`
        - `InvocationHandler.invoke(...)`
          - `JdkDynamicAopProxy.invoke(...)`
  - CGLIB 代理：
    - `proxy.method()`
      - CGLIB 生成的子类方法
        - `MethodInterceptor.intercept(...)`
          - `CglibAopProxy.DynamicAdvisedInterceptor.intercept(...)`
  - 后续共同进入：
    - 根据当前方法获取拦截器链
      - `ReflectiveMethodInvocation.proceed()`
        - 前置增强
        - 目标方法调用
        - 后置 / 返回 / 异常增强
  - 具体对应：
    - [[AOP源码实现及分析#3.1 JdkDynamicAopProxy 的 invoke() 拦截]]
    - [[AOP源码实现及分析#3.2 CglibAopProxy 的 intercept() 拦截]]
    - [[AOP源码实现及分析#3.4 AOP 拦截器链的调用]]
  - 关键点：
    - 代理对象创建出来时，不等于增强逻辑已经执行。
    - 真正的增强逻辑发生在业务代码调用 `proxy.method()` 时。

## 这一篇最终要记住什么

- AOP 主线不要按文件硬读，要按生命周期读。
- [[JDK动态代理的实现原理解析]] 放在 [00]，解决代理对象怎么接住调用。
- [[Spring-Aop如何生效]] 放在 [01]，解决 XML AOP 入口怎么注册 AutoProxyCreator。
- [[AOP源码实现及分析]] 放在 [04] 和 [05]，解决代理创建和拦截器链执行。
- 不同入口最终都要汇合到 AutoProxyCreator。
- AutoProxyCreator 后面作为 BeanPostProcessor 参与 Bean 创建。
- 普通情况在 `postProcessAfterInitialization(...)` 创建代理。
- 循环依赖情况可能在 `getEarlyBeanReference(...)` 提前创建代理。
