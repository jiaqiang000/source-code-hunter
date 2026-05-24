# 0、Spring AOP 全局地图

## 先建立边界：这一篇先解决注解 AOP 怎么接入 Spring

> [!note]
> 这篇先只建立全局位置：`@EnableAspectJAutoProxy` 怎么把 AOP 能力接入 Spring 容器。
> 
> 暂时不展开 Advisor 解析、JDK / CGLIB 代理创建、拦截器链执行细节。后面这些可以继续对应到 [[AOP源码实现及分析]] 和 [[JDK动态代理的实现原理解析]]。

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

## 这一阶段先记住什么

- `@EnableAspectJAutoProxy` 不直接代理业务 Bean。
- 它先导入 `AspectJAutoProxyRegistrar`。
- `AspectJAutoProxyRegistrar` 注册 `AnnotationAwareAspectJAutoProxyCreator`。
- `AnnotationAwareAspectJAutoProxyCreator` 后面作为 BeanPostProcessor 参与 Bean 创建。
- 普通情况在 `postProcessAfterInitialization(...)` 创建代理。
- 循环依赖情况可能在 `getEarlyBeanReference(...)` 提前创建代理。
