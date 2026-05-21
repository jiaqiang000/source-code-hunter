# Spring IoC 全局地图

> [!note] 阅读目标
> 这组文档不要一上来就按“某个类 / 某段源码”去读，而是先抓住一条主线：
> Spring 先把配置解析成 BeanDefinition，再把 BeanDefinition 注册进容器，之后在合适的时机创建 Bean，并在创建过程中完成依赖注入、初始化、后处理和循环依赖处理。

## 一句话主线

Spring IoC 的主线可以拆成两段：

1. 容器启动阶段：把配置变成 BeanDefinition，并注册到 BeanFactory。
2. Bean 创建阶段：根据 BeanDefinition 创建对象、注入依赖、执行初始化和后处理。

也就是说：

```text
配置文件 / 注解 / Java Config
        ↓
BeanDefinition
        ↓
BeanDefinitionRegistry / BeanFactory
        ↓
getBean() 或非懒加载单例预实例化
        ↓
createBean()
        ↓
实例化对象
        ↓
依赖注入
        ↓
初始化
        ↓
可用 Bean
```

## 全局流程地图

```text
ApplicationContext.refresh()
    ↓
1. 定位 BeanDefinition 资源
   [[1、BeanDefinition的资源定位过程]]

    ↓
2. 解析配置，封装成 BeanDefinition
   [[2、将bean解析封装成BeanDefinition]]

    ↓
3. 注册 BeanDefinition 到 IoC 容器
   [[3、将BeanDefinition注册进IoC容器]]

    ↓
4. BeanFactoryPostProcessor 修改 BeanDefinition
   [[BeanFactoryPostProcessor]]

    ↓
5. 注册 BeanPostProcessor
   [[BeanPostProcessor]]

    ↓
6. 触发 Bean 创建
   - getBean()
   - 非懒加载单例 preInstantiateSingletons()
   [[4、依赖注入(DI)]]

    ↓
7. Bean 创建主流程
   - createBeanInstance()
   - populateBean()
   - initializeBean()
   [[4、依赖注入(DI)]]

    ↓
8. 循环依赖处理
   - singletonObjects
   - earlySingletonObjects
   - singletonFactories
   [[循环依赖]]
```

> [!warning] 重要
> 上面代码框里的 `[[...]]` 只是视觉地图，不会被 Obsidian 当成双链。真正的双链放在下面各个阶段的小节里。

## 分阶段阅读地图

### L1. BeanDefinition 从哪里来

先看：

[[1、BeanDefinition的资源定位过程]]

这一篇解决的是：

- Spring 从哪里开始启动 IoC 容器。
- `refresh()` 为什么是容器启动主入口。
- XML 配置文件如何被定位成 `Resource`。
- `XmlBeanDefinitionReader` 是怎么被创建出来的。

这一阶段还没有真正创建业务 Bean。

这里重点理解一句话：

> Spring 先找到配置资源，还没有创建对象。

### L2. BeanDefinition 是怎么被解析出来的

再看：

[[2、将bean解析封装成BeanDefinition]]

这一篇解决的是：

- `<bean>` 标签如何被解析。
- `id`、`name`、`class`、`parent` 等信息怎么进入 BeanDefinition。
- `<property>`、`<constructor-arg>` 怎么变成 BeanDefinition 里的属性值。
- `ref` 为什么不是立刻变成对象，而是先变成 `RuntimeBeanReference`。

这里重点理解一句话：

> BeanDefinition 是 Bean 的“说明书”，不是 Bean 本身。

### L3. BeanDefinition 放到哪里

再看：

[[3、将BeanDefinition注册进IoC容器]]

这一篇解决的是：

- 解析出来的 BeanDefinition 如何注册进容器。
- `DefaultListableBeanFactory` 为什么是核心注册中心。
- `beanDefinitionMap` 和 `beanDefinitionNames` 分别保存什么。

这里重点理解一句话：

> IoC 容器启动早期，本质上是在建立一张 beanName 到 BeanDefinition 的注册表。

### L4. BeanDefinition 还能不能被改

接着看：

[[BeanFactoryPostProcessor]]

这一篇解决的是：

- BeanDefinition 已经加载完成之后，普通 Bean 创建之前，是否还能改 BeanDefinition。
- `BeanFactoryPostProcessor` 为什么作用在 BeanDefinition 层面。
- `BeanDefinitionRegistryPostProcessor` 为什么比普通 `BeanFactoryPostProcessor` 更早。
- 为什么不建议在 BeanFactoryPostProcessor 里过早调用 `getBean()`。

这里重点理解一句话：

> BeanFactoryPostProcessor 改的是“Bean 的说明书”，不是已经创建好的 Bean 对象。

### L5. Bean 创建过程中谁能插手

然后看：

[[BeanPostProcessor]]

这一篇解决的是：

- BeanPostProcessor 是怎么注册进容器的。
- 它为什么发生在 Bean 初始化前后。
- `postProcessBeforeInitialization` 和 `postProcessAfterInitialization` 分别在什么位置。
- 为什么 `AutowiredAnnotationBeanPostProcessor` 这种处理器会参与依赖注入。

这里重点理解一句话：

> BeanPostProcessor 处理的是 Bean 创建过程中的对象，很多 Spring 扩展能力都靠它插手生命周期。

### L6. Bean 什么时候真正创建

再看：

[[4、依赖注入(DI)]]

这一篇是核心。

它解决的是：

- `getBean()` 如何触发 Bean 创建。
- 非懒加载单例为什么会在容器启动末尾提前创建。
- `doGetBean()` 如何处理单例、原型、父容器、依赖关系。
- `createBean()` 和 `doCreateBean()` 的主流程是什么。

这里重点理解一句话：

> BeanDefinition 注册完成不等于 Bean 已经创建；真正创建通常发生在 getBean 或非懒加载单例预实例化时。

### L7. Bean 创建时依赖怎么注入

继续看：

[[4、依赖注入(DI)]]

重点看 `populateBean()` 这一段。

它解决的是：

- Bean 先实例化，再填充属性。
- `populateBean()` 为什么是依赖注入的核心位置。
- XML 里的 `<property ref="xxx">` 怎么最终变成 `getBean("xxx")`。
- `RuntimeBeanReference` 为什么会触发依赖 Bean 的递归创建。
- `BeanWrapper` 最后如何通过 setter 或字段写入属性值。

这里重点理解一句话：

> 依赖注入不是“解析配置时”完成的，而是在 Bean 创建过程中的 populateBean 阶段完成的。

### L8. 循环依赖为什么会发生

最后看：

[[循环依赖]]

这一篇解决的是：

- A 依赖 B，B 又依赖 A 时为什么会卡住。
- Spring 为什么需要三级缓存。
- `singletonObjects`、`earlySingletonObjects`、`singletonFactories` 分别解决什么问题。
- 为什么循环依赖主要发生在属性填充阶段。
- 为什么构造器循环依赖更难解决。

这里重点理解一句话：

> 循环依赖不是 BeanDefinition 解析阶段的问题，而是 Bean 创建和依赖注入阶段的问题。

## 各文件在整条链路中的位置

| 文件 | 主要位置 | 解决的问题 |
|---|---|---|
| [[1、BeanDefinition的资源定位过程]] | refresh 早期 | 配置资源从哪里来 |
| [[2、将bean解析封装成BeanDefinition]] | BeanDefinition 解析 | 配置如何变成 BeanDefinition |
| [[3、将BeanDefinition注册进IoC容器]] | BeanDefinition 注册 | BeanDefinition 放到哪个容器结构里 |
| [[BeanFactoryPostProcessor]] | Bean 创建前 | 如何修改 BeanDefinition |
| [[BeanPostProcessor]] | Bean 创建中 / 初始化前后 | 如何插手 Bean 生命周期 |
| [[4、依赖注入(DI)]] | Bean 创建主流程 | getBean、createBean、populateBean、initializeBean |
| [[循环依赖]] | Bean 创建中的依赖注入阶段 | Spring 如何解决属性注入场景下的循环依赖 |

## 和 @Autowired 的关系

学 `@Autowired` 之前，至少要先明白：

1. BeanDefinition 只是 Bean 的定义，不是对象。
2. `getBean()` 才会真正触发 Bean 创建。
3. Bean 创建时会经过 `doCreateBean()`。
4. 依赖注入主要发生在 `populateBean()`。
5. BeanPostProcessor 可以插手 Bean 创建过程。

所以 `@Autowired` 不适合作为 IoC 的第一个入口。

更合理的顺序是：

```text
BeanDefinition 是什么
    ↓
BeanFactory 里保存了什么
    ↓
getBean 如何创建 Bean
    ↓
populateBean 如何注入依赖
    ↓
BeanPostProcessor 如何插手
    ↓
AutowiredAnnotationBeanPostProcessor 如何处理 @Autowired
```

也就是说，`@Autowired` 应该放在 IoC 主线之后看：

[[Spring Bean 创建：Autowired 依赖注入流程]]

## 容易混淆的边界

### BeanDefinition 和 Bean 不是一回事

BeanDefinition 是定义信息。

Bean 是真正创建出来的对象。

```text
BeanDefinition：UserService 这个 Bean 应该用哪个 class、有哪些属性、依赖谁
Bean：new 出来的 UserServiceImpl 对象
```

### IoC 容器启动和依赖注入不是一回事

IoC 容器启动早期主要做 BeanDefinition 的定位、解析和注册。

依赖注入发生在 Bean 创建过程中。

### BeanFactoryPostProcessor 和 BeanPostProcessor 不是一回事

BeanFactoryPostProcessor 处理 BeanDefinition。

BeanPostProcessor 处理 Bean 对象创建过程。

### XML property 注入和 @Autowired 注入入口不同，但都会落到 Bean 创建阶段

XML `<property>` 的依赖信息来自 BeanDefinition。

`@Autowired` 的依赖信息来自 `AutowiredAnnotationBeanPostProcessor` 扫描字段、构造器或方法。

两者来源不同，但都服务于 Bean 创建过程中的依赖装配。

### 循环依赖不是单独的知识点

循环依赖要放在 `doCreateBean()` 和 `populateBean()` 里理解。

如果不知道 Bean 是先实例化、再属性填充，就很难理解为什么 Spring 可以暴露“半成品 Bean”。

## 建议阅读顺序

第一轮只读主线，不抠源码细节：

1. [[1、BeanDefinition的资源定位过程]]
2. [[2、将bean解析封装成BeanDefinition]]
3. [[3、将BeanDefinition注册进IoC容器]]
4. [[4、依赖注入(DI)]]
5. [[循环依赖]]

第二轮再补扩展点：

1. [[BeanFactoryPostProcessor]]
2. [[BeanPostProcessor]]
3. [[Spring Bean 创建：Autowired 依赖注入流程]]

## 最小理解闭环

如果只想先建立 IoC 的基本认知，可以先抓住这条链：

```text
refresh()
    ↓
loadBeanDefinitions()
    ↓
解析 XML / 配置
    ↓
注册 BeanDefinition
    ↓
getBean()
    ↓
doCreateBean()
    ↓
createBeanInstance()
    ↓
populateBean()
    ↓
initializeBean()
```

这条链跑通之后，再看 `@Autowired`、循环依赖、BeanPostProcessor，才不会觉得它们是散的。
