# BeanPostProcessor 源码分析

BeanPostProcessor 接口也叫 Bean 后置处理器，作用是在 Bean 对象实例化和依赖注入完成后，在配置文件 bean 的 init-method(初始化方法)或者 InitializingBean 的 afterPropertiesSet 的前后添加我们自己的处理逻辑。注意是 Bean 实例化完毕后及依赖注入完成后触发的，接口的源码如下。

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

> [!note] 先把名字拆开
> 狭义的 `BeanPostProcessor` 就是上面这个接口，核心只有初始化前后两个方法。
> 但是 Spring 里常说的 BeanPostProcessor，很多时候指的是一整套后处理器体系。
> 这套体系里有很多子接口，所以它们能插入 Bean 创建的不同阶段。

狭义 `BeanPostProcessor` 只对应：

```text
initializeBean()
  -> postProcessBeforeInitialization(...)
  -> invokeInitMethods(...)
  -> postProcessAfterInitialization(...)
```

也就是说，如果一个类只是：

```java
public class MyProcessor implements BeanPostProcessor {
}
```

那它稳定参与的是初始化前后：

```text
postProcessBeforeInitialization(...)
postProcessAfterInitialization(...)
```

它不会自动进入 `populateBean()` 的 `postProcessProperties(...)`，也不会自动参与构造器选择。

如果要进入更早的阶段，需要实现更具体的子接口。常见关系可以先这样记：

```text
BeanPostProcessor
  -> postProcessBeforeInitialization(...)
  -> postProcessAfterInitialization(...)

InstantiationAwareBeanPostProcessor
  extends BeanPostProcessor
  -> postProcessAfterInstantiation(...)
  -> postProcessProperties(...) / postProcessPropertyValues(...)

SmartInstantiationAwareBeanPostProcessor
  extends InstantiationAwareBeanPostProcessor
  -> determineCandidateConstructors(...)
  -> getEarlyBeanReference(...)

MergedBeanDefinitionPostProcessor
  extends BeanPostProcessor
  -> postProcessMergedBeanDefinition(...)
```

所以同样叫 `BeanPostProcessor`，位置可能完全不同：

```text
createBeanInstance()
  -> SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors(...)

doCreateBean()
  -> MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition(...)

populateBean()
  -> InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(...)
  -> InstantiationAwareBeanPostProcessor.postProcessProperties(...)

initializeBean()
  -> BeanPostProcessor.postProcessBeforeInitialization(...)
  -> BeanPostProcessor.postProcessAfterInitialization(...)
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

在此我举例一个典型的例子 AutowiredAnnotationBeanPostProcessor，是 BeanPostProcessor 的一个子类，是@Autowired 和@Value 的具体实现，其他的子类你也可以按如下的流程自行走一边，注意我的例子只是一个最为简单的例子，也就是用@Autowired 注入了一个普通的字段对象

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

BeanPostProcessor 的职责是在 bean 初始化后进行实例的更改，所以我们在普通 bean 实例化的时候就可以看见它的身影 AbstractAutowireCapableBeanFactory 中的 populateBean 就是给 bean 属性填充值，同样我们省略大部分代码：

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
