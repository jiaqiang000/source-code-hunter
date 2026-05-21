## 前言

这篇文章分享一下 spring IoC 容器初始化第三部分的代码，也就是将前面解析出来的 BeanDefinition 对象 注册进 IoC 容器，其实就是存入一个 ConcurrentHashMap<String, BeanDefinition> 中。

（PS：可以结合我 GitHub 上对 Spring 框架源码的翻译注释一起看，会更有助于各位同学理解，地址：  
spring-beans https://github.com/AmyliaY/spring-beans-reading  
spring-context https://github.com/AmyliaY/spring-context-reading
）

## 先建立边界：注册 BeanDefinition，不是创建 Bean

> [!note] 本篇只追一个问题
> 前面已经看到 XML 中的 `<bean>` 会先被解析成 `BeanDefinitionHolder`，并且源码马上会把这个 holder 交给注册入口。
> 本篇重点不是重新解析 `<bean>`，而是展开 `registerBeanDefinition(...)` 之后，`BeanDefinition` 最终怎么落到容器内部的注册表。
> 这里仍然没有创建业务 Bean，只是在保存“Bean 应该怎么创建”的元数据。

把前三篇连起来看：

```text
第 1 篇：BeanDefinition 的资源定位过程
  XML 配置位置
    -> Resource

第 2 篇：将 bean 解析封装成 BeanDefinition
  Resource / Document / <bean>
    -> BeanDefinition
    -> BeanDefinitionHolder

第 3 篇：将 BeanDefinition 注册进 IoC 容器
  BeanDefinitionHolder
    -> BeanDefinitionRegistry
    -> DefaultListableBeanFactory.beanDefinitionMap
```

所以这里的“注册进 IoC 容器”，更准确地说是：

```text
把 beanName -> BeanDefinition 这组映射，保存到 ApplicationContext 内部持有的 DefaultListableBeanFactory 中。
```

## 先看对象关系：registry 到底是谁

这篇最容易卡住的地方，是 `getReaderContext().getRegistry()`。

它的返回类型是 `BeanDefinitionRegistry`，看起来像一个新的对象；但在本文这条 XML 主线里，它运行时实际指向的就是前面创建出来的 `DefaultListableBeanFactory`。

```text
AbstractRefreshableApplicationContext
  refreshBeanFactory()
    -> createBeanFactory()
    -> 得到 DefaultListableBeanFactory beanFactory

AbstractXmlApplicationContext
  loadBeanDefinitions(beanFactory)
    -> new XmlBeanDefinitionReader(beanFactory)

AbstractBeanDefinitionReader
  构造时接收 BeanDefinitionRegistry registry
    -> this.registry = registry
    -> 这里的 registry 就是 beanFactory

XmlReaderContext
  getRegistry()
    -> reader.getRegistry()
    -> 返回的还是 reader 里保存的 beanFactory

DefaultBeanDefinitionDocumentReader
  processBeanDefinition(...)
    -> getReaderContext().getRegistry()
    -> 拿到 DefaultListableBeanFactory
```

> [!tip] 为什么要用 BeanDefinitionRegistry 接口
> `DefaultListableBeanFactory` 很大，它既能注册 BeanDefinition，也能创建 Bean、按类型查 Bean、管理单例缓存。
> 在注册阶段，调用方只需要“注册 BeanDefinition”这个能力，所以源码把它暴露成 `BeanDefinitionRegistry` 接口。
> 这不是换了一个容器，只是用接口把当前阶段需要的能力收窄了。

## 全局导图：从 BeanDefinitionHolder 到 beanDefinitionMap

先不要把下面这些方法看成彼此独立的代码片段。源码里“解析 `<bean>` 得到 holder”和“把 holder 注册进去”是紧挨着发生的；第 3 篇不是另起一条新流程，而是把第 2 篇 `[14] processBeanDefinition(...)` 的后半段放大来看。

```text
运行时先记住 5 个核心对象：

[对象1] beanFactory = DefaultListableBeanFactory 实例
        它同时也是：
        DefaultListableBeanFactory
          extends AbstractAutowireCapableBeanFactory
          extends AbstractBeanFactory
          extends FactoryBeanRegistrySupport
          extends DefaultSingletonBeanRegistry
          extends SimpleAliasRegistry
          implements BeanDefinitionRegistry

        关系：
        后面 reader、readerContext、documentReader 拿到的 registry，运行时都是这个 beanFactory

        作用：
        真正保存 BeanDefinition

[对象2] reader = XmlBeanDefinitionReader 实例
        它持有：
        registry = beanFactory

        关系：
        reader 负责读取 XML，解析出来的 BeanDefinition 会通过 registry 注册回 beanFactory

[对象3] readerContext = XmlReaderContext 实例
        它持有：
        reader = XmlBeanDefinitionReader

        关系：
        readerContext.getRegistry()
          -> reader.getRegistry()
          -> beanFactory

[对象4] documentReader = DefaultBeanDefinitionDocumentReader 实例
        它持有：
        readerContext = XmlReaderContext

        关系：
        documentReader 处理 XML Document 中的默认标签，比如 <bean>
        需要注册时，通过 readerContext.getRegistry() 找到 beanFactory

[对象5] bdHolder = BeanDefinitionHolder 实例
        它持有：
          - BeanDefinition
          - beanName
          - aliases

        关系：
        bdHolder 不是最终存储结构
        BeanDefinitionReaderUtils 会把它拆开：
          - beanName + BeanDefinition -> beanFactory.beanDefinitionMap
          - aliases -> aliasMap


调用树：

[01] documentReader::processBeanDefinition(ele, delegate)
     方法所属：DefaultBeanDefinitionDocumentReader
     关系：对应第 2 篇 [14] processBeanDefinition(...)
     作用：拿到 BeanDefinitionHolder，并把它交给注册工具类
     │
     ├─ [01.1] delegate::parseBeanDefinitionElement(ele)
     │        方法所属：BeanDefinitionParserDelegate
     │        对应：第 2 篇 [15]-[20]
     │        作用：把 <bean> 解析成 BeanDefinitionHolder
     │
     ├─ [01.2] delegate::decorateBeanDefinitionIfRequired(ele, bdHolder)
     │        方法所属：BeanDefinitionParserDelegate
     │        对应：第 2 篇 [21]
     │        作用：如果有自定义属性或装饰标签，对 bdHolder 做补充包装
     │
     ├─ [02] readerContext::getRegistry()
     │        方法所属：XmlReaderContext
     │        关系：从 reader 里取出 registry
     │        运行时结果：DefaultListableBeanFactory
     │
     ├─ [03] BeanDefinitionReaderUtils::registerBeanDefinition(bdHolder, registry)
     │        方法所属：BeanDefinitionReaderUtils
     │        关系：工具方法，对应第 2 篇 [22]，这是第 3 篇真正展开的注册主线
     │        作用：拆出主 beanName、BeanDefinition、aliases
     │        │
     │        ├─ [04] registry::registerBeanDefinition(beanName, beanDefinition)
     │        │        接口方法：BeanDefinitionRegistry
     │        │        运行时实现：DefaultListableBeanFactory.registerBeanDefinition(...)
     │        │        │
     │        │        ├─ [05] validate beanDefinition
     │        │        │        作用：校验 BeanDefinition 本身是否合法
     │        │        │
     │        │        ├─ [06] check existingDefinition
     │        │        │        作用：检查同名 beanName 是否已经注册，决定是否允许覆盖
     │        │        │
     │        │        └─ [07] beanDefinitionMap / beanDefinitionNames
     │        │                 作用：真正保存 BeanDefinition，并记录注册顺序
     │        │
     │        └─ [08] registry::registerAlias(beanName, alias)
     │                 接口方法：AliasRegistry
     │                 运行时实现：SimpleAliasRegistry.registerAlias(...)
     │                 作用：如果 <bean> 有别名，把 alias -> beanName 保存起来
     │
     └─ [09] readerContext::fireComponentRegistered(...)
              方法所属：XmlReaderContext
              作用：发布“组件已注册”的解析事件，不是创建 Bean
```

## 正文

### 01-02 DefaultBeanDefinitionDocumentReader：processBeanDefinition 是注册入口

回过头看一下前面在 DefaultBeanDefinitionDocumentReader 中实现的 processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) 方法。

这个方法本身不要拆开看，它的完整语义是：

```text
解析 <bean>
  -> 得到 BeanDefinitionHolder
  -> 必要时装饰 BeanDefinitionHolder
  -> 取出 registry
  -> 交给 BeanDefinitionReaderUtils 注册
  -> 注册完成后发送事件
```

```java
/**
 * 将 .xml 文件中的元素解析成 BeanDefinition对象，并注册到 IoC容器 中
 */
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {

    // BeanDefinitionHolder 是对 BeanDefinition 的进一步封装，它持有一个 BeanDefinition 对象 及其对应
    // 的 beanName、aliases别名。
    // 对 Document 对象中 <Bean> 元素的解析由 BeanDefinitionParserDelegate 实现
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 对 bdHolder 进行包装处理
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            /**
             * ！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
             * 向 IoC 容器注册解析完成的 BeanDefinition对象，这是 BeanDefinition 向 IoC 容器注册的入口
             * ！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
             */
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // 在完成向 IOC容器 注册 BeanDefinition对象 之后，发送注册事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

这里的关键点是：

```text
getReaderContext().getRegistry()
  -> 表面类型是 BeanDefinitionRegistry
  -> 当前 XML 主线里，运行时对象是 DefaultListableBeanFactory
```

也就是说，`processBeanDefinition(...)` 没有自己保存 `BeanDefinition`，它只是把 `bdHolder` 和 `registry` 一起交给下一个工具方法。

### 03 BeanDefinitionReaderUtils：registerBeanDefinition 拆出主名和别名

接着看一下 BeanDefinitionReaderUtils 的 registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) 方法。

这个工具方法也不要拆碎看，它做的事情很集中：

```text
BeanDefinitionHolder
  -> beanName
  -> BeanDefinition
  -> aliases

主 beanName：
  -> registry.registerBeanDefinition(beanName, beanDefinition)

别名 aliases：
  -> registry.registerAlias(beanName, alias)
```

```java
/**
 * 将解析到的 BeanDefinition对象 注册到 IoC容器
 */
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // 获取解析的 <bean>元素 的名称 beanName
    String beanName = definitionHolder.getBeanName();
    /**
     * ！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
     * 开始向 IoC容器 注册 BeanDefinition对象
     * ！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
     */
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 如果解析的 <bean>元素 有别名alias，向容器中注册别名
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String aliase : aliases) {
            registry.registerAlias(beanName, aliase);
        }
    }
}
```

所以 `BeanDefinitionReaderUtils` 不是最终存储容器，它只是把 `BeanDefinitionHolder` 拆开，然后调用 `registry` 的注册能力。

### 04-07 DefaultListableBeanFactory：registerBeanDefinition 真正写入容器注册表

BeanDefinitionRegistry 中的 registerBeanDefinition(String beanName, BeanDefinition beanDefinition) 方法在 DefaultListableBeanFactory 实现类中的具体实现。

> [!warning] 版本说明
> 下面保留的是原文中的代码片段，它能说明核心注册思想。
> 如果对照 Spring 5.3.20 源码，具体并发处理和字段声明已经有一些变化，但主线不变：
> `beanDefinitionMap` 保存 `beanName -> BeanDefinition`，`beanDefinitionNames` 保存注册顺序。

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {

    /** 按注册顺序排列的 beanDefinition名称列表(即 beanName)  */
    private final List<String> beanDefinitionNames = new ArrayList<String>();

    /** IoC容器 的实际体现，key --> beanName，value --> BeanDefinition对象 */
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(64);

    /**
     * 向 IoC容器 注册解析的 beanName 和 BeanDefinition对象
     */
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {

        Assert.hasText(beanName, "Bean name must not be empty");
        Assert.notNull(beanDefinition, "BeanDefinition must not be null");

        // 校验解析的 BeanDefiniton对象
        if (beanDefinition instanceof AbstractBeanDefinition) {
            try {
                ((AbstractBeanDefinition) beanDefinition).validate();
            }
            catch (BeanDefinitionValidationException ex) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Validation of bean definition failed", ex);
            }
        }

        // 注册的过程中需要线程同步，以保证数据的一致性
        synchronized (this.beanDefinitionMap) {
            Object oldBeanDefinition = this.beanDefinitionMap.get(beanName);

            // 检查是否有同名(beanName)的 BeanDefinition 存在于 IoC容器 中，如果已经存在，且不允许覆盖
            // 已注册的 BeanDefinition，则抛出注册异常，allowBeanDefinitionOverriding 默认为 true
            if (oldBeanDefinition != null) {
                if (!this.allowBeanDefinitionOverriding) {
                    throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                            "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
                            "': There is already [" + oldBeanDefinition + "] bound.");
                }
                // 如果允许覆盖同名的 bean，后注册的会覆盖先注册的
                else {
                    if (this.logger.isInfoEnabled()) {
                        this.logger.info("Overriding bean definition for bean '" + beanName +
                                "': replacing [" + oldBeanDefinition + "] with [" + beanDefinition + "]");
                    }
                }
            }
            // 若该 beanName 在 IoC容器 中尚未注册，将其注册到 IoC容器中，
            else {
                // 将 beanName 注册到 beanDefinitionNames列表
                this.beanDefinitionNames.add(beanName);
                this.frozenBeanDefinitionNames = null;
            }
            // beanDefinitionMap 是 IoC容器 的最主要体现，他是一个 ConcurrentHashMap，
            // 直接存储了 bean的唯一标识 beanName，及其对应的 BeanDefinition对象
            this.beanDefinitionMap.put(beanName, beanDefinition);
        }
        // 重置所有已经注册过的 BeanDefinition 的缓存
        resetBeanDefinition(beanName);
    }
}
```

这一步才是真正的“注册落点”：

```text
beanDefinitionMap
  -> key 是 beanName
  -> value 是 BeanDefinition
  -> 后面 getBean() 创建对象时，会根据 beanName 找到对应 BeanDefinition

beanDefinitionNames
  -> 保存 BeanDefinition 的注册顺序
  -> 后面按类型查找、预实例化单例等流程会用到这个顺序
```

### 08 SimpleAliasRegistry：registerAlias 注册别名

`BeanDefinitionReaderUtils.registerBeanDefinition(...)` 里除了注册主 beanName，还会注册 aliases：

```java
String[] aliases = definitionHolder.getAliases();
if (aliases != null) {
    for (String aliase : aliases) {
        registry.registerAlias(beanName, aliase);
    }
}
```

这里的 `registry` 同时也是 `AliasRegistry`。在 `DefaultListableBeanFactory` 的继承链上，别名最终由 `SimpleAliasRegistry` 保存：

```text
DefaultListableBeanFactory
  -> AbstractAutowireCapableBeanFactory
  -> AbstractBeanFactory
  -> FactoryBeanRegistrySupport
  -> DefaultSingletonBeanRegistry
  -> SimpleAliasRegistry
```

别名注册的本质是：

```text
aliasMap
  -> key 是 alias
  -> value 是 beanName
```

所以主名和别名的落点不同：

```text
beanName -> BeanDefinition
  -> DefaultListableBeanFactory.beanDefinitionMap

alias -> beanName
  -> SimpleAliasRegistry.aliasMap
```

### 09 DefaultBeanDefinitionDocumentReader：注册完成后发送事件

`processBeanDefinition(...)` 最后还有一行：

```java
getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
```

这一步只是通知解析上下文“这个组件已经注册完成”，不是创建 Bean，也不是再次保存 `BeanDefinition`。

> [!note] 这一篇最终要记住什么
> `BeanDefinition` 不是注册进 `ApplicationContext` 自己的某个 Map。
> 它注册进 `ApplicationContext` 内部持有的 `DefaultListableBeanFactory`。
> 一路上看到的 `BeanDefinitionRegistry` 只是接口视角，当前 XML 主线里的运行时对象就是同一个 `DefaultListableBeanFactory`。
