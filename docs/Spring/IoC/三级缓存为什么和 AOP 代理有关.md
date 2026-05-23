# 三级缓存为什么和 AOP 代理有关

## 这一篇解决什么问题

这篇文档解决一个连接问题：

```text
为什么讲单例属性循环依赖时，会突然牵涉到 AOP 代理？
为什么三级缓存里不直接放原始 Bean，而是放 ObjectFactory？
getEarlyBeanReference 到底在这条链里做什么？
createProxy 创建出来的代理对象，后面又是怎么真正执行增强逻辑的？
```

它连接的是几篇文档：

- [[4、依赖注入(DI)]]：Bean 创建主流程。
- [[循环依赖]]：A/B 互相依赖时，三级缓存怎么解循环。
- [[BeanPostProcessor]]：为什么 Spring 可以在 Bean 创建过程中插入扩展点。
- [[Spring-Aop如何生效]]：AOP 自动代理创建器怎么被注册进容器。
- [[AOP源码实现及分析]]：代理对象和拦截器链的后半段。

## 先建立边界：本文不是完整 AOP

本文不是从头讲 AOP，也不展开完整的切点解析、事务拦截器、Advisor 查找细节。

本文只追一条线：

```text
循环依赖导致 Bean 必须提前暴露
  -> 提前暴露时不能随便暴露原始对象
    -> 如果 Bean 最终应该是 AOP 代理，就要提前暴露代理对象
      -> 这就是三级缓存和 AOP 代理发生关系的地方
```


普通循环依赖只需要回答：

```text
A 还没完整创建，B 又需要 A，怎么先把 A 给 B？
```

AOP 场景还要多回答一个问题：

```text
给 B 的 A，到底应该是原始 A，还是代理 A？
```

如果给错了，可能出现：

```text
B.a = 原始 A
容器 getBean("a") = 代理 A
```

这会让 B 调用 A 时绕开事务、AOP 增强等代理逻辑。

## 先看对象关系：这条链上谁负责什么

```text
[对象1] raw bean
        关系：createBeanInstance() 刚创建出来的原始 Java 对象
        作用：还没有完成属性填充，也还没有完成初始化

[对象2] singletonFactories
        所属类：DefaultSingletonBeanRegistry
        关系：三级缓存
        作用：保存 ObjectFactory，不直接保存 raw bean

[对象3] ObjectFactory
        关系：放在 singletonFactories 里的工厂对象
        作用：等别人真的需要早期引用时，再执行 getObject()

[对象4] getEarlyBeanReference(...)
        所属类：AbstractAutowireCapableBeanFactory
        关系：ObjectFactory.getObject() 被触发时进入
        作用：决定早期暴露出去的是原始对象，还是被后处理器包装过的对象

[对象5] SmartInstantiationAwareBeanPostProcessor
        关系：BeanPostProcessor 的扩展接口
        作用：提供 getEarlyBeanReference(...) 扩展点

[对象6] AbstractAutoProxyCreator
        关系：AOP 自动代理创建器，也是 SmartInstantiationAwareBeanPostProcessor
        作用：循环依赖触发早期引用时，如果当前 Bean 需要 AOP/事务代理，就提前创建代理

[对象7] ProxyFactory
        关系：createProxy() 中创建
        作用：保存 TargetSource、Advisors、代理配置，然后生成 JDK/CGLIB 代理对象

[对象8] TargetSource
        关系：代理对象调用目标方法时，通过它拿真实目标对象
        作用：早期代理场景常见的是 SingletonTargetSource(raw bean)

[对象9] MethodInterceptor 链
        关系：代理方法被调用时，根据当前 method 从 Advisors 中筛出来
        作用：按顺序执行增强逻辑，最后调用目标对象方法
```

一句话概括：

```text
三级缓存保存的是“早期引用工厂”。
getEarlyBeanReference 决定早期引用是什么。
AbstractAutoProxyCreator 可以把早期引用变成代理对象。
代理对象被调用时，才通过拦截器链调用原始 Bean 的目标方法。
```

## 全局导图：从 addSingletonFactory 到代理方法调用

> [!note] 前置流程在哪里看
> 本文从 `doCreateBean("a")` 内部开始追，因为三级缓存和 AOP 代理的连接点发生在 `addSingletonFactory(...)` 之后。
>
> 如果要先看 `doCreateBean("a")` 之前是怎么走到这里的，回到 [[4、依赖注入(DI)#全局导图：从触发 getBean 到属性写入]]。当前图里的 `[00] beanFactory::doCreateBean("a")`，对应那篇导图里的 `[04] beanFactory::doCreateBean(...)`。

> [!note] 读这张图时先盯住两个时间点
> `addSingletonFactory(...)` 只是把 ObjectFactory 存进去，不执行 lambda。
> 只有 B 回头找正在创建中的 A 时，`getSingleton("a")` 才会执行 ObjectFactory。

```text
[00] beanFactory::doCreateBean("a")
     方法定义在：AbstractAutowireCapableBeanFactory
     对应正文：00 AbstractAutowireCapableBeanFactory：A 先被实例化，但还没完整创建
     作用：创建单个 Bean 的主流程，先得到原始对象 rawA
     │
     ├─ [00.1] createBeanInstance("a")
     │        作用：先创建原始对象 rawA
     │        状态：rawA 已经 new 出来，但还没有属性填充和初始化
     │
     ├─ [00.2] earlySingletonExposure 判断
     │        条件：singleton + allowCircularReferences + 正在创建中
     │        作用：决定是否允许把当前 Bean 的早期引用工厂放入三级缓存
     │
     ├─ [01] addSingletonFactory("a", ObjectFactory)
     │        方法定义在：DefaultSingletonBeanRegistry
     │        对应正文：01 AbstractAutowireCapableBeanFactory：addSingletonFactory 只是登记早期引用工厂
     │        作用：在 doCreateBean() 中登记早期引用工厂
     │        │
     │        └─ [02] DefaultSingletonBeanRegistry::addSingletonFactory("a", ObjectFactory)
     │           对应正文：02 DefaultSingletonBeanRegistry：ObjectFactory 被放入三级缓存
     │           作用：把 ObjectFactory 放入 singletonFactories
     │           注意：这里只保存 lambda，不执行 getEarlyBeanReference
     │
     ├─ populateBean("a")
     │        方法定义在：AbstractAutowireCapableBeanFactory
     │        作用：填充 A 的属性
     │        │
     │        └─ A 需要 B
     │           │
     │           └─ getBean("b")
     │              作用：递归创建 B
     │              │
     │              └─ beanFactory::doCreateBean("b")
     │                 作用：创建 B
     │                 │
     │                 ├─ createBeanInstance("b")
     │                 │  作用：先创建原始对象 rawB
     │                 │
     │                 ├─ addSingletonFactory("b", ObjectFactory)
     │                 │  作用：B 也先把自己的早期引用工厂放进三级缓存
     │                 │
     │                 ├─ populateBean("b")
     │                 │  作用：填充 B 的属性
     │                 │  │
     │                 │  └─ B 需要 A
     │                 │     │
     │                 │     └─ getBean("a")
     │                 │        │
     │                 │        └─ [03] getSingleton("a")
     │                 │           方法定义在：DefaultSingletonBeanRegistry
     │                 │           对应正文：03 DefaultSingletonBeanRegistry：什么时候才执行 ObjectFactory
     │                 │           作用：B 回头找正在创建中的 A，并触发三级缓存中的 ObjectFactory
     │                 │           │
     │                 │           ├─ 查 singletonObjects
     │                 │           │  结果：没有完整 A
     │                 │           │
     │                 │           ├─ 查 earlySingletonObjects
     │                 │           │  结果：没有已经取过的早期 A
     │                 │           │
     │                 │           └─ 查 singletonFactories
     │                 │              结果：找到 A 的 ObjectFactory
     │                 │              │
     │                 │              ├─ singletonFactory.getObject()
     │                 │              │  作用：这时才真正执行 01 中保存的 lambda
     │                 │              │  │
     │                 │              │  └─ [04] beanFactory::getEarlyBeanReference("a", mbd, rawA)
     │                 │              │     方法定义在：AbstractAutowireCapableBeanFactory
     │                 │              │     对应正文：04 AbstractAutowireCapableBeanFactory：getEarlyBeanReference 决定早期引用是什么
     │                 │              │     作用：生成准备提前暴露的 A
     │                 │              │     │
     │                 │              │     └─ 遍历 SmartInstantiationAwareBeanPostProcessor
     │                 │              │        │
     │                 │              │        └─ [05] 遍历 SmartInstantiationAwareBeanPostProcessor 扩展点
     │                 │              │           对应正文：05 SmartInstantiationAwareBeanPostProcessor：早期引用扩展点
     │                 │              │           关系：这是 Spring 留给后处理器改写早期引用的接口
     │                 │              │           作用：让每个处理器都有机会把 earlyA 从 rawA 改成包装对象
     │                 │              │           │
     │                 │              │           ├─ 默认接口实现
     │                 │              │           │  作用：不改对象，直接返回传入的 bean
     │                 │              │           │
     │                 │              │           └─ [06] AbstractAutoProxyCreator::getEarlyBeanReference(rawA, "a")
     │                 │              │              对应正文：06 AbstractAutoProxyCreator：AOP 场景提前创建代理
     │                 │              │              关系：它是 SmartInstantiationAwareBeanPostProcessor 的一个实现
     │                 │              │              作用：AOP/事务场景下，可能把 rawA 提前包装成 proxyA
     │                 │              │              │
     │                 │              │              └─ [07] wrapIfNecessary(rawA, "a", cacheKey)
     │                 │              │                 对应正文：07 AbstractAutoProxyCreator：wrapIfNecessary 判断是否需要代理
     │                 │              │                 作用：判断 A 是否需要被代理
     │                 │              │                 │
     │                 │              │                 ├─ 不需要代理
     │                 │              │                 │  │
     │                 │              │                 │  └─ 返回 rawA
     │                 │              │                 │     结果：B.a = rawA
     │                 │              │                 │     注意：后面 initializeBean() 仍会执行，BeanPostProcessor 初始化后回调也仍会执行；
     │                 │              │                 │           这里只是 AbstractAutoProxyCreator 不会再对 A 重复创建 AOP 代理
     │                 │              │                 │
     │                 │              │                 └─ 需要代理
     │                 │              │                    │
     │                 │              │                    └─ [08] createProxy(beanClass, beanName, advisors, SingletonTargetSource(rawA))
     │                 │              │                       对应正文：08 AbstractAutoProxyCreator：createProxy 只是创建代理，不执行方法
     │                 │              │                       作用：创建代理对象 proxyA
     │                 │              │                       注意：这里只创建代理，不执行目标方法
     │                 │              │                       │
     │                 │              │                       └─ [09] ProxyFactory / DefaultAopProxyFactory
     │                 │              │                          对应正文：09 ProxyFactory 和 DefaultAopProxyFactory：选择 JDK 还是 CGLIB
     │                 │              │                          作用：选择 JDK 动态代理或 CGLIB 代理，返回 proxyA
     │                 │              │                          结果：B.a = proxyA
     │                 │              │                          注意：后面 initializeBean() 仍会执行；
     │                 │              │                                这里只是 AbstractAutoProxyCreator 不会再对 A 重复创建另一个 AOP 代理
     │                 │              │
     │                 │              └─ getSingleton("a") 收到 earlyA
     │                 │                 earlyA：rawA 或 proxyA
     │                 │                 作用：放入 earlySingletonObjects，并移除 singletonFactories 中的 ObjectFactory
     │                 │                 │
     │                 │                 └─ 返回 earlyA 给 B
     │                 │                    │
     │                 │                    └─ B.a = earlyA
     │                 │                       说明：如果 A 需要代理，这里的 earlyA 就应该是 proxyA
     │                 │
     │                 └─ B 继续完成
     │                    │
     │                    ├─ populateBean("b") 完成
     │                    ├─ initializeBean("b") 完成
     │                    └─ getSingleton("b", ObjectFactory) 外层完成后调用 addSingleton("b", exposedB)
     │                       作用：B 进入 singletonObjects 一级缓存，并清理 B 的 singletonFactories / earlySingletonObjects
     │
     ├─ A 拿到 B 后继续完成
     │  │
     │  ├─ A.b = B
     │  ├─ populateBean("a") 完成
     │  ├─ initializeBean("a")
     │  │        作用：继续执行 A 的初始化流程
     │  │        │
     │  │        └─ AbstractAutoProxyCreator::postProcessAfterInitialization(rawA, "a")
     │  │           作用：普通 AOP / 事务代理通常在这里进入 wrapIfNecessary()
     │  │           对照：如果前面 06 已经走过 getEarlyBeanReference，
     │  │                 earlyProxyReferences 会让这里不再重复创建 AOP 代理
     │  │           注意：这里说的是“不重复创建 AOP 代理”，不是跳过 initializeBean()，
     │  │                 也不是跳过所有 BeanPostProcessor
     │  ├─ getSingleton("a", false)
     │  │        方法定义在：AbstractAutowireCapableBeanFactory#doCreateBean
     │  │        对应正文：09.1 回到 doCreateBean：用 earlyA 作为最终 exposedObject
     │  │        作用：检查 A 是否已经因为循环依赖被提前拿过早期引用
     │  │        注意：allowEarlyReference=false，不会再次触发 singletonFactory.getObject()
     │  │        │
     │  │        └─ 如果拿到 earlyA，并且 exposedObject 还是 rawA
     │  │           │
     │  │           └─ exposedObject = earlyA
     │  │              作用：让容器最终暴露的 A 和 B.a 注入的早期 A 保持一致
     │  │
     │  └─ getSingleton("a", ObjectFactory) 外层完成后调用 addSingleton("a", exposedA)
     │     作用：A 进入 singletonObjects 一级缓存，并清理 A 的 singletonFactories / earlySingletonObjects
     │
     └─ Bean 创建结束
        │
        └─ 后续运行时：只有业务代码调用 proxyA.method() 时，才进入 10-13
           说明：09 创建出 proxyA，09.1 保证最终暴露对象和早期注入对象一致；
                 事务、日志切面、权限切面和 rawA.method() 都不会在 createProxy() 时执行，
                 而是在这里的业务方法调用时执行
           边界：10-13 不是 Bean 创建过程本身，而是 Bean 创建完成后的代理方法调用链路
           Web 请求入口可以对照：09.2 从请求到代理 invoke/intercept 的调用链
           注意：如果前面返回的是 rawA 而不是 proxyA，就没有 JDK/CGLIB 代理调用入口
           │
           ├─ [10] JdkDynamicAopProxy::invoke(...)
           │  对应正文：10 JdkDynamicAopProxy：JDK 代理被调用时才进入增强链
           │  作用：JDK 代理对象被调用时进入这里
           │  │
           │  └─ 继续进入 [12]
           │
           ├─ [11] CglibAopProxy 的 MethodInterceptor::intercept(...)
           │  对应正文：11 CglibAopProxy：CGLIB 代理也是同一套 proceed 链路
           │  作用：CGLIB 代理对象被调用时进入这里
           │  │
           │  └─ 继续进入 [12]
           │
           └─ [12] getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)
              对应正文：12 AdvisedSupport 和 DefaultAdvisorChainFactory：Advisor 变成 MethodInterceptor 链
              作用：根据当前方法筛选 MethodInterceptor 链
              │
              └─ [13] ReflectiveMethodInvocation::proceed()
                 对应正文：13 ReflectiveMethodInvocation：proceed 怎么执行增强和目标方法
                 作用：用 currentInterceptorIndex 推进拦截器链；每个拦截器通过 proceed() 把调用权交给下一层
                 模型：拦截器链一层层包住目标方法，不是简单“先全部前置，再目标方法，再全部后置”
                 │
                 ├─ currentInterceptorIndex 初始为 -1
                 │
                 ├─ 第一次 proceed()
                 │  └─ Interceptor1::invoke(invocation)
                 │     │
                 │     ├─ Interceptor1 前置逻辑
                 │     ├─ invocation.proceed()
                 │     │  │
                 │     │  └─ Interceptor2::invoke(invocation)
                 │     │     │
                 │     │     ├─ Interceptor2 前置逻辑
                 │     │     ├─ invocation.proceed()
                 │     │     │  │
                 │     │     │  └─ 所有拦截器执行完
                 │     │     │     └─ invokeJoinpoint()
                 │     │     │        └─ rawA.method()
                 │     │     │
                 │     │     └─ Interceptor2 后置逻辑
                 │     │
                 │     └─ Interceptor1 后置逻辑
                 │
                 ├─ 如果某个 Interceptor 不调用 invocation.proceed()
                 │  └─ 后面的拦截器和 rawA.method() 可能不会执行
                 │
                 └─ [14] 回到三级缓存职责
                    对应正文：14 回到三级缓存：为什么第三级缓存要存 ObjectFactory
                    作用：把 singletonFactories、earlySingletonObjects、singletonObjects 的职责重新合起来
```

## 先补一层：BeanPostProcessor、早期代理和重复包装

这一节解决的是看完全局导图后最容易冒出来的问题：

```text
循环依赖里，A 已经在 06-09 走过早期代理判断了。
那回到 A 自己的 initializeBean(...) 时，还会不会继续调用 BeanPostProcessor？
AbstractAutoProxyCreator 还会不会再次 wrapIfNecessary？
earlyProxyReferences 到底存的是什么？
多个增强是不是就等于代理包代理？
```

### BeanPostProcessor 是什么：初始化前后可处理、也可替换 Bean 的扩展点

这里先说普通 `BeanPostProcessor`。

它是 Spring Bean 创建过程里的扩展点，不是业务方法拦截器。普通 `BeanPostProcessor` 主要围绕 Bean 初始化前后工作：

```java
postProcessBeforeInitialization(...)
postProcessAfterInitialization(...)
```

这个阶段一般发生在 Bean 已经实例化、属性也基本填充完之后。

它可以只做处理并返回原对象，比如检查标记接口、处理注解元数据、注册内部状态等；也可以返回一个新对象，比如包装对象或代理对象。

所以不要把 `BeanPostProcessor` 直接理解成“专门用来创建代理”的东西。更准确是：

```text
BeanPostProcessor 是初始化前后扩展点；
它可以只处理对象，也可以替换对象。
```

后面的 `AbstractAutoProxyCreator` 比普通 `BeanPostProcessor` 更特殊，因为它实现的是 `SmartInstantiationAwareBeanPostProcessor`，所以除了初始化后代理，还能参与循环依赖里的早期引用。

源码里关键点是：

```java
Object result = existingBean;
for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessAfterInitialization(result, beanName);
    if (current == null) {
        return result;
    }
    result = current;
}
return result;
```

注意这里是：

```text
result = current
```

所以前一个 `BeanPostProcessor` 如果返回了包装对象，后一个 `BeanPostProcessor` 拿到的就是这个包装后的对象。

### AbstractAutoProxyCreator 和 BeanPostProcessor 是什么关系

`AbstractAutoProxyCreator` 是一种特殊的 `BeanPostProcessor`。

关系可以先这样看：

```text
AbstractAutoProxyCreator
  implements SmartInstantiationAwareBeanPostProcessor
    extends InstantiationAwareBeanPostProcessor
      extends BeanPostProcessor
```

所以它既能参与普通初始化后代理：

```java
postProcessAfterInitialization(...)
```

也能参与循环依赖里的早期引用：

```java
getEarlyBeanReference(...)
```

也就是说，`AbstractAutoProxyCreator` 是 Spring AOP / 事务代理用 `BeanPostProcessor` 扩展点包装 Bean 的典型实现。

### 早期代理之后，initializeBean 还会不会继续调用

会继续调用。

回到 A 之后，A 的 `initializeBean(...)` 仍然会执行，并且里面仍然会调用所有 `BeanPostProcessor#postProcessAfterInitialization(...)`。

原因是这是 Bean 初始化生命周期的固定流程，不会因为前面 06-09 提前创建过代理就跳过。

流程是：

```text
B 依赖 A
  -> 提前 getEarlyBeanReference(rawA)
  -> 如果 A 需要代理，返回 proxyA 给 B

回到 A
  -> initializeBean(rawA) 仍然执行
  -> 所有 BeanPostProcessor 仍然执行
  -> AbstractAutoProxyCreator.postProcessAfterInitialization(rawA)
     发现 earlyProxyReferences 里已经记录过 rawA
     -> 不再创建第二个 AOP 代理
```

所以结论是：

```text
initializeBean 会继续走；
postProcessAfterInitialization 也会继续被调用；
但 AbstractAutoProxyCreator 不会重复给 A 包一层 AOP 代理。
```

原因是：在循环依赖场景下，B 回头找 A 时会触发 `getEarlyBeanReference(rawA, "a")`。`AbstractAutoProxyCreator` 在这里会先把 `rawA` 记录到 `earlyProxyReferences`，然后再执行 `wrapIfNecessary(...)` 判断是否要提前创建 `proxyA`。等 A 后面走到 `postProcessAfterInitialization(rawA, "a")` 时，它会发现 `earlyProxyReferences` 里已经记录过同一个 `rawA`，于是不会再次对这个 `rawA` 调用 `wrapIfNecessary(...)` 创建第二个 AOP 代理。

### earlyProxyReferences 存的是什么

`earlyProxyReferences` 是 `AbstractAutoProxyCreator` 自己的字段，不是三级缓存。

它也不是按所有 `BeanPostProcessor` 分开存的全局结构。更准确是：

```text
每个 AbstractAutoProxyCreator 实例自己维护一张 earlyProxyReferences 表。
这张表按 bean 的 cacheKey 存。
value 是这个 bean 的原始对象引用。
```

源码字段是：

```java
private final Map<Object, Object> earlyProxyReferences = new ConcurrentHashMap<>(16);
```

早期代理时：

```java
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyProxyReferences.put(cacheKey, bean);
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

假设是 A：

```text
key   = "a"
value = rawA
```

注意 value 是 `rawA`，不是 `proxyA`。

它表达的是：

```text
A 这个原始对象 rawA 已经走过 early proxy 判断了。
后面 initializeBean 结束后，不要再让 AbstractAutoProxyCreator 对 rawA 重复 wrapIfNecessary。
```

所以 `earlyProxyReferences` 可以先记成一句话：

```text
earlyProxyReferences 是 AbstractAutoProxyCreator 用来防止“同一个 rawA 在早期代理和初始化后代理阶段被重复 AOP 包装”的标记表。
它不是三级缓存，也不存 proxyA。
```

### 为什么 AbstractAutoProxyCreator 不会重复 wrapIfNecessary

后面 `postProcessAfterInitialization(...)` 里会这样判断：

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

意思是：

```text
如果 earlyProxyReferences 里拿出来的对象就是当前 rawA，
说明前面 getEarlyBeanReference(rawA) 已经处理过它了，
这里就不再 wrapIfNecessary。
```

注意，这里说的是 `AbstractAutoProxyCreator` 自己不再重复创建 AOP 代理，不是跳过 `initializeBean()`，也不是跳过所有 `BeanPostProcessor`。

`doCreateBean(...)` 后面还有一步：

```java
Object earlySingletonReference = getSingleton(beanName, false);
if (earlySingletonReference != null) {
    if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
    }
}
```

这一步会把最终暴露对象从 `rawA` 替换成之前给 B 注入过的 `earlyA`。

如果 A 需要代理，这个 `earlyA` 就是 `proxyA`。

### 多个增强不等于多层代理

这里最容易混的是：

```text
多个增强
≠
多层代理
```

日常最常见的场景是：

```text
一个 Bean 同时有 @Transactional
又命中了日志 @Aspect
又命中了权限 @Aspect
```

这通常不是：

```text
日志代理(事务代理(rawA))
```

而是：

```text
proxyA
  advisors:
    - TransactionInterceptor
    - LogAspect interceptor
    - AuthAspect interceptor
  target:
    rawA
```

也就是一个代理对象里有多个 `Advisor` / `Interceptor`。

源码也能对应上。`AbstractAdvisorAutoProxyCreator` 会找出当前 Bean 命中的所有 `Advisor`：

```java
List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
return advisors.toArray();
```

然后 `AbstractAutoProxyCreator#createProxy(...)` 里一次性加入：

```java
Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
proxyFactory.addAdvisors(advisors);
proxyFactory.setTargetSource(targetSource);
```

所以正常 Spring AOP 主线里：

```text
@Transactional + 日志切面
通常是一个代理 + 多个 Advisor，
不是事务代理外面再包日志代理。
```

### 什么情况下才会真的代理包代理

“代理包代理”是更特殊的情况，不是最常见主线。

比如另一个 `BeanPostProcessor` 没有接入 Spring Advisor 体系，而是在 `postProcessAfterInitialization(...)` 里自己直接返回一个新代理对象：

```java
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if ("a".equals(beanName)) {
        ProxyFactory factory = new ProxyFactory(bean);
        factory.addAdvice((MethodInterceptor) invocation -> invocation.proceed());
        return factory.getProxy();
    }
    return bean;
}
```

这不是三级缓存。

`ProxyFactory` 是一个底层代理创建工具：

```text
ProxyFactory
  -> DefaultAopProxyFactory
  -> JDK / CGLIB
  -> 生成代理对象
```

三级缓存是 BeanFactory 创建单例 Bean 时处理循环依赖的机制：

```text
singletonFactories
earlySingletonObjects
singletonObjects
```

所以：

```text
ProxyFactory 本身不走三级缓存。
```

只有这种情况才和三级缓存连上：

```text
B 创建时需要 A
  -> getSingleton("a")
  -> 从 singletonFactories 拿 ObjectFactory
  -> 执行 getEarlyBeanReference(rawA)
  -> 某个 SmartInstantiationAwareBeanPostProcessor 在这里用 ProxyFactory 创建 proxyA
```

也就是说：

```text
三级缓存触发 getEarlyBeanReference；
getEarlyBeanReference 里面可能用 ProxyFactory 创建代理。
```

如果没有循环依赖，另一个 `BeanPostProcessor` 又包一层，可能是：

```text
rawA
  -> AbstractAutoProxyCreator 包成 proxyA
  -> 自定义 BeanPostProcessor 再包成 myProxyA
  -> singletonObjects 里最终存 myProxyA
```

业务调用链就可能变成：

```text
myProxyA.method()
  -> 自定义包装逻辑
  -> proxyA.method()
    -> Spring AOP / 事务逻辑
    -> rawA.method()
```

但在循环依赖时会麻烦：

```text
B.a 已经提前注入 proxyA
A 后面 initializeBean 又被另一个 BeanPostProcessor 包成 myProxyA
```

这时容器最终想暴露 `myProxyA`，但 B 里面已经拿到了 `proxyA`。Spring 会检查这个不一致：

```java
Object earlySingletonReference = getSingleton(beanName, false);
if (earlySingletonReference != null) {
    if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
    }
    else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        throw new BeanCurrentlyInCreationException(...);
    }
}
```

所以标准主线里，`AbstractAutoProxyCreator` 用 `earlyProxyReferences` 是为了避免自己重复包；但如果另一个 `BeanPostProcessor` 后面又包一层，就可能造成“早期注入给 B 的对象”和“容器最终暴露的对象”不一致，循环依赖场景下默认可能直接报错。

最后压缩成一句话：

```text
多个增强 ≠ 多层代理。

@Transactional + 日志切面：
  通常是一个代理里多个 Advisor。

多层代理：
  通常是多个代理体系叠加，或者自定义 BeanPostProcessor 直接返回新代理对象。
```

## 先把核心问题说透：为什么不一开始就创建代理

这一节先回答一个最容易卡住的问题：

```text
既然 A 有 @Transactional，Spring 不是很早就能知道 A 最终需要代理吗？
那为什么不在 A 刚实例化后就直接创建 proxyA，然后提前暴露 proxyA？
```

这个问题不能只用“Spring 就是这样设计的”来回答。

结合上面的全局导图，真正要分清的是：

```text
Spring 能不能提前判断 A 可能需要代理
≠
Spring 是否应该现在就生成 earlyA
≠
这个 earlyA 是否真的已经被别的 Bean 提前拿走
```

先设一个具体例子：

```java
@Service
public class AService {
    @Autowired
    private BService bService;

    @Transactional
    public void pay() {
    }
}

@Service
public class BService {
    @Autowired
    private AService aService;
}
```

这里有两件事同时发生：

```text
AService 和 BService 形成属性循环依赖。
AService 有 @Transactional，所以最终对外应该暴露 proxyA，而不是 rawA。
```

### 对应导图：Spring 当前真正做了什么

看下面全局导图里的几个点：

```text
[00.1] createBeanInstance("a")
  得到 rawA

[01]-[02] addSingletonFactory("a", ObjectFactory)
  只把 ObjectFactory 放进 singletonFactories
  不执行 getEarlyBeanReference
  不创建 proxyA

[03] B 回头 getBean("a")
  这才表示真的有人需要 A 的早期引用

[04]-[09] singletonFactory.getObject()
  这时才执行 getEarlyBeanReference
  这时才通过 wrapIfNecessary 判断 earlyA 是 rawA 还是 proxyA

导图里“getSingleton("a") 收到 earlyA”这一段
  earlyA 被放入 earlySingletonObjects
  B.a = earlyA
```

对应源码在 `AbstractAutowireCapableBeanFactory#doCreateBean()`：

```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

这行对应导图 `[01]-[02]`。

它的关键点不是“创建 earlyA”，而是：

```text
登记一个以后可能生成 earlyA 的 ObjectFactory。
```

然后 `doCreateBean()` 继续走：

```java
populateBean(beanName, mbd, instanceWrapper);
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

这对应导图里 A 继续属性填充和初始化。

真正判断早期引用的地方在 `AbstractAutoProxyCreator#getEarlyBeanReference()`：

```java
this.earlyProxyReferences.put(cacheKey, bean);
return wrapIfNecessary(bean, beanName, cacheKey);
```

这对应导图 `[06]-[07]`。

普通 AOP / 事务代理的常规入口在 `AbstractAutoProxyCreator#postProcessAfterInitialization()`：

```java
if (this.earlyProxyReferences.remove(cacheKey) != bean) {
    return wrapIfNecessary(bean, beanName, cacheKey);
}
return bean;
```

这对应导图里 `initializeBean("a")` 下的 `postProcessAfterInitialization(...)`。

这两段源码放在一起看，才能看清本质。

### 为什么不是在 [01] 就提前创建 proxyA

如果在 `[01] addSingletonFactory(...)` 的位置就提前调用：

```java
getEarlyBeanReference(beanName, mbd, rawA)
```

那会提前进入：

```text
AbstractAutoProxyCreator#getEarlyBeanReference
  -> earlyProxyReferences.put(cacheKey, rawA)
  -> wrapIfNecessary(rawA)
```

如果 A 有事务，确实可以提前得到：

```text
proxyA
```

所以你的问题是合理的：

```text
既然最终都要 proxyA，早创建晚创建不都一样吗？
```

关键在于：**Spring 还必须知道这个 proxyA 有没有真的被别人提前拿走过。**

也就是说，Spring 要区分两件事：

```text
A 可以被提前暴露
A 已经真的被提前暴露
```

如果 `[01]` 就创建 proxyA，会出现两种方案。

### 方案一：提前调用 getEarlyBeanReference，但不把结果固定成 earlyA

假设你在 `[01]` 提前调用了：

```text
getEarlyBeanReference(rawA)
```

它会执行：

```text
earlyProxyReferences.put(cacheKey, rawA)
```

这个标记在 `AbstractAutoProxyCreator` 里的含义是：

```text
这个 rawA 已经走过早期代理路径。
后面 postProcessAfterInitialization 不要再重复创建 AOP 代理。
```

但是如果后面并没有发生循环依赖：

```text
没有 B 回头 getBean("a")
没有人真正拿走 earlyA
earlySingletonObjects 里也没有 A
```

那么 A 继续走正常初始化：

```text
initializeBean("a")
  -> postProcessAfterInitialization(rawA, "a")
```

这时 `AbstractAutoProxyCreator` 看到：

```text
earlyProxyReferences.remove(cacheKey) == rawA
```

于是它会认为：

```text
A 已经走过早期代理路径，不要再 wrapIfNecessary。
```

结果就是：

```text
postProcessAfterInitialization 返回 rawA
```

后面 `doCreateBean()` 又会执行：

```java
Object earlySingletonReference = getSingleton(beanName, false);
```

这个调用对应导图里：

```text
getSingleton("a", false)
  作用：检查 A 是否已经因为循环依赖被提前拿过早期引用
```

注意这里 `allowEarlyReference=false`，它不会再触发三级缓存里的 `ObjectFactory`。

如果没有人真的提前拿过 A：

```text
earlySingletonObjects 里没有 A
getSingleton("a", false) 返回 null
```

最后 A 可能就以 rawA 暴露出去。

所以，**提前调用 getEarlyBeanReference 但不固定结果，会破坏 `earlyProxyReferences` 的语义**。

它把“提前判断过”误当成了：

```text
早期代理真的已经被别人拿走过。
```

### 方案二：提前创建 proxyA，并把它固定成 earlyA

另一种做法是：

```text
[01] 创建 proxyA
[01] 直接把 proxyA 放入 earlySingletonObjects
```

这样即使后面没有 B 回头拿 A，`getSingleton("a", false)` 也能拿到 proxyA，最终暴露 proxyA。

这个方案看起来能解决问题，但它等于改变了生命周期：

```text
普通情况：
  A 还没有被任何 Bean 提前依赖
  也会在 createBeanInstance 之后、initializeBean 之前创建代理

Spring 当前设计：
  没有人提前依赖 A
  就不执行 getEarlyBeanReference
  AOP/事务代理仍然在 postProcessAfterInitialization 里创建
```

这不是“绝对不能做”，而是它把一个特殊路径变成了常规路径：

```text
特殊路径：循环依赖真的发生，A 被提前拿走，才生成 earlyA。
常规路径：没有循环依赖，A 初始化后再代理。
```

如果想既提前创建 proxyA，又不改变普通生命周期，那你还要继续记录：

```text
proxyA 已经生成，但还没有被别人拿走。
proxyA 已经生成，并且已经被别人拿走。
```

这又回到了状态拆分的问题。

### 三级缓存真正拆开的三个状态

所以三级缓存不是在回答：

```text
能不能提前知道 A 要不要代理？
```

它真正回答的是：

```text
A 处在什么暴露状态？
```

结合导图和源码，它拆成三种状态：

```text
singletonFactories
  对应导图 [01]-[02]
  状态：A 可以被提前暴露，但 earlyA 还没有生成。
  源码：addSingletonFactory(beanName, () -> getEarlyBeanReference(...))

earlySingletonObjects
  对应导图 [03]-[04] 执行 ObjectFactory 之后
  状态：有人真的因为循环依赖拿过 A，earlyA 已经生成并固定。
  earlyA 可能是 rawA，也可能是 proxyA。

singletonObjects
  对应导图 A 完整创建结束后
  状态：A 已经完整创建完成，是最终单例对象。
```

这就是为什么第三级缓存存的是 `ObjectFactory`。

它不是为了多一层 Map，而是为了表达：

```text
现在只是“可以提前暴露”，还不是“已经提前暴露”。
```

### rawA 和 proxyA 不是谁消灭谁

还有一个容易混的点：

```text
如果 A 需要代理，那是不是 rawA 就没用了？
```

不是。

如果 A 需要事务/AOP，正常对外依赖引用应该是：

```text
proxyA
```

但是 rawA 仍然存在。

它通常作为代理内部的 target：

```text
B.a = proxyA
B 调用 proxyA.pay()
  -> 进入事务 / AOP 拦截器
  -> 最后调用 rawA.pay()
```

所以：

```text
proxyA 是对外暴露的 Bean 引用。
rawA 是代理内部最终调用的目标对象。
```

这也是后面 `[10]-[13]` 要讲的内容：

```text
[10]/[11] 代理对象被调用
[12] 找 MethodInterceptor 链
[13] 最后调用 rawA 的目标方法
```

### 最后把问题压缩成一句话

这篇文档里讨论的三级缓存，不是为了解决：

```text
Spring 能不能提前判断 @Transactional。
```

而是为了解决：

```text
只有当 A 的早期引用真的被别人拿走时，
才生成并固定 earlyA；
否则继续走正常 initializeBean -> postProcessAfterInitialization 的代理流程。
```

换成导图语言就是：

```text
[01]-[02] 只登记 ObjectFactory，表示 A 可以被提前暴露。
[03] 真的有人回头找 A。
[04]-[09] 这时才生成 earlyA，决定是 rawA 还是 proxyA。
“A 拿到 B 后继续完成”这一段里，A 后面继续初始化，并检查 earlyA 是否真的被拿走过。
```

这才是三级缓存和 AOP 代理真正连起来的地方。

## 正文

### 00 AbstractAutowireCapableBeanFactory：A 先被实例化，但还没完整创建

这段发生在 `doCreateBean()`。

```java
// doCreateBean() 中，先准备 BeanWrapper，用来包住刚创建出来的原始 Java 对象。
BeanWrapper instanceWrapper = null;
if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
}
if (instanceWrapper == null) {
    // 这里完成实例化，只是把 raw bean 创建出来，还没有开始属性填充和初始化。
    instanceWrapper = createBeanInstance(beanName, mbd, args);
}
// bean 就是后面要提前暴露的原始对象 rawA。
Object bean = instanceWrapper.getWrappedInstance();
```

这里的 `bean` 是原始对象。

此时它只是被实例化了：

```text
已经 new 出来
还没有 populateBean()
还没有 initializeBean()
还没有放入 singletonObjects 一级缓存
```

所以它不是完整 Bean，只能叫 `raw bean` 或早期对象。

### 01 AbstractAutowireCapableBeanFactory：addSingletonFactory 只是登记早期引用工厂
addSingletonFactory 在3级缓存中什么位置可以查看[[循环依赖#三级缓存分别负责什么]]
```java
// 只有单例、允许循环依赖、并且当前 Bean 正在创建中，才需要提前暴露。
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    /**
     * ！！！！！！！！！！！！！
     * addSingletonFactory 后面 02 具体展开：它只把 ObjectFactory 放入三级缓存。
     * singletonFactory.getObject 后面 03 具体展开：别人回头找 A 时才会执行 lambda。
     * getEarlyBeanReference 后面 04 具体展开：lambda 执行后才决定返回 rawA 还是 proxyA。
     * ！！！！！！！！！！！！！
     */
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```

这里很容易误解。

```java
() -> getEarlyBeanReference(beanName, mbd, bean)
```

整体是一个 `ObjectFactory` 的实现对象，不是 `getEarlyBeanReference(...)` 的返回值。

它等价于：

```java
new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
        /**
         * ！！！！！！！！！！！！！
         * getEarlyBeanReference 后面 04 具体展开。
         * 这里不是 ObjectFactory 创建时执行，而是 singletonFactory.getObject() 被调用时执行。
         * ！！！！！！！！！！！！！
         */
        return getEarlyBeanReference(beanName, mbd, bean);
    }
}
```

所以这一行不会执行：

```java
getEarlyBeanReference(beanName, mbd, bean)
```

它只是把“以后可能执行的方法”包装成一个工厂对象传进去。

> [!note] 这里先记一句
> `addSingletonFactory(...)` 是登记工厂。
> `singletonFactory.getObject()` 才是执行工厂。

### 02 DefaultSingletonBeanRegistry：ObjectFactory 被放入三级缓存

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            // 三级缓存：这里存的是 ObjectFactory，不是 raw bean，也不是 proxy bean。
            this.singletonFactories.put(beanName, singletonFactory);
            // 防止同一个 bean 同时在二级缓存里留下旧的早期引用。
            this.earlySingletonObjects.remove(beanName);
            // 这里只记录 singleton beanName，不属于三级缓存本身。
            this.registeredSingletons.add(beanName);
        }
    }
}
```

这里对应三级缓存中的第三级：

```text
singletonFactories
  key = beanName
  value = ObjectFactory
```

它存的是工厂，不是对象。

`registeredSingletons` 不是三级缓存的一层。它只是登记 singleton beanName，用于单例注册表管理，比如获取单例名称、销毁单例时按注册顺序处理等。

严格的三级缓存只有：[[循环依赖#三级缓存分别负责什么]]

```text
singletonObjects
earlySingletonObjects
singletonFactories
```

### 03 DefaultSingletonBeanRegistry：什么时候才执行 ObjectFactory

当 B 创建过程中又需要 A，会重新触发：

```java
getBean("a")
```

它会先尝试从单例缓存拿 A：

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 一级缓存：完整创建好的单例 Bean。
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 二级缓存：已经被提前取出来用过的早期引用。
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        // 三级缓存：保存 ObjectFactory，真正需要早期引用时才取出来执行。
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            /**
                             * ！！！！！！！！！！！！！
                             * singletonFactory.getObject 会执行 01 中保存的 lambda。
                             * lambda 内部的 getEarlyBeanReference 后面 04 具体展开。
                             * 如果 AOP 介入，会继续走 06、07、08、09。
                             * ！！！！！！！！！！！！！
                             */
                            singletonObject = singletonFactory.getObject();
                            // ObjectFactory 执行后，早期引用进入二级缓存。
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            // 同一个 ObjectFactory 通常只执行一次，执行后从三级缓存移除。
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

执行点在这里：

```java
singletonObject = singletonFactory.getObject();
```

这时才会触发前面保存的 lambda：

```java
() -> getEarlyBeanReference(beanName, mbd, bean)
```

也就是这时才进入：

```java
getEarlyBeanReference(beanName, mbd, bean)
```

执行完成后：

```java
this.earlySingletonObjects.put(beanName, singletonObject);
this.singletonFactories.remove(beanName);
```

意思是：

```text
第一次从三级缓存拿到早期引用后，
把这个早期引用放到二级缓存，
并移除三级缓存里的 ObjectFactory。
```

所以 `ObjectFactory` 通常只执行一次。

### 04 AbstractAutowireCapableBeanFactory：getEarlyBeanReference 决定早期引用是什么

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    // 默认准备提前暴露原始对象 rawA。
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        // 遍历支持“早期引用处理”的 BeanPostProcessor。
        for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
            /**
             * ！！！！！！！！！！！！！
             * SmartInstantiationAwareBeanPostProcessor 默认实现后面 05 具体展开。
             * AbstractAutoProxyCreator 的 AOP 实现后面 06 具体展开。
             * ！！！！！！！！！！！！！
             */
            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        }
    }
    return exposedObject;
}
```

先看变量：

```text
bean
  原始对象 rawA

exposedObject
  准备提前暴露给其他 Bean 的对象
```

一开始：

```java
Object exposedObject = bean;
```

说明默认暴露原始对象。

后面遍历：

```java
SmartInstantiationAwareBeanPostProcessor
```

是为了给后处理器一个机会：

```text
如果这个 Bean 最终应该被包装，
现在就提前包装，
避免其他 Bean 拿到错误的原始对象。
```

为什么代码是：

```java
exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
```

因为这是处理链。前一个后处理器返回什么，下一个后处理器就基于什么继续处理。

### 05 SmartInstantiationAwareBeanPostProcessor：早期引用扩展点

`SmartInstantiationAwareBeanPostProcessor` 在这里不是一个具体处理器，而是一类扩展点。

它给后处理器一个机会：

```text
在 Bean 还没有完整初始化之前，
先决定要暴露给其他 Bean 的早期引用是什么。
```

默认接口实现是：

```java
default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    // 默认实现不改对象，直接返回传入的 bean。
    return bean;
}
```

所以默认情况下：

```text
输入 rawA
输出 rawA
```

这说明两件事：

```text
1. getEarlyBeanReference 不等于一定创建代理。
2. 真正会不会改写早期引用，要看具体实现类。
```

真正和 AOP 代理有关的是它的实现类，比如：

```text
AbstractAutoProxyCreator
AnnotationAwareAspectJAutoProxyCreator
```

其中 `AnnotationAwareAspectJAutoProxyCreator` 可以理解为更具体的 AOP 自动代理创建器，它继承了 `AbstractAutoProxyCreator` 这条能力。

### 06 AbstractAutoProxyCreator：AOP 场景提前创建代理

`AbstractAutoProxyCreator` 会重写：

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    // 标记这个 bean 已经走过早期代理路径，避免初始化后重复包装代理。
    this.earlyProxyReferences.put(cacheKey, bean);
    /**
     * ！！！！！！！！！！！！！
     * wrapIfNecessary 后面 07 具体展开。
     * 这里会判断当前 Bean 是否命中 AOP 增强，命中才会创建代理。
     * ！！！！！！！！！！！！！
     */
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

这里做两件事。

第一，记录这个 Bean 已经走过早期代理路径：

```java
this.earlyProxyReferences.put(cacheKey, bean);
```

这是为了避免后面 `postProcessAfterInitialization()` 再重复创建一个代理。

这里要说准确：

```text
postProcessAfterInitialization() 回调本身仍然会执行。
initializeBean() 也仍然会执行。
其他 BeanPostProcessor 也可能继续处理这个 Bean。
```

`earlyProxyReferences` 只是在 `AbstractAutoProxyCreator` 自己这里做标记：

```text
如果当前 rawA 已经通过 getEarlyBeanReference() 走过早期代理路径，
那么后面 AbstractAutoProxyCreator.postProcessAfterInitialization() 再执行时，
就不要对同一个 rawA 再次 wrapIfNecessary() 创建 AOP 代理。
```

所以它防的是：

```text
rawA -> proxyA
rawA -> proxyA2
```

不是防止初始化流程执行，也不是防止所有后处理器执行。

第二，调用：

```java
wrapIfNecessary(bean, beanName, cacheKey)
```

判断当前 Bean 是否需要代理。

> [!note] 这里判断的是 A，不是 B
> B 创建时回头找 A，触发的是 A 的 `ObjectFactory`，所以 `getEarlyBeanReference(rawA)` 判断的是 A 需不需要代理。
>
> B 自己需不需要代理，要看 B 自己后面的创建流程。没有循环依赖提前暴露时，B 的 AOP/事务代理通常会在 B 的 `postProcessAfterInitialization()` 里创建。

> [!note] 普通 AOP 和循环依赖 AOP 的差别
> 如果没有循环依赖，`getEarlyBeanReference()` 通常不会被触发。
> 普通 AOP 代理更多是在 `postProcessAfterInitialization()` 里创建。
> 这里讨论的是：循环依赖已经迫使 Spring 提前暴露 A，此时 AOP 必须提前介入，避免其他 Bean 注入到原始 A。

### 07 AbstractAutoProxyCreator：wrapIfNecessary 判断是否需要代理

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 已经通过自定义 TargetSource 处理过的 Bean，不再重复包装。
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    // 已经判断过不需要代理的 Bean，直接返回原始对象。
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    // Spring AOP 基础设施类本身不应该再被 AOP 代理。
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 查找当前 Bean 是否命中 Advisor/Advice；没有命中就不需要代理。
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        /**
         * ！！！！！！！！！！！！！
         * createProxy 后面 08 具体展开。
         * ProxyFactory / DefaultAopProxyFactory 后面 09 具体展开。
         * 代理对象后续被调用时，会进入 10 或 11。
         * ！！！！！！！！！！！！！
         */
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

核心判断是：

```java
Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
```

如果没有增强：

```text
返回 DO_NOT_PROXY
最后返回原始 bean
```

如果有增强：

```text
createProxy(...)
返回代理对象
```

这里第一次出现 `SingletonTargetSource(bean)`。

`SingletonTargetSource` 是 `TargetSource` 的一种简单实现，它固定持有一个目标对象。

在当前早期代理场景里：

```text
new SingletonTargetSource(bean)
  这里的 bean 就是 rawA
```

它的作用是：

```text
proxyA 对外暴露
SingletonTargetSource 内部持有 rawA
后面业务代码调用 proxyA.method() 时，再通过 TargetSource 找到 rawA
```

### 08 AbstractAutoProxyCreator：createProxy 只是创建代理，不执行方法

先把 07-09 的真实对象链画出来：

```text
rawA
  关系：createBeanInstance() 刚实例化出来的原始对象
  │
  └─ SingletonTargetSource(rawA)
      关系：TargetSource 的一种实现
      持有：target = rawA
      作用：后续代理方法调用时，通过 getTarget() 返回 rawA
      │
      └─ ProxyFactory
          关系：代理配置对象 + 代理创建入口，不是最终业务代理对象
          │
          ├─ 继承得到的配置能力
          │   ProxyFactory
          │     extends ProxyCreatorSupport
          │       extends AdvisedSupport
          │         extends ProxyConfig
          │
          │   因为 ProxyFactory 继承了 AdvisedSupport，所以同一个 ProxyFactory 对象里保存着：
          │     - targetSource = SingletonTargetSource(rawA)
          │     - advisors = 当前 Bean 命中的增强规则
          │     - interfaces = 需要实现的接口
          │     - proxyTargetClass / exposeProxy 等代理配置
          │     - advisorChainFactory = DefaultAdvisorChainFactory
          │
          └─ 通过父类 ProxyCreatorSupport 持有代理创建策略
              持有：aopProxyFactory = DefaultAopProxyFactory
              │
              └─ ProxyFactory.getProxy(classLoader)
                  │
                  └─ ProxyCreatorSupport.createAopProxy()
                      │
                      └─ DefaultAopProxyFactory.createAopProxy(this)
                          这里的 this 是当前 ProxyFactory。
                          因为 ProxyFactory 继承了 AdvisedSupport，
                          所以 this 会作为 AdvisedSupport 配置传给 DefaultAopProxyFactory。
                          │
                          ├─ 根据 config 选择 JDK
                          │   └─ new JdkDynamicAopProxy(config)
                          │       └─ getProxy(classLoader)
                          │           └─ 运行时生成 $Proxy... 对象，也就是 proxyA
                          │               └─ proxyA.method() 进入 10 JdkDynamicAopProxy::invoke(...)
                          │
                          └─ 根据 config 选择 CGLIB
                              └─ new ObjenesisCglibAopProxy(config)
                                  └─ getProxy(classLoader)
                                      └─ 运行时生成 Xxx$$EnhancerBySpringCGLIB$$... 对象，也就是 proxyA
                                          └─ proxyA.method() 进入 11 CglibAopProxy::intercept(...)
```

> [!note] 代理对象可能早于 initializeBean 出现，但增强逻辑不会在这里执行
> 在“循环依赖 + 当前 Bean 需要 AOP 代理”的场景下，`getEarlyBeanReference(...)` 可能会在 `initializeBean(...)` 之前创建出 `proxyA`，并把这个 `proxyA` 提前注入给别的 Bean。这里和依赖注入直接相关：如果别的 Bean 提前依赖 A，而 A 最终应该被 AOP 代理，那么注入进去的应该是 `proxyA`，不是 `rawA`。这里提前出现的只是代理对象本身；事务、切面等增强逻辑不会在 `createProxy(...)` 时执行，而是等后面业务代码真正调用 `proxyA.method()` 时，才进入 `JdkDynamicAopProxy::invoke(...)` 或 `CglibAopProxy::intercept(...)`。

所以这里不是 `ProxyFactory -> AdvisedSupport` 两个并列对象。

更准确地说，`ProxyFactory` 通过继承拥有 `AdvisedSupport` 的配置能力，又通过父类 `ProxyCreatorSupport` 持有 `DefaultAopProxyFactory`。最后 `ProxyFactory` 把自己作为 `AdvisedSupport config` 交给 `DefaultAopProxyFactory`，由它选择 JDK 还是 CGLIB。

所以 `createProxy(...)` 不是把 `rawA` 改没了。

它创建的是一个对外暴露的代理对象 `proxyA`，而 `rawA` 会作为目标对象被 `TargetSource` 持有。

这里说的 `proxyA` 不是源码里提前写好的 Java 类。

如果走 JDK 动态代理，JDK 会在运行时根据接口生成类似这样的类名：

```text
com.sun.proxy.$Proxy123
jdk.proxy2.$Proxy123
```

如果走 CGLIB，CGLIB 会在运行时生成目标类的子类，类名通常类似：

```text
AServiceImpl$$EnhancerBySpringCGLIB$$xxxx
```

`$` / `$$` 只是生成类名的一部分。Java 类名里允许出现 `$`，编译器和代理框架经常用它标识内部类、匿名类、运行时生成类。

所以这里没有一个固定的 `ProxyA.java` 可以贴。

能读到的真实源码是这条对象链：

```text
ProxyFactory.getProxy()
  -> ProxyCreatorSupport.createAopProxy()
    -> DefaultAopProxyFactory.createAopProxy(this)
       这里的 this 是 ProxyFactory，也是 AdvisedSupport 配置对象
      -> JdkDynamicAopProxy(config) 或 ObjenesisCglibAopProxy(config)
        -> getProxy(classLoader) 运行时生成 proxyA
```

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    // ProxyFactory 保存代理配置：TargetSource、Advisor、proxyTargetClass 等。
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    // 把当前 Bean 命中的增强包装为 Advisor，后续方法调用时再筛选成 MethodInterceptor 链。
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    // TargetSource 决定代理对象调用时从哪里拿原始目标对象。
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    ClassLoader classLoader = getProxyClassLoader();
    /**
     * ！！！！！！！！！！！！！
     * proxyFactory.getProxy 后面 09 具体展开。
     * JDK 代理方法调用入口后面 10 具体展开。
     * CGLIB 代理方法调用入口后面 11 具体展开。
     * ！！！！！！！！！！！！！
     */
    return proxyFactory.getProxy(classLoader);
}
```

这里不要误解。

`createProxy(...)` 做的是：

```text
准备代理配置
  -> 目标对象从哪里拿：TargetSource
  -> 有哪些增强：Advisors
  -> 用什么代理方式：JDK 或 CGLIB
  -> 返回代理对象
```

它不会立刻执行：

```text
Advice
MethodInterceptor
目标方法
```

这些都发生在代理对象后续被业务代码调用时。

换成后面 10-13 的运行时链路，就是：

```text
业务代码调用 proxyA.method()
  -> 10 JDK invoke 或 11 CGLIB intercept
    -> 从 TargetSource 拿 rawA
    -> 从 Advisors 筛出当前 method 的 MethodInterceptor 链
    -> 13 proceed()
      -> 执行增强
      -> 最后调用 rawA.method()
```

### 09 ProxyFactory 和 DefaultAopProxyFactory：选择 JDK 还是 CGLIB

先把边界放清楚：JDK 代理还是 CGLIB 代理，是 `DefaultAopProxyFactory` 根据当前 `ProxyFactory` 里的 `AdvisedSupport config` 决定的，不是整个项目只能选一种。

这份 `config` 里包含 `targetClass`、`interfaces`、`proxyTargetClass`、`optimize` 等信息。一个应用里可以同时出现 JDK 代理和 CGLIB 代理，因为选择是按每个被代理 Bean 分别判断的；但对同一个 Bean 的这一层 AOP 代理来说，最终通常只会选一种，不会同一个 `proxyA` 既是 `$Proxy...` 又是 `$$EnhancerBySpringCGLIB$$...`。

可以先这样记：

```text
有接口 + 没强制 proxyTargetClass：通常 JDK 动态代理
没有接口 / 强制 proxyTargetClass=true：通常 CGLIB 代理
```

`ProxyFactory` 最后会走：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    /**
     * ！！！！！！！！！！！！！
     * createAopProxy 在本节下面继续展开：它决定使用 JDK 代理还是 CGLIB 代理。
     * ！！！！！！！！！！！！！
     */
    return createAopProxy().getProxy(classLoader);
}
```

`createAopProxy()` 会委托给 `DefaultAopProxyFactory`：

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // 根据代理配置选择 JDK 动态代理或 CGLIB 代理。
    if (!NativeDetector.inNativeImage() &&
            (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)) {
            /**
             * ！！！！！！！！！！！！！
             * JDK 动态代理的调用入口后面 10 具体展开。
             * ！！！！！！！！！！！！！
             */
            return new JdkDynamicAopProxy(config);
        }
        /**
         * ！！！！！！！！！！！！！
         * CGLIB 代理的调用入口后面 11 具体展开。
         * ！！！！！！！！！！！！！
         */
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        /**
         * ！！！！！！！！！！！！！
         * JDK 动态代理的调用入口后面 10 具体展开。
         * ！！！！！！！！！！！！！
         */
        return new JdkDynamicAopProxy(config);
    }
}
```

大体规则是：

```text
有接口，并且没有强制 proxyTargetClass
  -> JDK 动态代理

强制 proxyTargetClass，或者没有合适接口
  -> CGLIB 代理
```

不管是哪种代理，都会持有同一份代理配置：

```text
TargetSource
Advisors
proxyTargetClass
exposeProxy
```

区别在于代理对象的方法调用入口不同。

### 09.1 回到 doCreateBean：用 earlyA 作为最终 exposedObject

08/09 讲的是早期引用路径里可能创建出 `proxyA`。

但是创建出 `proxyA` 还不等于 A 这个 Bean 的创建流程结束了。B 拿到 `earlyA` 之后，调用栈会回到 A 自己的 `doCreateBean(...)`，A 还要继续完成 `populateBean(...)`、`initializeBean(...)` 和最后的单例注册。

这时 `doCreateBean(...)` 收尾处会做一个检查：

```java
if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
        if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
        }
        else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            // 这里省略异常处理分支
        }
    }
}
```

这里对应全局导图里的：

```text
getSingleton("a", false)
  -> 如果拿到 earlyA，并且 exposedObject 还是 rawA
     -> exposedObject = earlyA
```

注意这里的 `allowEarlyReference=false`。

它不会再次触发三级缓存里的 `ObjectFactory`，也不会再次执行：

```java
getEarlyBeanReference(...)
```

它只是检查前面是否已经有人因为循环依赖提前拿过 A 的早期引用。

如果 B 前面确实回头找过 A，那么在 `03 getSingleton("a")` 里已经发生过：

```text
singletonFactory.getObject()
  -> 得到 earlyA
  -> earlySingletonObjects.put("a", earlyA)
  -> singletonFactories.remove("a")
```

所以这里的 `getSingleton("a", false)` 能从 `earlySingletonObjects` 里拿到同一个 `earlyA`。

如果 A 需要代理：

```text
earlyA = proxyA
B.a = proxyA
```

那么这里会把：

```text
exposedObject = rawA
```

替换成：

```text
exposedObject = proxyA
```

这样最终放入一级缓存、对外暴露的 A，和之前注入给 B 的 A 是同一个对象。

这一步解决的是一致性问题：

```text
B.a 已经拿到了 earlyA；
容器最终暴露出去的 A 也应该是同一个 earlyA。
```

否则就会出现：

```text
B.a 是 proxyA
applicationContext.getBean("a") 又是 rawA 或另一个对象
```

这就会破坏 Spring 单例 Bean 对外引用的一致性。

### 09.2 从请求到代理 invoke/intercept 的调用链

到 09.1 为止，Bean 创建流程里已经保证最终暴露的 A 和前面注入给 B 的早期 A 保持一致。

但是 AOP 增强逻辑仍然还没有执行。

如果 A 需要代理，此时得到的是：

```text
proxyA
  持有：TargetSource(rawA)
  持有：Advisors
  类型：JDK 代理或 CGLIB 代理
```

但这时还没有执行事务、日志切面、权限切面，也没有执行 `rawA.method()`。

这些都要等后面真的有业务代码调用代理对象的方法。

如果从一次 Web 请求看，链路大概是：

```text
HTTP 请求
  │
  └─ DispatcherServlet::doDispatch(request, response)
      作用：Spring MVC 请求总入口；查 handler、选 adapter、执行 controller、处理返回值
      │
      ├─ getHandler(request)
      │   结果：找到当前请求对应的 HandlerExecutionChain
      │
      ├─ getHandlerAdapter(handler)
      │   结果：通常拿到 RequestMappingHandlerAdapter
      │
      └─ HandlerAdapter::handle(request, response, handler)
          │
          └─ RequestMappingHandlerAdapter::handleInternal(...)
              │
              └─ RequestMappingHandlerAdapter::invokeHandlerMethod(...)
                  作用：为本次 Controller 方法调用准备参数解析器、返回值处理器等
                  │
                  └─ ServletInvocableHandlerMethod::invokeAndHandle(...)
                      │
                      └─ InvocableHandlerMethod::invokeForRequest(...)
                          │
                          ├─ getMethodArgumentValues(...)
                          │   作用：解析 Controller 方法参数，例如 @RequestParam、@PathVariable、@RequestBody
                          │
                          └─ doInvoke(args)
                              │
                              └─ method.invoke(getBean(), args)
                                  作用：反射调用 Controller 方法
                                  │
                                  └─ 进入你的 Controller 代码
                                      │
                                      └─ orderController.createOrder(...)
                                          │
                                          └─ orderService.createOrder()
                                              说明：orderService 字段是依赖注入进来的对象
                                              │
                                              ├─ 如果 orderService 没有被代理
                                              │   └─ 直接调用 rawService.createOrder()
                                              │
                                              └─ 如果 orderService 被 AOP/事务代理
                                                  └─ 实际调用的是 proxyA.createOrder()
                                                      │
                                                      ├─ JDK 动态代理
                                                      │   └─ $Proxy...::createOrder(...)
                                                      │       └─ InvocationHandler::invoke(proxy, method, args)
                                                      │           └─ [10] JdkDynamicAopProxy::invoke(...)
                                                      │
                                                      └─ CGLIB 代理
                                                          └─ Xxx$$EnhancerBySpringCGLIB::createOrder(...)
                                                              └─ [11] CglibAopProxy::intercept(...)
```

这里有一个很重要的边界：

```text
Spring MVC 负责把 HTTP 请求调到 Controller 方法。
JDK/CGLIB 代理负责在 Controller 调用被代理 Service 方法时接管这次方法调用。
```

也就是说，`DispatcherServlet` 不会直接调用 `JdkDynamicAopProxy::invoke(...)`。

真正触发 10 或 11 的瞬间，是你的业务代码里发生了这类调用：

```java
orderService.createOrder();
```

并且这个 `orderService` 字段里注入的正好是 `proxyA`。

所以 10-13 讲的不是 Bean 创建过程本身，也不是 Spring MVC 请求分发本身，而是 Bean 创建完成后，代理对象被业务方法调用时的运行时链路。

### 10 JdkDynamicAopProxy：JDK 代理被调用时才进入增强链

JDK 动态代理的调用入口是：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    // 从 TargetSource 拿到真正被调用的原始目标对象。
    target = targetSource.getTarget();
    Class<?> targetClass = (target != null ? target.getClass() : null);

    /**
     * ！！！！！！！！！！！！！
     * getInterceptorsAndDynamicInterceptionAdvice 后面 12 具体展开。
     * 这里会根据当前 method 从 Advisors 中筛出 MethodInterceptor 链。
     * ！！！！！！！！！！！！！
     */
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    if (chain.isEmpty()) {
        // 没有增强链时，直接反射调用目标方法。
        Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
        retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
    }
    else {
        MethodInvocation invocation =
                new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
        /**
         * ！！！！！！！！！！！！！
         * ReflectiveMethodInvocation.proceed 后面 13 具体展开。
         * 这里开始按顺序执行拦截器链，最后调用目标方法。
         * ！！！！！！！！！！！！！
         */
        retVal = invocation.proceed();
    }
}
```

这时才真正发生：

```text
从 TargetSource 拿目标对象 rawA
根据当前 method 找拦截器链
执行 ReflectiveMethodInvocation.proceed()
```

所以 AOP 代理对象不是替代了原始对象的业务逻辑。

它是：

```text
代理对象接住方法调用
  -> 执行增强链
    -> 最后再调用原始对象的方法
```

### 11 CglibAopProxy：CGLIB 代理也是同一套 proceed 链路

CGLIB 的入口名字不同，是 `intercept(...)`：

```java
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    // CGLIB 代理被调用时，同样先拿到原始目标对象。
    Object target = targetSource.getTarget();
    Class<?> targetClass = (target != null ? target.getClass() : null);
    /**
     * ！！！！！！！！！！！！！
     * getInterceptorsAndDynamicInterceptionAdvice 后面 12 具体展开。
     * CGLIB 路线和 JDK 路线一样，也要先筛选当前方法的拦截器链。
     * ！！！！！！！！！！！！！
     */
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    if (chain.isEmpty() && CglibMethodInvocation.isMethodProxyCompatible(method)) {
        // 没有增强链时，直接调用目标方法。
        Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
        retVal = invokeMethod(target, method, argsToUse, methodProxy);
    }
    else {
        /**
         * ！！！！！！！！！！！！！
         * CglibMethodInvocation 复用 ReflectiveMethodInvocation.proceed，后面 13 具体展开。
         * ！！！！！！！！！！！！！
         */
        retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
    }
}
```

它和 JDK 动态代理的主线一样：

```text
代理方法被调用
  -> 取 target
  -> 找 MethodInterceptor 链
  -> proceed()
  -> 最后调用目标方法
```

区别只是底层代理技术不同。

### 12 AdvisedSupport 和 DefaultAdvisorChainFactory：Advisor 变成 MethodInterceptor 链

代理对象持有的是 `Advisors`。

方法调用时，需要根据当前方法筛出真正要执行的拦截器链：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        /**
         * ！！！！！！！！！！！！！
         * DefaultAdvisorChainFactory 的筛选逻辑在本节下面继续展开。
         * 它会把 Advisors 转成当前 method 真正要执行的 MethodInterceptor 链。
         * ！！！！！！！！！！！！！
         */
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}
```

真正筛选发生在 `DefaultAdvisorChainFactory`：

```java
for (Advisor advisor : advisors) {
    if (advisor instanceof PointcutAdvisor) {
        PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
        // 先判断类级别是否匹配。
        if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
            MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
            // 再判断当前被调用的方法是否匹配。
            boolean match = mm.matches(method, actualClass);
            if (match) {
                // 把 Advisor 中的 Advice 适配成统一的 MethodInterceptor。
                MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }
    }
}
```

这里完成的是：

```text
Advisor
  -> 判断 class 是否匹配
  -> 判断 method 是否匹配
  -> 把 Advice 适配成 MethodInterceptor
  -> 得到当前方法真正要执行的拦截器链
```

所以不是所有 AOP 增强都会在每个方法上执行。

每次方法调用，都会围绕当前 `method` 找匹配的链。

> [!note] 这里只截取主线
> 上面的代码主要展示 `PointcutAdvisor` 这条常见路径。
> 实际源码还会处理 `IntroductionAdvisor` 和其他 Advisor 类型，但核心目的相同：把 Advisor 转成当前方法要执行的拦截器链。

### 13 ReflectiveMethodInvocation：proceed 怎么执行增强和目标方法

先把执行模型放清楚：`proceed()` 的本质不是“把所有拦截器依次执行完再调用目标方法”，而是每个 `MethodInterceptor` 通过 `invocation.proceed()` 把调用权交给下一个拦截器。

目标方法被最里面那层 `invokeJoinpoint()` 调用，返回后再一层层回到外层拦截器。

```text
Interceptor1 前置逻辑
  -> invocation.proceed()
     Interceptor2 前置逻辑
       -> invocation.proceed()
          Interceptor3 前置逻辑
            -> invocation.proceed()
               -> invokeJoinpoint()
                  -> rawA.method()
            <- Interceptor3 后置逻辑
       <- Interceptor2 后置逻辑
  <- Interceptor1 后置逻辑
```

所以它更像一层层包住目标方法：

```text
Interceptor1(
  Interceptor2(
    Interceptor3(
      rawA.method()
    )
  )
)
```

如果某个 `MethodInterceptor` 不调用 `invocation.proceed()`，后面的拦截器和 `rawA.method()` 甚至可以不执行。

```java
public Object proceed() throws Throwable {
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        /**
         * ！！！！！！！！！！！！！
         * invokeJoinpoint 在本节下面继续展开。
         * 所有拦截器都执行完后，才真正调用目标对象方法。
         * ！！！！！！！！！！！！！
         */
        return invokeJoinpoint();
    }

    // 每次 proceed() 都向后推进一个拦截器。
    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

    // 当前拦截器内部如果继续调用 mi.proceed()，就会回到本方法执行下一个拦截器。
    return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
}
```

`currentInterceptorIndex` 初始是 `-1`。

每调用一次 `proceed()`：

```text
推进到下一个 MethodInterceptor
执行当前 MethodInterceptor.invoke(this)
```

典型拦截器内部会做：

```text
前置逻辑
mi.proceed()
后置逻辑
```

当所有拦截器都执行完：

```java
protected Object invokeJoinpoint() throws Throwable {
    // 真正调用原始目标对象的方法：target.method(arguments)。
    return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
}
```

这里才真正调用：

```text
target.method(args)
```

也就是原始 Bean 的目标方法。

### 14 回到三级缓存：为什么第三级缓存要存 ObjectFactory

现在可以回头看这个问题：

```text
为什么 singletonFactories 不直接存 rawA？
```

因为 Spring 需要把决定权延迟到“真的有人需要早期引用”的时刻：

```text
如果没有循环依赖：
  A 创建完成后通过 addSingleton 进入 singletonObjects
  ObjectFactory 可能根本不会执行

如果有普通循环依赖：
  ObjectFactory 执行
  getEarlyBeanReference 返回 rawA
  B.a = rawA

如果有 AOP/事务代理循环依赖：
  ObjectFactory 执行
  getEarlyBeanReference 经过 AbstractAutoProxyCreator
  wrapIfNecessary 创建 proxyA
  B.a = proxyA
```

所以第三级缓存的价值不是“多放一层 Map”。

真正价值是：

```text
把早期引用的生成延迟到真正需要时，
并且给 BeanPostProcessor/AOP 一个机会，
决定这个早期引用应该是原始对象还是代理对象。
```
