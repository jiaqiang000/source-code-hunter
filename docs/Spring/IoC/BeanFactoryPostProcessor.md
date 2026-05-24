# BeanFactoryPostProcessor 源码分析

BeanFactoryPostProcessor 是当 BeanDefinition 读取完元数据（也就是从任意资源中定义的 bean 数据）后还未实例化之前可以进行修改

抄录并翻译官方的语句

> `BeanFactoryPostProcessor` 操作 bean 的元数据配置. 也就是说,Spring IoC 容器允许 `BeanFactoryPostProcessor` 读取配置元数据, 并可能在容器实例化除 `BeanFactoryPostProcessor` 实例之外的任何 bean _之前_ 更改它

tip:

> 在 `BeanFactoryPostProcessor` (例如使用 `BeanFactory.getBean()`) 中使用这些 bean 的实例虽然在技术上是可行的,但这么来做会将 bean 过早实例化, 这违反了标准的容器生命周期. 同时也会引发一些副作用,例如绕过 bean 的后置处理。

```java
public interface BeanFactoryPostProcessor {

	/**
	 *通过ConfigurableListableBeanFactory这个可配置的BeanFactory对我们的bean原数据进行修改
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

## 先建立边界：它不是在创建普通 Bean

> [!note] 先把位置放清楚
> `BeanFactoryPostProcessor` 发生在普通 Bean 创建之前。
> 它处理的主要对象不是已经 new 出来的业务对象，而是 `BeanDefinition`、`BeanFactory` 里的配置和能力。

这篇文章要回答的是：

```text
1、2、3 篇已经把 BeanDefinition 放进 BeanFactory 了。
4、依赖注入(DI) 还没有开始真正创建普通 Bean。

那中间的 invokeBeanFactoryPostProcessors(beanFactory) 在干嘛？
```

它的位置是：

```text
ApplicationContext.refresh()
  -> obtainFreshBeanFactory()
     -> 定位资源：[[1、BeanDefinition的资源定位过程]]
     -> 解析 BeanDefinition：[[2、将bean解析封装成BeanDefinition]]
     -> 注册 BeanDefinition：[[3、将BeanDefinition注册进IoC容器]]
  -> prepareBeanFactory(beanFactory)
  -> postProcessBeanFactory(beanFactory)
  -> invokeBeanFactoryPostProcessors(beanFactory)
     -> 本文：执行 BeanFactoryPostProcessor / BeanDefinitionRegistryPostProcessor
  -> registerBeanPostProcessors(beanFactory)
     -> [[BeanPostProcessor]]
  -> finishBeanFactoryInitialization(beanFactory)
     -> [[4、依赖注入(DI)]]
```

所以这篇不是突然插进来的孤立知识。

它接在 `BeanDefinition` 注册之后，发生在普通 Bean 创建之前。

> [!note] 缩写先说明
> `BFPP` = `BeanFactoryPostProcessor`。
> 
> `BDRPP` = `BeanDefinitionRegistryPostProcessor`。
> 
> `BDRPP` 是 `BFPP` 的子接口。它多了一个更早执行的方法：`postProcessBeanDefinitionRegistry(registry)`。
> 所以它不仅能像普通 `BFPP` 一样处理 `BeanFactory`，还可以先直接操作 `BeanDefinitionRegistry`，继续注册或修改 `BeanDefinition`。

## 全局导图：从 refresh 到 BFPP 生效

```text
[00] AbstractApplicationContext::refresh()
     作用：容器启动总流程
     │
     ├─ [00.1] obtainFreshBeanFactory()
     │        作用：加载、解析、注册 BeanDefinition
     │        结果：DefaultListableBeanFactory 里已经有 beanDefinitionMap
     │
     ├─ [00.2] invokeBeanFactoryPostProcessors(beanFactory)
     │        方法定义在：AbstractApplicationContext
     │        作用：进入 BFPP / BDRPP 执行阶段
     │        缩写：BFPP = BeanFactoryPostProcessor；BDRPP = BeanDefinitionRegistryPostProcessor
     │        │
     │        └─ [01] PostProcessorRegistrationDelegate::invokeBeanFactoryPostProcessors(...)
     │           作用：真正按顺序找出并执行后处理器
     │           │
     │           ├─ [01.1] 先处理 BeanDefinitionRegistryPostProcessor
     │           │        关系：BDRPP 是 BFPP 的子接口
     │           │        作用：可以继续注册或修改 BeanDefinition
     │           │        顺序：PriorityOrdered -> Ordered -> 其他
     │           │
     │           ├─ [01.2] 再调用这些 BDRPP 的 postProcessBeanFactory(...)
     │           │        关系：BDRPP 也继承了 BFPP，所以还有一次 BeanFactory 级别回调
     │           │
     │           ├─ [01.3] 再处理普通 BeanFactoryPostProcessor
     │           │        作用：修改 BeanDefinition 元数据，或给 BeanFactory 增加能力
     │           │        顺序：PriorityOrdered -> Ordered -> 其他
     │           │
     │           └─ [01.4] clearMetadataCache()
     │                    作用：BFPP 可能改过元数据，所以清理合并 BeanDefinition 缓存
     │
     ├─ [00.3] registerBeanPostProcessors(beanFactory)
     │        作用：注册 BPP，后面 Bean 创建时才参与实例处理
     │
     └─ [00.4] finishBeanFactoryInitialization(beanFactory)
              作用：开始普通 Bean 创建
              对应：[[4、依赖注入(DI)]]
```

一句话：

```text
BeanFactoryPostProcessor 是在“BeanDefinition 已经有了，但普通 Bean 还没创建”时，
提前改配方或改工厂能力。
```

## 01 PostProcessorRegistrationDelegate：执行 BFPP / BDRPP 的总流程

ApplicationContext 的 refresh() 中的 invokeBeanFactoryPostProcessors 方法就开始创建我们的 BFPP(BeanFactoryPostProcessor)了

具体执行方法 invokeBeanFactoryPostProcessors，虽然一百多行代码，其实只需要特别了解的地方就几处。

```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    /**
     * ！！！！！！！！！！！！！
     * 阅读标注：这里对应导图 01，PostProcessorRegistrationDelegate 执行 BFPP / BDRPP 的总入口。
     * 关键边界：这里不是创建普通业务 Bean，而是创建并执行后处理器 Bean。
     * 普通 Bean 要留到 finishBeanFactoryInitialization(beanFactory) 之后再创建。
     * ！！！！！！！！！！！！！
     */
    Set<String> processedBeans = new HashSet<>();

    // 由于我们的beanFactory是DefaultListableBeanFactory实例是BeanDefinitionRegistry的子类所以可以进来
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                    (BeanDefinitionRegistryPostProcessor) postProcessor;
                /**
                 * ！！！！！！！！！！！！！
                 * 阅读标注：这里对应导图 01.1 的手动注册 BDRPP 分支。
                 * postProcessBeanDefinitionRegistry(registry) 可以直接操作 BeanDefinitionRegistry。
                 * 所以它能继续注册新的 BeanDefinition，或者修改已有 BeanDefinition。
                 * ！！！！！！！！！！！！！
                 */
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }
        // BeanDefinitionRegistryPostProcessor是BFPP的子类但是比BFPP提前执行
        // 顺序实现PriorityOrdered接口先被执行，然后是Ordered接口，最后是什么都没实现的BeanDefinitionRegistryPostProcessor

        /**
              *都有beanFactory.getBean方法，证明BeanDefinitionRegistryPostProcessor这个bean现在已经被创建了
              */

        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                /**
                 * ！！！！！！！！！！！！！
                 * 阅读标注：这里对应导图 01.1，第一批 BDRPP：PriorityOrdered。
                 * getBean(ppName, BeanDefinitionRegistryPostProcessor.class) 会提前创建后处理器 Bean。
                 * 这是允许的，因为它们本身就是为了在普通 Bean 创建前执行。
                 * ！！！！！！！！！！！！！
                 */
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        /**
         * ！！！！！！！！！！！！！
         * 阅读标注：这里真正执行第一批 BDRPP 的 postProcessBeanDefinitionRegistry。
         * 执行完这一行后，它们已经有机会改 BeanDefinitionRegistry。
         * ！！！！！！！！！！！！！
         */
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
        currentRegistryProcessors.clear();

        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                /**
                 * ！！！！！！！！！！！！！
                 * 阅读标注：这里对应导图 01.1，第二批 BDRPP：Ordered。
                 * processedBeans 用来避免同一个后处理器重复执行。
                 * ！！！！！！！！！！！！！
                 */
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
        currentRegistryProcessors.clear();

        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    /**
                     * ！！！！！！！！！！！！！
                     * 阅读标注：这里对应导图 01.1，第三批 BDRPP：普通 BDRPP。
                     * 这里用 while 不是多余的：某个 BDRPP 执行时可能又注册新的 BDRPP，
                     * 所以 Spring 要反复查，直到没有新的 BDRPP 出现。
                     * ！！！！！！！！！！！！！
                     */
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
            currentRegistryProcessors.clear();
        }

        /**
         * ！！！！！！！！！！！！！
         * 阅读标注：这里对应导图 01.2。
         * BDRPP 是 BFPP 的子接口，所以它先执行完 postProcessBeanDefinitionRegistry，
         * 还要继续执行 postProcessBeanFactory。
         * ！！！！！！！！！！！！！
         */
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }
    // BFPP的执行顺序与上一样
    /**
        *都有beanFactory.getBean方法，证明BFPP这个bean现在已经被创建了
        */
    String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);


    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {

        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            /**
             * ！！！！！！！！！！！！！
             * 阅读标注：这里对应导图 01.3，第一批普通 BFPP：PriorityOrdered。
             * 例如有些占位符、配置类相关处理器会要求优先执行。
             * ！！！！！！！！！！！！！
             */
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }


    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    /**
     * ！！！！！！！！！！！！！
     * 阅读标注：这里开始真正执行普通 BFPP 的 postProcessBeanFactory。
     * 这一类处理器主要改 BeanDefinition 元数据，或给 BeanFactory 注册额外能力。
     * ！！！！！！！！！！！！！
     */
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);


    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    /**
     * ！！！！！！！！！！！！！
     * 阅读标注：这里对应导图 01.3，第二批普通 BFPP：Ordered。
     * ！！！！！！！！！！！！！
     */
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);


    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    /**
     * ！！！！！！！！！！！！！
     * 阅读标注：这里对应导图 01.3，第三批普通 BFPP：没有排序接口的处理器。
     * ！！！！！！！！！！！！！
     */
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);


    /**
     * ！！！！！！！！！！！！！
     * 阅读标注：这里对应导图 01.4。
     * 因为 BFPP 可能改过 BeanDefinition 元数据，所以清理合并元数据缓存。
     * 后续普通 Bean 创建时，会重新使用最新的元数据。
     * ！！！！！！！！！！！！！
     */
    beanFactory.clearMetadataCache();
}
```

到这里，主线已经很清楚了：

```text
BDRPP 先执行：
  因为它可以继续注册 / 修改 BeanDefinition。

BFPP 后执行：
  因为它主要处理 BeanFactory 或已经存在的 BeanDefinition 元数据。

普通 Bean 还没开始创建：
  后面才会进入 registerBeanPostProcessors 和 finishBeanFactoryInitialization。
```

我们可以具体分析一下 BeanFactoryPostProcessor 的子类 CustomEditorConfigurer 自定义属性编辑器来巩固一下执行流程

所谓属性编辑器是当你要自定义更改配置文件中的属性属性时，如 String 类型转为 Date 或者其他，下面的一个小例子展示如何 String 类型的属性怎么转化为 Address 属性

## 02 CustomEditorConfigurer 示例：先注册编辑器，后面属性填充时使用

> [!note] 这个例子为什么放在 BFPP 里
> `CustomEditorConfigurer` 不是在这里直接修改 `person` 这个 Bean 实例。
> 它是在普通 Bean 创建前，把“String 怎么转 Address”的编辑器注册到 `BeanFactory`。
> 后面 `populateBean()` / `applyPropertyValues()` 写属性时，才会真正使用这个编辑器。

整体链路是：

```text
[02.1] XML 中声明 CustomEditorConfigurer
       │
       └─ Spring 把它当成 BeanFactoryPostProcessor
          │
          └─ [02.2] invokeBeanFactoryPostProcessors 阶段提前创建并执行它
             │
             └─ [02.3] CustomEditorConfigurer::postProcessBeanFactory(beanFactory)
                │
                └─ beanFactory.addPropertyEditorRegistrar(MyCustomEditor)
                   │
                   └─ [02.4] 后面创建 person 时进入 populateBean / applyPropertyValues
                      │
                      └─ [02.5] BeanWrapper 类型转换时调用 MyCustomEditor
                         │
                         └─ AddressParse.setAsText("四川,成都") 得到 Address 对象
```

### 简单工程（Spring-version-5.3.18)

Person 类

```java
package cn.demo1;

import lombok.Getter;
import lombok.Setter;

@Setter
@Getter
@ToString
public class Person {
    private String name;
    private Address address;
}
```

Address 类

```java
package cn.demo1;

@Setter
@Getter
@ToString
public class Address {
    private String province;
    private String city;
}

```

AddressParse 类

```java
package cn.demo1;

import java.beans.PropertyEditorSupport;

public class AddressParse extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        final String[] vals = text.split(",");
        Address addr = new Address();
        addr.setProvince(vals[0]);
        addr.setCity(vals[1]);
        setValue(addr);
    }
}
```

MyCustomEditor 类

```java
package cn.demo1;

import org.springframework.beans.PropertyEditorRegistrar;
import org.springframework.beans.PropertyEditorRegistry;


public class MyCustomEditor implements PropertyEditorRegistrar {
    @Override
    public void registerCustomEditors(PropertyEditorRegistry registry) {
        registry.registerCustomEditor(Address.class, new AddressParse());
    }
}
```

配置文件 test1.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

    <!--    待属性编辑的bean，value代表的就是string类型-->
    <bean class="cn.demo1.Person" id="person">
        <property name="name" value="李华"/>
        <property name="address" value="四川,成都"/>
    </bean>

    <!--    注册属性编辑器-->
    <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer" id="configurer">
        <property name="propertyEditorRegistrars">
            <list>
                <bean class="cn.demo1.MyCustomEditor"/>
            </list>
        </property>
    </bean>
</beans>
```

测试类 EdT

```java
package cn.test1;

import cn.demo1.Person;
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class EdT {
    @Test
    public void test1() {
        ApplicationContext context = new ClassPathXmlApplicationContext("test1.xml");
        final Person bean = context.getBean(Person.class);
        System.out.println(bean);
    }
}

=====================测试结果

Person(name=李华, address=Address(province=四川, city=成都))
```

可以看见我们成功的将 String 类型转化为 Address 类型，让我们来看看实现流程，

- 首先实现 PropertyEditorSupport 来自定义属性编辑规则
- 其次将你的编辑规则给到 PropertyEditorRegistrar 子类里进行注册
- 最后在 Spring 中配置 CustomEditorConfigurer 类然后注入你的 PropertyEditorRegistrar 注册器

这里最容易断开的点是：

```text
MyCustomEditor 不是直接给 person.address 赋值。
CustomEditorConfigurer 也不是直接给 person.address 赋值。

它们只是提前把“Address 类型怎么从 String 转换出来”的规则注册进 BeanFactory。
真正赋值发生在后面的 applyPropertyValues / BeanWrapper.setPropertyValues。
```

让我们 debug 走一遍

如果你已经耐心看完上面的 `01 PostProcessorRegistrationDelegate：执行 BFPP / BDRPP 的总流程`，那么你应该可以知道接下来我们的步骤应该是进入 invokeBeanFactoryPostProcessors 这个方法里了

```java
private static void invokeBeanFactoryPostProcessors(
			Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

		for (BeanFactoryPostProcessor postProcessor : postProcessors) {
			StartupStep postProcessBeanFactory = beanFactory.getApplicationStartup().start("spring.context.bean-factory.post-process")
					.tag("postProcessor", postProcessor::toString);
			/**
			 * ！！！！！！！！！！！！！
			 * 阅读标注：这里对应导图 02.2。
			 * 对 CustomEditorConfigurer 来说，这里会调用它的 postProcessBeanFactory(beanFactory)。
			 * 这一步发生在普通 person Bean 创建之前。
			 * ！！！！！！！！！！！！！
			 */
			postProcessor.postProcessBeanFactory(beanFactory);
			postProcessBeanFactory.end();
		}
	}
```

很明显它执行 postProcessBeanFactory 这个方法

我们探究的 BFPP 正是 CustomEditorConfigurer，所以这个是 CustomEditorConfigurer 对 BFPP 的 postProcessBeanFactory 实现

```java
// 必然有个set方法让我们进行注入
public void setPropertyEditorRegistrars(PropertyEditorRegistrar[] propertyEditorRegistrars) {
    this.propertyEditorRegistrars = propertyEditorRegistrars;
}

@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    if (this.propertyEditorRegistrars != null) {
        for (PropertyEditorRegistrar propertyEditorRegistrar : this.propertyEditorRegistrars) {
            /**
             * ！！！！！！！！！！！！！
             * 阅读标注：这里对应导图 02.3。
             * 这里不是给某个 Bean 属性赋值，而是把 MyCustomEditor 这种注册器放进 BeanFactory。
             * 后面 BeanFactory 创建 BeanWrapper / TypeConverter 时，会把这些编辑器注册进去。
             * ！！！！！！！！！！！！！
             */
            beanFactory.addPropertyEditorRegistrar(propertyEditorRegistrar);
        }
    }
    if (this.customEditors != null) {
        /**
         * ！！！！！！！！！！！！！
         * 阅读标注：另一种注册方式。
         * propertyEditorRegistrars 是注册器方式；customEditors 是直接按类型注册 PropertyEditor 类。
         * ！！！！！！！！！！！！！
         */
        this.customEditors.forEach(beanFactory::registerCustomEditor);
    }
}
```

关于这个注册器使用要到后面填充属性的时候才会用到，

> 我其实觉得这个有点瑕疵，因为 BFPP 作用影响应该是当 Spring 还未创建 bean 的时候，可以用 BFPP 进行修改操作，可是这个属性编辑却影响了 bean 创建过后的修改操作，那么它就替代了 BPP（BeanPostProcessor)的作用发挥了。（以上仅仅代表个人的观点，有可能是我想错了）

> [!warning] 这里不要理解成替代 BeanPostProcessor
> `CustomEditorConfigurer` 的动作仍然发生在普通 Bean 创建之前。
> 它只是把类型转换能力提前注册到 `BeanFactory`。
> 后面 `person` 创建时用到这个能力，不代表 BFPP 在 Bean 创建后修改了 `person`。
>
> 所以它和 `BeanPostProcessor` 的区别仍然是：
> - `BeanFactoryPostProcessor`：提前改 BeanFactory / BeanDefinition / 工厂能力。
> - `BeanPostProcessor`：普通 Bean 实例创建过程中，围绕实例对象做处理。

当我们 debug 到 AbstractAutowireCapableBeanFactory 的 populateBean 这个方法填充 bean 的属性的时候，

让我们看看它的方法，其中我省略了大部分无关代码

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    /**
     * ！！！！！！！！！！！！！
     * 阅读标注：这里对应导图 02.4。
     * 现在才进入普通 Bean 的属性填充阶段。
     * 前面的 CustomEditorConfigurer 已经把 PropertyEditorRegistrar 注册进 BeanFactory。
     * ！！！！！！！！！！！！！
     */
    // 这个是如果你配置的bean中有属性值的话
    // 也就是如下的配置，那么pvs不会为空的
    /**
    <bean class="cn.demo1.Person" id="person">
        <property name="name" value="李华"/>
        <property name="address" value="四川,成都"/>
    </bean>
    */
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    if (pvs != null) {
        /**
         * ！！！！！！！！！！！！！
         * 阅读标注：这里会进入 [[4、依赖注入(DI)]] 中的 applyPropertyValues 主线。
         * 本例的 address="四川,成都" 会在后续类型转换时用到前面注册的 AddressParse。
         * ！！！！！！！！！！！！！
         */
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

让我们继续看看 applyPropertyValues 这个方法，无关的代码我也给省略了

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    // PropertyValues接口的默认实现。允许对属性进行简单操作，并提供构造函数以支持从 Map 进行深度复制和构造。
    MutablePropertyValues mpvs = null;
    List<PropertyValue> original;
    // 可以进去
    if (pvs instanceof MutablePropertyValues) {
        mpvs = (MutablePropertyValues) pvs;
    // 默认为false，即我们需要类型转换
        if (mpvs.isConverted()) {
            // Shortcut: use the pre-converted values as-is.
            try {
                bw.setPropertyValues(mpvs);
                return;
            }
            catch (BeansException ex) {
                throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Error setting property values", ex);
            }
        }
      // 把bean的属性以列表的形式展示出来
        original = mpvs.getPropertyValueList();
    }
    else {
        original = Arrays.asList(pvs.getPropertyValues());
    }
    // 默认为空
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        /**
         * ！！！！！！！！！！！！！
         * 阅读标注：这里使用 BeanWrapper 做默认 TypeConverter。
         * BeanWrapper 在创建 / 初始化时会拿到 BeanFactory 中注册的自定义编辑器。
         * ！！！！！！！！！！！！！
         */
        converter = bw;
    }
    // 就一个组合类，帮助更好的bean的属性的解析
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

    // 深拷贝
    List<PropertyValue> deepCopy = new ArrayList<>(original.size());
    boolean resolveNecessary = false;
    for (PropertyValue pv : original) {
        if (pv.isConverted()) {
            deepCopy.add(pv);
        }
        else {
            // 获取bean的属性名字
            String propertyName = pv.getName();
            //获取bean属性值的包装对象
            Object originalValue = pv.getValue();
            // 自动装配的事情
            if (originalValue == AutowiredPropertyMarker.INSTANCE) {
                Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
                if (writeMethod == null) {
                    throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
                }
                originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
            }
            // 把bean的属性值从包装类中分离出来
            /**
             * ！！！！！！！！！！！！！
             * 阅读标注：这里主要负责把 BeanDefinition 里的配置值解析成运行期值。
             * 对 ref 来说，这里可能递归 getBean(refName)。
             * 对本例 address="四川,成都" 这种字面量来说，真正的 String -> Address 转换在 convertForProperty。
             * ！！！！！！！！！！！！！
             */
            Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
            Object convertedValue = resolvedValue;
            // 一般为true
            boolean convertible = bw.isWritableProperty(propertyName) &&
                !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
            if (convertible) {
                /**
                 * ！！！！！！！！！！！！！
                 * 阅读标注：这里对应导图 02.5。
                 * 本例的重点在这里：把 String "四川,成都" 转成 Address。
                 * 它会继续进入 BeanWrapperImpl / TypeConverterDelegate，最终调用 AddressParse.setAsText。
                 * ！！！！！！！！！！！！！
                 */
                convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
            }
	}
```

继续追踪

```java
@Nullable
private Object convertForProperty(
    @Nullable Object value, String propertyName, BeanWrapper bw, TypeConverter converter) {
    // BeanWrapperImpl是继承TypeConverter的
    if (converter instanceof BeanWrapperImpl) {
        // 所以执行下面的方法
        /**
         * ！！！！！！！！！！！！！
         * 阅读标注：这里继续走 BeanWrapperImpl 的类型转换能力。
         * 前面 CustomEditorConfigurer 注册进 BeanFactory 的 PropertyEditor，
         * 会在 BeanWrapper / TypeConverter 体系里被使用。
         * ！！！！！！！！！！！！！
         */
        return ((BeanWrapperImpl) converter).convertForProperty(value, propertyName);
    }
    else {
        PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
        MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
        return converter.convertIfNecessary(value, pd.getPropertyType(), methodParam);
    }
}
```

```java
@Nullable
public Object convertForProperty(@Nullable Object value, String propertyName) throws TypeMismatchException {
    CachedIntrospectionResults cachedIntrospectionResults = getCachedIntrospectionResults();
    PropertyDescriptor pd = cachedIntrospectionResults.getPropertyDescriptor(propertyName);
    if (pd == null) {
        throw new InvalidPropertyException(getRootClass(), getNestedPath() + propertyName,
                                           "No property '" + propertyName + "' found");
    }
    TypeDescriptor td = cachedIntrospectionResults.getTypeDescriptor(pd);
    if (td == null) {
        td = cachedIntrospectionResults.addTypeDescriptor(pd, new TypeDescriptor(property(pd)));
    }
    // 上面的工作不用管，全是一些前戏工作，这个才是主题，至此我们的流程就到这里结束吧
    // 后面的流程太多了，大部分都是处理细节，你只需要知道大概的脉络就行，就是最终它肯定会
    // 走到AddressParse这个核心处理
    /**
     * ！！！！！！！！！！！！！
     * 阅读标注：这里继续做目标属性类型判断。
     * 对 person.address 来说，目标类型是 Address，原始值是 String。
     * 如果注册过 Address 对应的 PropertyEditor，就可以调用 AddressParse.setAsText 完成转换。
     * ！！！！！！！！！！！！！
     */
    return convertForProperty(propertyName, null, value, td);
}
```

## 03 这一篇最终要记住什么

这篇的核心不是 `AddressParse` 本身，而是这个前后连接：

```text
BeanDefinition 已经注册
  -> invokeBeanFactoryPostProcessors(beanFactory)
    -> 提前创建并执行 BFPP / BDRPP
      -> CustomEditorConfigurer 把 PropertyEditorRegistrar 注册进 BeanFactory
        -> 后面普通 Bean 创建
          -> populateBean()
            -> applyPropertyValues()
              -> convertForProperty()
                -> 使用前面注册的 AddressParse 做类型转换
```

所以它和导图的对应关系是：

```text
[[0、Spring IoC 全局地图]]
  -> invokeBeanFactoryPostProcessors(beanFactory)
     -> 本文

[[4、依赖注入(DI)]]
  -> populateBean()
     -> applyPropertyValues()
        -> 本文 CustomEditorConfigurer 示例的后半段会在这里体现效果
```

你可以自己可以尝试 debug 一下，看别人实践真的不如自己动手实践一下，Spring 的包装类实属太多，但是可以抓住核心流程进行 debug。
