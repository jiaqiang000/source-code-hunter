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

## refresh 之前：是谁把容器启动起来的

> [!note] 入口边界
> `ApplicationContext.refresh()` 不是 JVM 自动调用的。
> 它一定是某个启动入口先创建了 `ApplicationContext`，配置好资源位置、父容器、环境等信息，然后显式或间接调用 `refresh()`。

- 普通 XML 启动
  - 典型写法：`new ClassPathXmlApplicationContext(...)` / `new FileSystemXmlApplicationContext(...)`
  - 构造器接收 XML 路径。
  - 构造器内部设置 `configLocations`。
  - 如果 `refresh=true`，构造器内部直接调用 `refresh()`。
  - 然后进入下面的 `ApplicationContext.refresh()` 主流程。

- 普通注解启动
  - 典型写法：`new AnnotationConfigApplicationContext(AppConfig.class)` 或 `new AnnotationConfigApplicationContext("basePackage")`
  - 构造器先注册配置类，或者扫描包路径。
  - 注册和扫描会先形成一批 `BeanDefinition`。
  - 构造器随后调用 `refresh()`。
  - 然后进入下面的 `ApplicationContext.refresh()` 主流程。

- 传统 Web 父容器启动
  - Servlet 容器读取 `web.xml`。
  - 发现 `ContextLoaderListener`。
  - 调用 `ContextLoaderListener.contextInitialized(...)`。
  - 进入 `ContextLoader.initWebApplicationContext(servletContext)`。
  - 根据 `contextClass` 创建 `WebApplicationContext`。
  - 根据 `contextConfigLocation` 设置配置入口。
  - 调用 `ContextLoader.configureAndRefreshWebApplicationContext(...)`。
  - 最后调用 `wac.refresh()`。
  - 然后进入下面的 `ApplicationContext.refresh()` 主流程。
  - 你项目里的 `biz` 和 `web_bookln_java` 都是这种形态：父容器由 `ContextLoaderListener` 启动，`contextClass` 指向 `AnnotationConfigWebApplicationContext`，`contextConfigLocation` 分别指向项目配置类。

- Spring MVC 子容器启动
  - Servlet 容器读取 `web.xml`。
  - 发现 `DispatcherServlet`，并且 `load-on-startup` 大于等于 0。
  - Servlet 容器初始化 `DispatcherServlet`。
  - 进入 `FrameworkServlet.initServletBean()`。
  - 调用 `FrameworkServlet.initWebApplicationContext()`。
  - 如果没有外部传入现成的子容器，就创建一个新的 `WebApplicationContext`。
  - 把父容器设置为前面的 root `WebApplicationContext`。
  - 根据 Servlet 自己的 `contextConfigLocation` 设置 MVC 配置入口。
  - 调用 `FrameworkServlet.configureAndRefreshWebApplicationContext(...)`。
  - 最后调用 `wac.refresh()`。
  - 然后进入下面的 `ApplicationContext.refresh()` 主流程。
  - 这也是为什么传统 Web 项目里可能看到不止一次 `refresh()`：父容器刷新一次，MVC 子容器也会刷新一次。

- Spring Boot 启动
  - 典型写法：`SpringApplication.run(...)`。
  - `SpringApplication` 创建具体的 `ApplicationContext`。
  - 准备环境、监听器、初始化器等启动上下文。
  - 调用内部刷新入口，最终仍然落到 `ApplicationContext.refresh()`。
  - 本文不展开 Boot 自动配置，只把它当作另一种调用到 `refresh()` 的入口。

## 完整生命周期地图

> [!note] 地图边界
> 这张地图以当前 `source-code-hunter/docs/Spring/IoC` 这组 XML 主线文档为骨架。
> 注解、Java Config、`@MapperScan` 等入口也会形成 `BeanDefinition`，但具体解析入口不同；本文先把它们放在同一条 IoC 生命周期里理解。
> 本图只补到 Spring IoC 容器启动和 Bean 创建生命周期；Web 请求链、AOP、事务、应用关闭等阶段不在这里展开。

- `ApplicationContext.refresh()`
  - `prepareRefresh()`
    - 准备刷新状态、环境属性和容器 active 标记。
    - 当前没有专门文档，先不链接。
  - 条件装配 / 条件注册边界
    - 对应 `L1.1`。
    - Spring Boot 的 `@Conditional`、自动配置条件等，会影响哪些配置类或 BeanDefinition 能进入后续流程。
    - 这不是当前 XML 主线文档的重点，先只放在 BeanDefinition 形成之前理解，不展开 Boot 自动配置。
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
      - 对应 `L2.1.1 Bean 实例化`。
      - 选择构造器：构造器注入、`@Autowired` 构造器判断会落在这里。
      - 解析构造器参数依赖：`BeanFactory` 根据类型、名称、`@Qualifier`、`@Primary` 等规则找依赖。
      - 调用构造器创建对象：构造器注入最终发生在这里。
      - 对应：[[4、依赖注入(DI)]]
    - `populateBean()`
      - Bean 创建中的属性填充入口。
      - XML `<property ref="...">` 注入：[[4、依赖注入(DI)]]
      - `@Autowired` / `@Inject` / `@Value` 注解注入：[[BeanPostProcessor]]，详细见 [[Spring Bean 创建：Autowired 依赖注入流程]]
      - `@Resource` 注解注入：[[BeanPostProcessor]]，由 `CommonAnnotationBeanPostProcessor` 参与处理。
      - 对应 `L2.1.2 依赖注入 / 属性填充`。
      - 解析字段 / 方法注入点：找出哪些字段、setter 或普通方法需要被注入。
      - 查找依赖 Bean：根据注入点信息从 `BeanFactory` 中解析候选 Bean。
      - 写入字段或调用方法：字段注入、方法注入最终发生在这里。
      - 循环依赖暴露点：[[循环依赖]]
    - `applyPropertyValues()`
      - 解析 `PropertyValue`，把 `RuntimeBeanReference`、`TypedStringValue`、集合等元数据变成真实值。
      - 对应：[[4、依赖注入(DI)]]
      - 如果遇到 `RuntimeBeanReference`，会递归调用 `getBean(refName)`。
    - `BeanWrapper.setPropertyValues()`
      - 把解析后的值真正写入 Java 对象。
      - 对应：[[4、依赖注入(DI)]]
    - `initializeBean()`
      - 属性填充之后进入初始化阶段，对应 `L2.1.3 - L2.1.6`。
      - `invokeAwareMethods(...)`
        - Aware 回调的一部分，对应 `L2.1.3`。
        - 这里直接处理 `BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware`。
        - `ApplicationContextAware` 等上下文回调通常由 `ApplicationContextAwareProcessor` 在初始化前处理阶段触发，这里先不展开。
      - `applyBeanPostProcessorsBeforeInitialization(...)`
        - BeanPostProcessor 初始化前处理，对应 `L2.1.4`：[[BeanPostProcessor]]
        - `CommonAnnotationBeanPostProcessor` 会在这一段触发 `@PostConstruct`。
      - `invokeInitMethods(...)`
        - `InitializingBean.afterPropertiesSet()` / `init-method`，对应 `L2.1.5`。
        - 当前没有专门文档，先不链接。
      - `applyBeanPostProcessorsAfterInitialization(...)`
        - BeanPostProcessor 初始化后处理，对应 `L2.1.6`：[[BeanPostProcessor]]
        - AOP 代理通常在这一类后处理阶段完成，这里不展开 AOP 主线。
  - `finishRefresh()`
    - 发布容器刷新完成事件，对应 `L2.2`。
    - 当前没有专门文档，先不链接。
