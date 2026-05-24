# 0、Spring AOP 全局地图

## 先建立边界：这一篇要把三篇 AOP 文档串起来

> [!note]
> 这篇不是单独讲 `@EnableAspectJAutoProxy`。
> 
> 它的作用是先把 AOP 主线定住：AOP 怎么开启、AutoProxyCreator 怎么进入 IoC、Bean 什么时候被包装成代理、代理方法调用时怎么进入增强链。

## 三篇文档在 AOP 主线中的位置

- [[JDK动态代理的实现原理解析]]
  - 解决的问题：代理对象为什么能接住方法调用。
  - 核心直觉：`$Proxy0.method()` 不是直接调用目标对象，而是回调 `InvocationHandler.invoke(...)`。
  - 它不直接等于 Spring AOP，但它是理解 `JdkDynamicAopProxy.invoke(...)` 的前置知识。
  - 建议先看：代理对象、`InvocationHandler`、`Proxy.newProxyInstance(...)`、`$Proxy0`。

- [[Spring-Aop如何生效]]
  - 解决的问题：AOP 开关怎么注册进 Spring 容器。
  - 现有文档重点：XML 入口 `<aop:aspectj-autoproxy>`。
  - 它主要对应本地图里的“入口注册阶段”。
  - 需要注意：你的项目不一定显式写 `<aop:aspectj-autoproxy>`，Spring Boot 也可能通过自动配置间接打开 AOP。

- [[AOP源码实现及分析]]
  - 解决的问题：Spring AOP 主体源码怎么跑。
  - 它覆盖：`Advice`、`Pointcut`、`Advisor`、`ProxyFactory`、JDK 代理、CGLIB 代理、拦截器链。
  - 它主要对应本地图里的“代理创建阶段”和“代理方法调用阶段”。
  - 需要注意：这篇有些内容从 `ProxyFactoryBean` 这种老式入口讲起，不要把入口形式和 AOP 主线混为一谈。

## 完整生命周期地图：从开启 AOP 到方法调用

- [00] 代理基础：先知道代理对象怎么接住方法调用
  - 对应文档：[[JDK动态代理的实现原理解析]]
  - 关键对象：`Proxy`、`InvocationHandler`、`$Proxy0`
  - 关键问题：
    - 为什么调用代理对象的方法，会进入 `InvocationHandler.invoke(...)`
    - Spring AOP 里的 `JdkDynamicAopProxy.invoke(...)` 后面为什么能接住方法调用

- [01] AOP 入口：把 AutoProxyCreator 注册进容器
  - 注解入口：`@EnableAspectJAutoProxy`
    - 对应本篇：[[#注解 AOP 入口路径：从 @EnableAspectJAutoProxy 到 AutoProxyCreator]]
  - XML 入口：`<aop:aspectj-autoproxy>`
    - 对应文档：[[Spring-Aop如何生效]]
  - Spring Boot 入口：`AopAutoConfiguration`
    - 作用：在满足条件时自动引入 `@EnableAspectJAutoProxy`
  - 事务入口：`@EnableTransactionManagement`
    - 说明：这不是 `@Aspect` 注解切面入口，但也是通过代理 / Advisor / Interceptor 思路实现事务增强。
  - 共同目标：
    - 注册某种 AutoProxyCreator。
    - 让它后面作为 BeanPostProcessor 参与 Bean 创建。

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
  - 对应 IoC 文档：[[4、依赖注入(DI)]]
  - 普通非循环依赖：
    - `initializeBean(...)`
      - `applyBeanPostProcessorsAfterInitialization(...)`
        - `AbstractAutoProxyCreator.postProcessAfterInitialization(...)`
          - `wrapIfNecessary(...)`
            - 查找 Advisor / Advice
            - 判断 Pointcut 是否匹配当前 Bean
            - 匹配则创建代理
  - 循环依赖提前暴露：
    - `addSingletonFactory(...)`
      - `ObjectFactory.getObject()`
        - `AbstractAutoProxyCreator.getEarlyBeanReference(...)`
          - `wrapIfNecessary(...)`
            - 如果最终应该被代理，就提前返回代理对象
  - 对应 IoC 文档：[[三级缓存为什么和 AOP 代理有关]]

- [04] 代理创建阶段：把 raw bean 包成 proxy bean
  - 对应文档：[[AOP源码实现及分析#2 Spring AOP 的设计与实现]]
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

## 入口不同，但后面主线要合并看

- `biz` 这类老 Spring 项目
  - 常见入口：`@ImportResource(...)` 导入 XML。
  - XML 里使用 `<aop:aspectj-autoproxy>`。
  - 对应主线：[01] XML 入口。

- `slt` 这类 Spring Boot 项目
  - 常见入口：引入 `spring-boot-starter-aop`。
  - Spring Boot 通过 `AopAutoConfiguration` 间接启用 `@EnableAspectJAutoProxy`。
  - 对应主线：[01] Spring Boot 入口。

- `biz_boot` 里大量 `@Transactional`
  - 常见入口：`@EnableTransactionManagement`。
  - 它不一定出现 `@Aspect` 或 `@EnableAspectJAutoProxy`。
  - 但它仍然会走代理增强思路：事务 Advisor + `TransactionInterceptor` + AutoProxyCreator。
  - 后面仍然会和 [03] Bean 创建阶段、[04] 代理创建阶段、[05] 方法调用阶段发生关系。

## 注解 AOP 入口路径：从 @EnableAspectJAutoProxy 到 AutoProxyCreator

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

## 入口路径和 IoC 生命周期怎么接起来

- 配置类解析阶段
  - `@EnableAspectJAutoProxy`
    - 导入 `AspectJAutoProxyRegistrar`
      - 注册 `AnnotationAwareAspectJAutoProxyCreator` 的 BeanDefinition
        - 此时只是注册定义，还没有开始给业务 Bean 创建代理。

- BeanFactory 准备阶段
  - Spring 会把 `AnnotationAwareAspectJAutoProxyCreator` 创建出来。
  - 它会被加入 BeanPostProcessor 体系。
  - 对应 IoC 笔记：[[BeanPostProcessor]]

- 普通 Bean 创建阶段
  - 业务 Bean 正常实例化、属性填充、初始化。
  - 初始化后处理时进入：
    - `AbstractAutoProxyCreator.postProcessAfterInitialization(...)`
      - 调用：`wrapIfNecessary(...)`
        - 查找当前 Bean 是否命中 Advisor / Advice。
        - 命中则调用 `createProxy(...)` 创建代理。
        - 没命中则返回原始 Bean。
  - 对应 IoC 笔记：[[4、依赖注入(DI)]]

- 循环依赖提前暴露阶段
  - 如果 A 创建过程中提前暴露，B 又回头依赖 A。
  - 三级缓存里的 `ObjectFactory` 会触发：
    - `AbstractAutoProxyCreator.getEarlyBeanReference(...)`
      - 调用：`wrapIfNecessary(...)`
        - 如果 A 最终应该被代理，这里提前返回代理对象。
        - 如果 A 不需要代理，这里返回原始对象。
  - 对应 IoC 笔记：[[三级缓存为什么和 AOP 代理有关]]

## 注解路径和 XML 路径在哪里汇合

- 注解路径
  - `@EnableAspectJAutoProxy`
    - `AspectJAutoProxyRegistrar`
      - `AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`

- XML 路径
  - `<aop:aspectj-autoproxy>`
    - `AspectJAutoProxyBeanDefinitionParser`
      - `AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`
        - `AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`
  - 对应现有笔记：[[Spring-Aop如何生效]]

- 汇合点
  - 两条路最终都会注册 `AnnotationAwareAspectJAutoProxyCreator`。
  - 所以“注解 AOP”和“XML AOP”入口不同，但后面都会进入 AutoProxyCreator 参与 Bean 创建的主线。

## 这一篇最终要记住什么

- [[JDK动态代理的实现原理解析]] 解决“代理对象为什么能接住方法调用”。
- [[Spring-Aop如何生效]] 解决“XML AOP 开关怎么注册 AutoProxyCreator”。
- [[AOP源码实现及分析]] 解决“Advisor / ProxyFactory / JDK / CGLIB / 拦截器链怎么跑”。
- `@EnableAspectJAutoProxy`、`<aop:aspectj-autoproxy>`、Spring Boot AOP 自动配置，入口不同，但最终都要把 AutoProxyCreator 放进 Spring 容器。
- AutoProxyCreator 后面作为 BeanPostProcessor 参与 Bean 创建。
- 普通情况在 `postProcessAfterInitialization(...)` 创建代理。
- 循环依赖情况可能在 `getEarlyBeanReference(...)` 提前创建代理。
- 代理对象创建出来之后，真正的增强逻辑不是创建时执行，而是业务代码调用 `proxy.method()` 时进入 `invoke(...)` / `intercept(...)` 再执行。
