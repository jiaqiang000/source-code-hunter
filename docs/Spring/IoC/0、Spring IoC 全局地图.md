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

- Spring 从哪里开始启动 IoC 容器：通常从 `ApplicationContext.refresh()` 这条主线进入；XML 容器的构造方法最终也会走到这里。
- `refresh()` 为什么是容器启动主入口：因为它负责重新准备 BeanFactory、加载 BeanDefinition、注册后处理器，并触发非懒加载单例的创建。
- XML 配置文件如何被定位成 `Resource`：Spring 先把配置路径交给容器的资源加载能力，包装成 `Resource`，后续 reader 才能统一读取。
- `XmlBeanDefinitionReader` 是怎么被创建出来的：`AbstractXmlApplicationContext` 在加载 BeanDefinition 时创建它，并给它设置 BeanFactory、Environment、ResourceLoader 等上下文。

这一阶段还没有真正创建业务 Bean。

这里重点理解一句话：

> Spring 先找到配置资源，还没有创建对象。

### L2. BeanDefinition 是怎么被解析出来的

再看：

[[2、将bean解析封装成BeanDefinition]]

这一篇解决的是：

- `<bean>` 标签如何被解析：Spring 会遍历 XML 节点，把默认命名空间下的 `<bean>` 交给 `BeanDefinitionParserDelegate` 解析。
- `id`、`name`、`class`、`parent` 等信息怎么进入 BeanDefinition：这些标签属性会被读出来，填进 `BeanDefinition` 的元信息里。
- `<property>`、`<constructor-arg>` 怎么变成 BeanDefinition 里的属性值：它们会被解析成 `PropertyValue` 或构造器参数，暂存在 BeanDefinition 中。
- `ref` 为什么不是立刻变成对象，而是先变成 `RuntimeBeanReference`：因为目标 Bean 可能还没创建，Spring 这里只能先记录“将来要引用谁”。

这里重点理解一句话：

> BeanDefinition 是 Bean 的“说明书”，不是 Bean 本身。

### L3. BeanDefinition 放到哪里

再看：

[[3、将BeanDefinition注册进IoC容器]]

这一篇解决的是：

- 解析出来的 BeanDefinition 如何注册进容器：`BeanDefinitionReaderUtils` 最终会调用注册器，把 beanName 和 BeanDefinition 放进 BeanFactory。
- `DefaultListableBeanFactory` 为什么是核心注册中心：它既保存 BeanDefinition，也负责后续按这些定义创建 Bean。
- `beanDefinitionMap` 和 `beanDefinitionNames` 分别保存什么：前者按 beanName 查 BeanDefinition，后者保存 beanName 的注册顺序和列表。

这里重点理解一句话：

> IoC 容器启动早期，本质上是在建立一张 beanName 到 BeanDefinition 的注册表。

### L4. BeanDefinition 还能不能被改

接着看：

[[BeanFactoryPostProcessor]]

这一篇解决的是：

- BeanDefinition 已经加载完成之后，普通 Bean 创建之前，是否还能改 BeanDefinition：可以，这正是 `BeanFactoryPostProcessor` 的主要位置。
- `BeanFactoryPostProcessor` 为什么作用在 BeanDefinition 层面：它运行时大部分普通 Bean 还没创建，所以更适合修改 Bean 的定义信息。
- `BeanDefinitionRegistryPostProcessor` 为什么比普通 `BeanFactoryPostProcessor` 更早：因为它还能新增或注册 BeanDefinition，必须先把“有哪些 Bean”确定下来。
- 为什么不建议在 BeanFactoryPostProcessor 里过早调用 `getBean()`：这会提前创建普通 Bean，可能绕过还没准备好的后处理器、代理和配置逻辑。

这里重点理解一句话：

> BeanFactoryPostProcessor 改的是“Bean 的说明书”，不是已经创建好的 Bean 对象。

### L5. Bean 创建过程中谁能插手

然后看：

[[BeanPostProcessor]]

这一篇解决的是：

- BeanPostProcessor 是怎么注册进容器的：`refresh()` 中会执行 `registerBeanPostProcessors(beanFactory)`，把这些处理器提前加入 BeanFactory。
- 它为什么发生在 Bean 初始化前后：它的设计目的就是在 Bean 初始化回调前后插手，比如补充处理、包装对象或创建代理。
- `postProcessBeforeInitialization` 和 `postProcessAfterInitialization` 分别在什么位置：前者在初始化方法之前，后者在初始化方法之后。
- 为什么 `AutowiredAnnotationBeanPostProcessor` 这种处理器会参与依赖注入：它实现了更早的属性处理扩展点，`populateBean()` 会调用它来处理 `@Autowired` 字段和方法。

这里重点理解一句话：

> BeanPostProcessor 处理的是 Bean 创建过程中的对象，很多 Spring 扩展能力都靠它插手生命周期。

### L6. Bean 什么时候真正创建

再看：

[[4、依赖注入(DI)]]

这一篇是核心。

它解决的是：

- `getBean()` 如何触发 Bean 创建：`getBean()` 会进入 `doGetBean()`，如果缓存里没有现成单例，就会走到 `createBean()`。
- 非懒加载单例为什么会在容器启动末尾提前创建：这样可以让单例 Bean 在启动时就准备好，也能尽早暴露配置或依赖错误。
- `doGetBean()` 如何处理单例、原型、父容器、依赖关系：它先查缓存和父容器，再处理 `dependsOn`，最后按 scope 决定创建或返回对象。
- `createBean()` 和 `doCreateBean()` 的主流程是什么：`createBean()` 是外层入口，`doCreateBean()` 负责实例化、依赖注入、初始化和销毁注册。

这里重点理解一句话：

> BeanDefinition 注册完成不等于 Bean 已经创建；真正创建通常发生在 getBean 或非懒加载单例预实例化时。

### L7. Bean 创建时依赖怎么注入

继续看：

[[4、依赖注入(DI)]]

重点看 `populateBean()` 这一段。

它解决的是：

- Bean 先实例化，再填充属性：Spring 必须先有一个对象实例，才能给它设置属性或注入依赖。
- `populateBean()` 为什么是依赖注入的核心位置：它负责处理属性值、自动装配和属性相关的后处理器，是 Bean 创建中“把依赖塞进去”的阶段。
- XML 里的 `<property ref="xxx">` 怎么最终变成 `getBean("xxx")`：`ref` 会先变成 `RuntimeBeanReference`，真正注入时再解析这个引用并调用 `getBean()`。
- `RuntimeBeanReference` 为什么会触发依赖 Bean 的递归创建：如果被引用的 Bean 还没创建，解析引用时就会进入它自己的 `getBean()` 创建流程。
- `BeanWrapper` 最后如何通过 setter 或字段写入属性值：普通 XML property 通常通过属性访问器调用 setter 写入；字段注入更多是 `@Autowired` 这类后处理器直接反射写入。

这里重点理解一句话：

> 依赖注入不是“解析配置时”完成的，而是在 Bean 创建过程中的 populateBean 阶段完成的。

### L8. 循环依赖为什么会发生

最后看：

[[循环依赖]]

这一篇解决的是：

- A 依赖 B，B 又依赖 A 时为什么会卡住：A 创建时需要 B，B 创建时又需要 A，如果只能等完整对象，就会互相等待。
- Spring 为什么需要三级缓存：它需要同时区分完整单例、提前暴露的半成品对象，以及可以生成早期引用的工厂。
- `singletonObjects`、`earlySingletonObjects`、`singletonFactories` 分别解决什么问题：一级缓存放完整 Bean，二级缓存放早期引用，三级缓存放能生成早期引用的 `ObjectFactory`。
- 为什么循环依赖主要发生在属性填充阶段：对象已经实例化但属性还没填完，此时才会因为注入其他 Bean 触发互相引用。
- 为什么构造器循环依赖更难解决：构造器没执行完之前对象还不存在，Spring 没有可以提前暴露的半成品实例。

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
