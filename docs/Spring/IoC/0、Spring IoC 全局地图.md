# Spring IoC 全局地图

> [!note] 阅读目标
> 这组文档不要按“一个文档只能属于一个位置”来读。
> Spring IoC 是一条生命周期主线，很多扩展点会在多个位置出现：有的位置负责注册，有的位置负责真正执行。

## 一句话主线

Spring IoC 可以先分成两大段：

1. 容器启动阶段：把配置解析成 `BeanDefinition`，注册到 `BeanFactory`，并准备好后处理器。
2. Bean 创建阶段：根据 `BeanDefinition` 创建对象、填充属性、执行初始化、处理代理和循环依赖。

最小闭环是：

- 配置文件 / 注解 / Java Config
  - 变成 `BeanDefinition`
    - 注册到 `BeanDefinitionRegistry` / `BeanFactory`
      - 通过 `getBean()` 或非懒加载单例预实例化触发创建
        - `createBean()`
          - 实例化对象
            - 依赖注入
              - 初始化
                - 可用 Bean

## 完整生命周期地图

> [!note] 地图边界
> 这张地图以当前 `source-code-hunter/docs/Spring/IoC` 这组 XML 主线文档为骨架。
> 注解、Java Config、`@MapperScan` 等入口也会形成 `BeanDefinition`，但具体解析入口不同；本文先把它们放在同一条 IoC 生命周期里理解。

- `ApplicationContext.refresh()`
  - `prepareRefresh()`
    - 准备刷新状态、环境属性和容器 active 标记。
    - 当前没有专门文档，先不链接。
  - `obtainFreshBeanFactory()`
    - 定位 BeanDefinition 资源：[[1、BeanDefinition的资源定位过程]]
      - 重点：从容器入口走到 XML 配置资源定位。
      - 这里还没有创建业务 Bean。
    - 解析配置，封装成 BeanDefinition：[[2、将bean解析封装成BeanDefinition]]
      - 重点：`<bean>`、`<property>`、`ref` 等配置变成 `BeanDefinition` 内部元数据。
      - `ref` 此时通常只是 `RuntimeBeanReference`，还不是目标对象。
    - 注册 BeanDefinition 到容器：[[3、将BeanDefinition注册进IoC容器]]
      - 重点：`DefaultListableBeanFactory` 实现了 `BeanDefinitionRegistry`。
      - `BeanDefinition` 最终保存到 `DefaultListableBeanFactory.beanDefinitionMap`。
  - `prepareBeanFactory(beanFactory)`
    - 给 `BeanFactory` 装基础能力。
    - 当前没有专门文档，先不链接。
  - `postProcessBeanFactory(beanFactory)`
    - 留给 `ApplicationContext` 子类扩展。
    - 当前没有专门文档，先不链接。
  - `invokeBeanFactoryPostProcessors(beanFactory)`
    - 执行 BeanFactoryPostProcessor：[[BeanFactoryPostProcessor]]
      - 重点：这个阶段主要处理 `BeanDefinition` 或 `BeanFactory` 配置。
      - 它发生在普通 Bean 创建之前。
      - 这篇里的 `CustomEditorConfigurer` 例子会影响后续属性转换，所以它的效果会延伸到 Bean 创建阶段。
    - 执行 BeanDefinitionRegistryPostProcessor：[[BeanFactoryPostProcessor]]
      - 重点：它比普通 `BeanFactoryPostProcessor` 更早。
      - 它还能继续注册或修改 `BeanDefinition`。
  - `registerBeanPostProcessors(beanFactory)`
    - 注册 BeanPostProcessor：[[BeanPostProcessor]]
      - 重点：这里是注册位置。
      - `BeanPostProcessor` 会先被创建并加入 `BeanFactory`，准备插手后续 Bean 创建流程。
      - 这篇不能只放在这里理解，因为它真正执行是在后面的 Bean 创建阶段。
  - `initMessageSource()`
    - 初始化国际化消息能力。
    - 当前没有专门文档，先不链接。
  - `initApplicationEventMulticaster()`
    - 初始化事件广播器。
    - 当前没有专门文档，先不链接。
  - `onRefresh()`
    - 留给具体上下文做特殊初始化。
    - 当前没有专门文档，先不链接。
  - `registerListeners()`
    - 注册监听器。
    - 当前没有专门文档，先不链接。
  - `finishBeanFactoryInitialization(beanFactory)`
    - 触发非懒加载单例预实例化：[[4、依赖注入(DI)]]
      - 更准确地说，这篇现在覆盖的是“Bean 创建与依赖注入主流程”。
      - 原标题叫 DI，但它前半部分实际在讲 `getBean()`、`createBean()`、`doCreateBean()`。
    - `preInstantiateSingletons()`
      - 遍历非懒加载单例。
      - 对符合条件的 Bean 调用 `getBean()`。
      - 对应：[[4、依赖注入(DI)]]
    - `getBean()` / `doGetBean()`
      - Bean 创建触发入口。
      - 先查缓存，再查父容器，再拿 `RootBeanDefinition`，最后按作用域决定是否创建。
      - 对应：[[4、依赖注入(DI)]]
    - `createBean()` / `doCreateBean()`
      - 单个 Bean 创建主流程。
      - 串起实例化、属性填充、初始化、销毁注册。
      - 对应：[[4、依赖注入(DI)]]
    - `createBeanInstance()`
      - 先创建一个 Java 对象。
      - 可能走默认构造器、构造器自动装配、工厂方法或 CGLIB。
      - 对应：[[4、依赖注入(DI)]]
    - `populateBean()`
      - Bean 创建中的属性填充入口。
      - XML `<property ref="...">` 注入：[[4、依赖注入(DI)]]
      - `@Autowired` / `@Value` 注解注入：[[BeanPostProcessor]]，详细见 [[Spring Bean 创建：Autowired 依赖注入流程]]
      - 循环依赖暴露点：[[循环依赖]]
    - `applyPropertyValues()`
      - 解析 `PropertyValue`，把 `RuntimeBeanReference`、`TypedStringValue`、集合等元数据变成真实值。
      - 对应：[[4、依赖注入(DI)]]
      - 如果遇到 `RuntimeBeanReference`，会递归调用 `getBean(refName)`。
    - `BeanWrapper.setPropertyValues()`
      - 把解析后的值真正写入 Java 对象。
      - 对应：[[4、依赖注入(DI)]]
    - `initializeBean()`
      - 属性填充之后进入初始化阶段。
      - `postProcessBeforeInitialization()` / `postProcessAfterInitialization()` 的执行位置：[[BeanPostProcessor]]
  - `finishRefresh()`
    - 发布容器刷新完成事件。
    - 当前没有专门文档，先不链接。
