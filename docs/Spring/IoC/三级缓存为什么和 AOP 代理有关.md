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
     │                 │              │        ├─ [05] SmartInstantiationAwareBeanPostProcessor::getEarlyBeanReference(...)
     │                 │              │        │  对应正文：05 SmartInstantiationAwareBeanPostProcessor：默认什么都不做
     │                 │              │        │  作用：默认返回原始对象 rawA
     │                 │              │        │
     │                 │              │        └─ [06] AbstractAutoProxyCreator::getEarlyBeanReference(rawA, "a")
     │                 │              │           对应正文：06 AbstractAutoProxyCreator：AOP 场景提前创建代理
     │                 │              │           作用：AOP 场景提前尝试生成代理
     │                 │              │           │
     │                 │              │           └─ [07] wrapIfNecessary(rawA, "a", cacheKey)
     │                 │              │              对应正文：07 AbstractAutoProxyCreator：wrapIfNecessary 判断是否需要代理
     │                 │              │              作用：判断 A 是否需要被代理
     │                 │              │              │
     │                 │              │              ├─ 不需要代理
     │                 │              │              │  │
     │                 │              │              │  └─ 返回 rawA
     │                 │              │              │     结果：B.a = rawA
     │                 │              │              │     注意：后面 initializeBean() 仍会执行，BeanPostProcessor 初始化后回调也仍会执行；
     │                 │              │              │           这里只是 AbstractAutoProxyCreator 不会再对 A 重复创建 AOP 代理
     │                 │              │              │
     │                 │              │              └─ 需要代理
     │                 │              │                 │
     │                 │              │                 └─ [08] createProxy(beanClass, beanName, advisors, SingletonTargetSource(rawA))
     │                 │              │                    对应正文：08 AbstractAutoProxyCreator：createProxy 只是创建代理，不执行方法
     │                 │              │                    作用：创建代理对象 proxyA
     │                 │              │                    注意：这里只创建代理，不执行目标方法
     │                 │              │                    │
     │                 │              │                    └─ [09] ProxyFactory / DefaultAopProxyFactory
     │                 │              │                       对应正文：09 ProxyFactory 和 DefaultAopProxyFactory：选择 JDK 还是 CGLIB
     │                 │              │                       作用：选择 JDK 动态代理或 CGLIB 代理，返回 proxyA
     │                 │              │                       结果：B.a = proxyA
     │                 │              │                       注意：后面 initializeBean() 仍会执行；
     │                 │              │                             这里只是 AbstractAutoProxyCreator 不会再对 A 重复创建另一个 AOP 代理
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
                 作用：依次执行拦截器，最后调用原始 rawA 的目标方法
                 │
                 └─ [14] 回到三级缓存职责
                    对应正文：14 回到三级缓存：为什么第三级缓存要存 ObjectFactory
                    作用：把 singletonFactories、earlySingletonObjects、singletonObjects 的职责重新合起来
```

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

### 05 SmartInstantiationAwareBeanPostProcessor：默认什么都不做

`SmartInstantiationAwareBeanPostProcessor` 的默认实现是：

```java
default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    // 默认实现不做包装，直接返回原始对象。
    // AOP 的关键是 AbstractAutoProxyCreator 会重写这个方法。
    return bean;
}
```

所以不是所有后处理器都会创建代理。

普通情况下，它可能什么都不做：

```text
输入 rawA
输出 rawA
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

这里的 `SingletonTargetSource(bean)` 很重要：

```text
代理对象对外暴露
TargetSource 内部持有原始 bean
后面代理方法被调用时，再通过 TargetSource 找到这个原始 bean
```

### 08 AbstractAutoProxyCreator：createProxy 只是创建代理，不执行方法

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

### 09 ProxyFactory 和 DefaultAopProxyFactory：选择 JDK 还是 CGLIB

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

## 这一篇最终要记住什么

```text
1. addSingletonFactory 只把 ObjectFactory 放进三级缓存，不执行 lambda。

2. B 回头找 A 时，getSingleton 才会执行 singletonFactory.getObject()。

3. ObjectFactory.getObject() 会进入 getEarlyBeanReference。

4. getEarlyBeanReference 会遍历 SmartInstantiationAwareBeanPostProcessor。

5. AbstractAutoProxyCreator 可以在这里提前创建 AOP 代理。

6. 如果 A 最终应该是代理对象，那么 B.a 也必须拿到代理对象，而不是原始对象。

7. createProxy 只创建代理对象，不执行目标方法。

8. 代理对象被调用时，才进入 JDK invoke 或 CGLIB intercept。

9. invoke/intercept 会筛选 MethodInterceptor 链。

10. ReflectiveMethodInvocation.proceed() 依次执行拦截器，最后调用原始目标方法。
```

一句话总结：

```text
三级缓存解决的不只是“提前拿到 A”，
更关键的是在 AOP/事务代理场景下，
提前拿到“正确的 A”。
```
