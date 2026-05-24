# BeanPostProcessor 源码分析

狭义的 `BeanPostProcessor` 接口也叫 Bean 后置处理器，作用是在 Bean 对象实例化和依赖注入完成后，在配置文件 bean 的 init-method(初始化方法)或者 `InitializingBean.afterPropertiesSet()` 的前后添加我们自己的处理逻辑。

注意：这句话只描述下面这个狭义接口的两个方法。Spring 里更常说的 `BeanPostProcessor` 体系还包括多个子接口，它们可以插入到实例化前、构造器选择、属性填充、早期代理、初始化前后等不同阶段。

接口的源码如下。

```java
public interface BeanPostProcessor {
    /**
     * 实例化、依赖注入完毕，
     * 在调用显示的初始化之前完成一些定制的初始化任务
     */
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    /**
     * 实例化、依赖注入、初始化完毕时执行
     */
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

## 先区分：狭义 BeanPostProcessor 和广义 BeanPostProcessor 体系

> [!note] 先抓核心
> 不是所有 `BeanPostProcessor` 都会经历所有阶段。
>
> Spring 的逻辑是：
>
> 1. 先把所有 `BeanPostProcessor` 注册到 `BeanFactory`。
> 2. 再按 `instanceof` 把它们分到不同缓存列表里。
> 3. Bean 创建到不同阶段时，Spring 只遍历对应类型的处理器列表。
> 4. 某个处理器真正做不做事，取决于它有没有实现 / 重写对应方法。

### 1. 狭义 BeanPostProcessor：只管初始化前后

狭义的 `BeanPostProcessor` 只有两个核心方法：

```text
BeanPostProcessor
  -> postProcessBeforeInitialization(...)
  -> postProcessAfterInitialization(...)
```

它对应的位置是：

```text
initializeBean()
  -> applyBeanPostProcessorsBeforeInitialization(...)
  -> invokeInitMethods(...)
  -> applyBeanPostProcessorsAfterInitialization(...)
```

也就是说，如果一个类只是：

```java
public class MyProcessor implements BeanPostProcessor {
}
```

那它会经过的典型回调是：

```text
postProcessBeforeInitialization(...)
postProcessAfterInitialization(...)
```

也就是说，它稳定参与的是 `initializeBean()` 初始化前后，不会自动参与 `populateBean()` 的属性注入，也不会自动参与构造器选择。

### 2. 广义 BeanPostProcessor 体系：接口、缓存列表、源码调用点要一起看

Spring 里常说的 `BeanPostProcessor`，很多时候不是只指狭义接口，而是指一整套后处理器体系。

不要把它理解成：

```text
一个 BeanPostProcessor 自动经历所有阶段
```

更准确是：

```text
具体处理器实现了哪些接口
  -> Spring 注册后用 instanceof 把它分到哪些缓存列表
    -> Bean 创建到某个阶段时，只遍历对应缓存列表
      -> 调用这个接口在该阶段暴露的方法
```

常见接口、缓存列表、调用位置可以放在一张图里看：

```text
beanFactory.beanPostProcessors
  说明：所有注册进 BeanFactory 的 BeanPostProcessor 都先在这个总列表里
  │
  ├─ BeanPostProcessor
  │  接口方法：
  │    - postProcessBeforeInitialization(...)
  │    - postProcessAfterInitialization(...)
  │  调用位置：
  │    initializeBean()
  │      -> applyBeanPostProcessorsBeforeInitialization(...)
  │      -> invokeInitMethods(...)
  │      -> applyBeanPostProcessorsAfterInitialization(...)
  │
  ├─ instantiationAware 缓存列表
  │  分类条件：
  │    bp instanceof InstantiationAwareBeanPostProcessor
  │  接口关系：
  │    InstantiationAwareBeanPostProcessor extends BeanPostProcessor
  │  接口方法：
  │    - postProcessBeforeInstantiation(...)
  │    - postProcessAfterInstantiation(...)
  │    - postProcessProperties(...)
  │    - postProcessPropertyValues(...)
  │  调用位置：
  │    resolveBeforeInstantiation()
  │      -> postProcessBeforeInstantiation(...)
  │
  │    populateBean()
  │      -> postProcessAfterInstantiation(...)
  │      -> postProcessProperties(...)
  │
  ├─ smartInstantiationAware 缓存列表
  │  分类条件：
  │    bp instanceof SmartInstantiationAwareBeanPostProcessor
  │  接口关系：
  │    SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor
  │  接口方法：
  │    - predictBeanType(...)
  │    - determineCandidateConstructors(...)
  │    - getEarlyBeanReference(...)
  │  调用位置：
  │    createBeanInstance()
  │      -> determineCandidateConstructors(...)
  │
  │    doCreateBean() 循环依赖提前暴露
  │      -> getEarlyBeanReference(...)
  │
  └─ mergedDefinition 缓存列表
     分类条件：
       bp instanceof MergedBeanDefinitionPostProcessor
     接口关系：
       MergedBeanDefinitionPostProcessor extends BeanPostProcessor
     接口方法：
       - postProcessMergedBeanDefinition(...)
     调用位置：
       doCreateBean()
         -> applyMergedBeanDefinitionPostProcessors(...)
```

所以真正要记住的是：

```text
接口决定“它有没有资格进入某类缓存列表”。
缓存列表决定“某个生命周期阶段会遍历哪些处理器”。
源码调用点决定“这个阶段具体调用哪个方法”。
具体实现类决定“被调用后实际做什么”。
```

一个处理器如果实现多个接口，就可能进入多个缓存列表。

但是：

```text
进入多个缓存列表
  不等于每个阶段都一定做事
  还要看它有没有重写对应方法，以及方法里有没有实际逻辑
```

### 3. AutowiredAnnotationBeanPostProcessor 是具体实现类，不是新阶段

`AutowiredAnnotationBeanPostProcessor` 是一个具体实现类。

它的声明大体是：

```text
AutowiredAnnotationBeanPostProcessor
  implements SmartInstantiationAwareBeanPostProcessor
  implements MergedBeanDefinitionPostProcessor
  implements PriorityOrdered
  implements BeanFactoryAware
```

所以它可能出现在多个阶段：

```text
createBeanInstance()
  -> SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors(...)
     典型实现：AutowiredAnnotationBeanPostProcessor
     作用：处理 @Autowired 构造器候选判断

doCreateBean()
  -> MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition(...)
     典型实现：AutowiredAnnotationBeanPostProcessor
     作用：提前解析/缓存字段和方法注入元数据

populateBean()
  -> InstantiationAwareBeanPostProcessor.postProcessProperties(...)
     典型实现：AutowiredAnnotationBeanPostProcessor
     作用：真正处理 @Autowired / @Value 字段或方法注入
     对应：[[4、依赖注入(DI)]] 的 [04.4.4]

doCreateBean() 循环依赖提前暴露
  -> SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference(...)
     AutowiredAnnotationBeanPostProcessor 有资格被遍历
     但它不是 AOP 早期代理的重点实现，AOP 重点看 AbstractAutoProxyCreator

initializeBean()
  -> BeanPostProcessor.postProcessBeforeInitialization(...)
  -> BeanPostProcessor.postProcessAfterInitialization(...)
     AutowiredAnnotationBeanPostProcessor 也具备 BeanPostProcessor 身份
     但 @Autowired 字段 / 方法注入主线不在这里完成
```

### 4. 回到 @Autowired：最重要的是 populateBean 这条线

对于字段注入、方法注入，重点链路是：

```text
populateBean()
  -> for (InstantiationAwareBeanPostProcessor bp : instantiationAware)
     -> bp.postProcessProperties(pvs, bean, beanName)
        -> 如果 bp 的真实对象是 AutowiredAnnotationBeanPostProcessor
           -> findAutowiringMetadata(...)
           -> metadata.inject(...)
           -> 完成 @Autowired / @Value 字段或方法注入
```

所以：

```text
InstantiationAwareBeanPostProcessor 是接口阶段。
AutowiredAnnotationBeanPostProcessor 是这个阶段里的具体实现。
```

这就是为什么 `4、依赖注入(DI)` 里的 `[04.4.4] 可选：注解注入后处理器处理字段 / 方法注入`，可以对应到本文的 `AutowiredAnnotationBeanPostProcessor` 例子。

### 5. 这一节最终要记住什么

不要把 `BeanPostProcessor` 理解成一个固定阶段。

更准确的理解是：

```text
BeanPostProcessor 是一套扩展点体系。

Spring 在 Bean 创建生命周期的不同位置，
按接口类型筛选并遍历处理器。

一个具体处理器实现了哪些接口，
决定它能参与哪些阶段。

它在某个阶段真正做什么，
取决于它重写了哪个方法。
```

## resolveBeforeInstantiation 常见吗：业务项目一般不用自己扩展

问题是：

`resolveBeforeInstantiation` 是需要自己扩展的吗？本地代码有这样做的吗？常见吗？一般在什么情况下使用？

不需要你自己扩展。准确说：

`resolveBeforeInstantiation()` 是 Spring 内部方法，你不会自己调用它。
如果你真想参与这个入口，才需要自己写一个 `InstantiationAwareBeanPostProcessor`，重写 `postProcessBeforeInstantiation(...)`。

但这个入口不常见，尤其在业务项目里很少自己写。

本地业务项目里没看到业务代码直接实现：

```java
InstantiationAwareBeanPostProcessor
SmartInstantiationAwareBeanPostProcessor
postProcessBeforeInstantiation
```

本地常见的是普通 `BeanPostProcessor`，例如：

- [RequestAuthInfoAnnotationBeanPostProcessor.java](/Users/luojiaqiang/IdeaProjects/gateway&commons-web&bookln/commons-web/commons-web-servlet/src/main/java/com/cqtouch/web/auth/RequestAuthInfoAnnotationBeanPostProcessor.java:18)
  `implements BeanPostProcessor`，在初始化前扫描 `@RequestAuthInfo`，整理 `uri -> AuthInfo`。

- [RequestParamsProcessor.java](/Users/luojiaqiang/IdeaProjects/slt/app/src/main/java/com/yunti/boot/slt/processor/RequestParamsProcessor.java:22)
  `implements BeanPostProcessor`，在初始化后扫描 Controller 参数规则。

- [RequestMappingHandlerMappingBeanPostProcessor.java](/Users/luojiaqiang/IdeaProjects/gateway/src/main/java/com/yunti/boot/gateway/beans/processor/RequestMappingHandlerMappingBeanPostProcessor.java:7)
  `implements BeanPostProcessor`，调整 `RequestMappingHandlerMapping` 的 order。

这些都不是 `resolveBeforeInstantiation` 那条路。

更基础地说，Spring 创建 Bean 时是这样问的：

```text
createBean("a")
  -> resolveBeforeInstantiation("a")
     -> 遍历 InstantiationAwareBeanPostProcessor
     -> 调用 postProcessBeforeInstantiation(beanClass, beanName)
        -> 如果返回 null：继续正常创建 Bean
        -> 如果返回非 null：直接拿这个对象当 Bean，跳过普通实例化
```

Spring 源码里也写得很清楚：`postProcessBeforeInstantiation` 是特殊用途，主要给框架内部用，普通场景尽量实现普通 `BeanPostProcessor` 就行。位置在 [InstantiationAwareBeanPostProcessor.java](/Users/luojiaqiang/IdeaProjects/spring-source-study/spring-framework/spring-beans/src/main/java/org/springframework/beans/factory/config/InstantiationAwareBeanPostProcessor.java:35)。

所以你可以这样建立直觉：

```text
BeanPostProcessor：
  常见。
  Bean 已经创建出来了，你在初始化前后看一眼、改一改、记录元数据、甚至返回代理。

InstantiationAwareBeanPostProcessor：
  更底层。
  可以插到实例化前、实例化后、属性填充前。
  业务项目很少自己写。

postProcessBeforeInstantiation：
  更特殊。
  Bean 还没有 new 出来。
  你如果返回一个对象，Spring 就不再正常 new 原始 Bean。
```

普通 AOP / `@Transactional` 一般也不是靠这里直接创建代理。你本地 Spring 源码里的 `AbstractAutoProxyCreator#postProcessBeforeInstantiation(...)` 确实有这个方法，但它主要在有 custom `TargetSource` 时才提前创建代理；正常事务/AOP 更常见是在 `postProcessAfterInitialization(...)` 创建代理，循环依赖时才走 `getEarlyBeanReference(...)`。

所以结论是：

`resolveBeforeInstantiation` 要认识，但先不用重点学怎么自定义。
你现在更应该重点掌握的是普通 `BeanPostProcessor`、`populateBean()`、`initializeBean()`、`postProcessAfterInitialization()`、以及循环依赖里的 `getEarlyBeanReference()`。

## 为什么实现子接口后就能在对应阶段运行

不是接口继承本身有魔法，而是 Spring 源码在不同阶段显式遍历处理器列表，并用 `instanceof` 判断它有没有实现某个更具体的接口。

比如属性填充阶段大概就是这种逻辑：

```java
for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp =
                (InstantiationAwareBeanPostProcessor) bp;

        ibp.postProcessAfterInstantiation(bean, beanName);
    }
}
```

所以真正的过程是：

```text
1. Spring 先把所有 BeanPostProcessor 注册到 BeanFactory。
2. 后面创建普通 Bean 时，Spring 走到某个生命周期阶段。
3. Spring 遍历这批 BeanPostProcessor。
4. 如果当前处理器实现了某个子接口，就调用这个子接口对应的方法。
```

## 多个 BeanPostProcessor 会怎么处理

可以有多个类实现 `BeanPostProcessor`，也可以有多个类实现 `InstantiationAwareBeanPostProcessor`。

Spring 注册时会先找出所有实现 `BeanPostProcessor` 的 Bean：

```java
String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
```

然后按顺序分组：

```text
PriorityOrdered
  -> Ordered
    -> 普通 BeanPostProcessor
```

后面每创建一个普通 Bean，都会按注册顺序遍历这批处理器。

如果有两个普通处理器：

```java
public class AProcessor implements BeanPostProcessor {}
public class BProcessor implements BeanPostProcessor {}
```

那么初始化阶段大致是：

```text
AProcessor.postProcessBeforeInitialization(...)
BProcessor.postProcessBeforeInitialization(...)
invokeInitMethods(...)
AProcessor.postProcessAfterInitialization(...)
BProcessor.postProcessAfterInitialization(...)
```

如果有两个 `InstantiationAwareBeanPostProcessor`：

```java
public class CProcessor implements InstantiationAwareBeanPostProcessor {}
public class DProcessor implements InstantiationAwareBeanPostProcessor {}
```

那么属性填充阶段会额外经过：

```text
CProcessor.postProcessAfterInstantiation(...)
DProcessor.postProcessAfterInstantiation(...)

CProcessor.postProcessProperties(...)
DProcessor.postProcessProperties(...)
```

这里有两个细节：

```text
postProcessAfterInstantiation(...)
  如果某个处理器返回 false，Spring 可能直接停止后续属性填充。

postProcessProperties(...)
  是链式处理，前一个处理器返回的 PropertyValues 会继续传给后一个处理器。
```

所以可以这样总结：

> [!summary] 最终理解
> `BeanPostProcessor` 狭义上只有初始化前后两个方法。
> 
> 但 Spring 的 BeanPostProcessor 体系有很多子接口。Spring 在不同生命周期阶段主动遍历处理器列表，并用 `instanceof` 判断是否调用对应子接口方法。
> 
> 因此，多个处理器可以同时存在；它们按注册顺序依次参与对应阶段。

## BeanPostProcessor 的注册时机和排序

共有两种方式实现：

- 实现 BeanPostProcessor 接口，然后将此类注册到 Spring 即可；
- 第二种是通过`ConfigurableBeanFactory` 的 addBeanPostProcessor 方法进行注册。

BeanPostProcess 可以有多个，并且可以通过设置 order 属性来控制这些 BeanPostProcessor 实例的执行顺序。 仅当 BeanPostProcessor 实现 Ordered 接口时,才能设置此属性，或者 PriorityOrdered 接口。

如果某个类实现了 BeanPostProcessor 则它会在 AbstractApplicationContext 中的 registerBeanPostProcessors(beanFactory)方法中创建 bean 而不是和普通的 bean 一样在 finishBeanFactoryInitialization(beanFactory)中才被创建。

当我们注册 BeanPostProcessor 的时候，其中我省略了大部分无关代码：

```java
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    // 找到实现BeanPostProcessor接口的子类bean名称
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    // 加一是因为下一行又加了一个BPP(BeanPostProcessor)
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

    // 排序省略，没啥好讲的，你只需要知道没有实现排序接口的BPP放在了nonOrderedPostProcessorNames这里
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // 中间省略了一些....

    // 重点来了
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String ppName : nonOrderedPostProcessorNames) {
        // 当它getBean的时候我们的BPP就开始创建
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
    	    internalPostProcessors.add(pp);
        }
    }
}
```

## AutowiredAnnotationBeanPostProcessor：在 populateBean 阶段处理 @Autowired

在此我举例一个典型的例子 `AutowiredAnnotationBeanPostProcessor`。

它属于 `BeanPostProcessor` 体系，但更准确地说，它实现了 `SmartInstantiationAwareBeanPostProcessor`、`MergedBeanDefinitionPostProcessor`、`PriorityOrdered`、`BeanFactoryAware` 等接口。它不是只靠狭义 `BeanPostProcessor` 的 `postProcessBeforeInitialization` / `postProcessAfterInitialization` 两个方法工作。

这里先只看它和 `@Autowired` / `@Value` 字段或方法注入最相关的一条线：在 `populateBean()` 阶段，Spring 遍历 `InstantiationAwareBeanPostProcessor`，调用 `postProcessProperties(...)`，`AutowiredAnnotationBeanPostProcessor` 就是在这里解析注入点并完成注入。

对应到 [[4、依赖注入(DI)]]，它就是全局导图里的 `[04.4.4] 可选：注解注入后处理器处理字段 / 方法注入`。

> [!note] 为什么遍历的是 InstantiationAwareBeanPostProcessor，却会执行 AutowiredAnnotationBeanPostProcessor
> 这是接口继承和实现类的关系：
>
> ```text
> BeanPostProcessor                                      接口
> └─ InstantiationAwareBeanPostProcessor                 extends BeanPostProcessor
>    └─ SmartInstantiationAwareBeanPostProcessor          extends InstantiationAwareBeanPostProcessor
>       └─ AutowiredAnnotationBeanPostProcessor           implements SmartInstantiationAwareBeanPostProcessor
> ```
>
> 所以 `AutowiredAnnotationBeanPostProcessor` 同时具备上面这些接口身份，也就能被放进 `InstantiationAwareBeanPostProcessor` 这一类处理器集合里。
>
> 源码里用 `InstantiationAwareBeanPostProcessor` 这个接口类型遍历一组处理器：
>
> ```java
> for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
>     bp.postProcessProperties(pvs, bean, beanName);
> }
> ```
>
> 但集合里的真实对象可以是 `AutowiredAnnotationBeanPostProcessor`。
> 调用接口方法时，会动态派发到它自己的 `postProcessProperties(...)` 实现。
>
> 所以不是“遍历一个东西之后突然变成另一个东西”，而是“用父接口类型遍历，实际对象是某个具体实现类”。

下面这个例子是最简单的场景：用 `@Autowired` 注入一个普通字段对象。其他子类也可以按类似方式沿着生命周期阶段去看。

我们看看 AutowiredAnnotationBeanPostProcessor 类，当然也是省略大部分代码：

```java
// 这个类可以看见当我们创建AutowiredAnnotationBeanPostProcessor对象的时候完成了一个工作就是给
// autowiredAnnotationTypes赋值,这个操作有点超前，后面根据这个判断要注入的类中是否有如下的注解
public AutowiredAnnotationBeanPostProcessor() {
    this.autowiredAnnotationTypes.add(Autowired.class);
    this.autowiredAnnotationTypes.add(Value.class);
    try {
        this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
                                          ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
        logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
    }
    catch (ClassNotFoundException ex) {
        // JSR-330 API not available - simply skip.
    }
}
// 从InstantiationAwareBeanPostProcessors继承而来
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    // 寻找注入的元数据，其中它有注解扫描，和类属性信息的填充
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
       // 把数据注入到当前的bean，由于需要分析的过程太多就略过怎么实现的
        metadata.inject(bean, beanName, pvs);
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

这里不要理解成“所有 BeanPostProcessor 都是在初始化后修改实例”。

更准确地说：

```text
狭义 BeanPostProcessor：
  主要在 initializeBean() 的初始化前后执行。

InstantiationAwareBeanPostProcessor：
  是 BeanPostProcessor 的子接口，可以在 populateBean() 这类更早阶段参与属性填充。

AutowiredAnnotationBeanPostProcessor：
  通过 postProcessProperties(...) 参与 @Autowired / @Value 的字段或方法注入。
```

所以我们在普通 bean 的属性填充阶段就可以看见它的身影。`AbstractAutowireCapableBeanFactory` 中的 `populateBean()` 就是给 bean 属性填充值，同样我们省略大部分代码：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // true 因为我们有InstantiationAwareBeanPostProcessors的实现子类
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            // 主要方法
            PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                if (filteredPds == null) {
                    filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                }
                pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    return;
                }
            }
            pvs = pvsToUse;
        }
    }
}
```

至此当前的 bean 就实现了@Autowired 的字段注入，整个过程看似简单，但却有诸多细节。
