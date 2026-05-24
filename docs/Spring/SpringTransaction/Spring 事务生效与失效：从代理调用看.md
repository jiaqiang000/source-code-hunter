# Spring 事务生效与失效：从代理调用看

## 先建立边界：@Transactional 不是自己生效

> [!note]
> `@Transactional` 本身只是事务元数据。
> 
> 真正让事务生效的是代理对象上的 `TransactionInterceptor`。方法调用必须先进入代理对象，才有机会进入事务拦截器。

这篇不是重新讲完整事务源码，而是作为排查地图使用：当你看到 `@Transactional` 好像没生效时，先判断这次调用有没有经过代理，再判断有没有进入事务拦截器，最后再看回滚规则和事务管理器。

相关笔记：

- [[0、Spring AOP 全局地图]]
- [[AOP源码实现及分析]]
- [[JDK动态代理的实现原理解析]]
- [[Spring声明式事务处理]]
- [[Spring-transaction]]

## 全局导图：一次事务方法调用怎么生效

- [00] 开启事务能力
  - 常见入口：
    - `@EnableTransactionManagement`
    - Spring Boot 自动配置
  - 结果：
    - 注册事务相关基础设施
    - 注册事务 Advisor
    - 注册 `TransactionInterceptor`
    - 注册或复用 AutoProxyCreator
  - 对应源码笔记：
    - [[Spring-transaction#EnableTransactionManagement]]
    - [[Spring-transaction#ProxyTransactionManagementConfiguration]]

- [01] Bean 创建阶段
  - AutoProxyCreator 检查当前 Bean 是否命中事务 Advisor
  - 如果某些方法匹配 `@Transactional`
    - raw bean 会被包装成 proxy bean
  - 对应 AOP 笔记：
    - [[0、Spring AOP 全局地图#完整生命周期地图：从开启 AOP 到方法调用]]
    - [[AOP源码实现及分析#全局导图：从代理创建到拦截器链调用]]

- [02] 外部代码调用事务方法
  - 外部拿到的是 Spring 容器返回的代理对象
  - 调用：
    - `proxy.method()`
      - 进入 JDK / CGLIB 代理回调
      - 进入拦截器链
      - 进入 `TransactionInterceptor.invoke(...)`
  - 对应 AOP 笔记：
    - [[JDK动态代理的实现原理解析#全局导图：从 target 到 $Proxy0.invoke]]
    - [[AOP源码实现及分析#3 Spring AOP 拦截器调用的实现]]

- [03] TransactionInterceptor 包住真实方法调用
  - `TransactionInterceptor.invoke(...)`
    - 读取事务属性
    - 找到 `PlatformTransactionManager`
    - 必要时开启事务
    - 调用真实目标方法
    - 正常返回：提交事务
    - 抛出需要回滚的异常：回滚事务
  - 对应事务笔记：
    - [[Spring声明式事务处理#2.3 事务处理拦截器的设计与实现]]
    - [[Spring-transaction#TransactionInterceptor]]

## 什么才叫走代理

走代理的关键不是“这个类上有没有 `@Service`”，而是“这次方法调用的接收者是不是 Spring 返回的代理对象”。

走代理的典型情况：

```java
@RestController
public class OrderController {
    @Autowired
    private OrderService orderService;

    public void test() {
        orderService.b();
    }
}
```

如果 `orderService` 是 Spring 注入进来的代理对象，那么调用链是：

- `controller.orderService.b()`
  - `proxy.b()`
    - `TransactionInterceptor.invoke(...)`
      - 开启事务
      - `target.b()`
      - 提交 / 回滚

这里的重点是：外部 Bean 拿到的是 Spring 容器暴露出来的对象。这个对象可能已经不是原始 `OrderService`，而是代理对象。

## 为什么 this.b() 不是走代理

同类内部调用：

```java
@Service
public class OrderService {
    public void a() {
        this.b();
    }

    @Transactional
    public void b() {
    }
}
```

如果外部调用 `orderService.a()`，第一跳可能是代理：

- 外部 Bean
  - `proxy.a()`
    - `target.a()`
      - `this.b()`
        - `target.b()`

问题就在 `this.b()` 这一跳。

进入 `target.a()` 以后，`this` 是原始对象 `target`，不是外面的 `proxy`。所以 `this.b()` 是普通 Java 方法调用，不会重新进入代理对象，也就不会进入 `TransactionInterceptor`。

> [!warning]
> 自调用不是“注解消失了”，而是调用路径绕过了代理。
> 
> `@Transactional` 仍然在 `b()` 上，但这次调用没有经过代理对象，所以事务拦截器没有机会执行。

还有一个容易混的点：

- 如果 `a()` 本身有事务，那么 `this.b()` 会运行在 `a()` 已经开启的事务里。
- 但 `b()` 自己声明的事务属性不会单独生效。
- 比如 `b()` 写了 `REQUIRES_NEW`，自调用时通常不会新开事务，因为没有经过代理重新进入事务拦截器。

## private / final / static 为什么影响事务

这些情况本质都是：代理对象没有办法拦住这次方法调用。

### 先分清：失败可能发生在三个阶段

先看总图：

- [01] 先决定用哪种代理
  - JDK 动态代理
  - CGLIB 代理

- [02] 代理能不能创建出来
  - `final class` 主要卡在这里，针对 CGLIB

- [03] 这个方法能不能被事务 Advisor 匹配
  - `private` / 非 public 方法常卡在这里

- [04] 调用这个方法时能不能进入代理回调
  - JDK：`invoke(...)`
  - CGLIB：`intercept(...)`

- [05] 能不能进入 `TransactionInterceptor`
  - 进不去就表现为 `@Transactional` 失效

### JDK 代理和 CGLIB 代理不是同一种限制

#### final class 发生在哪里

如果走 CGLIB，Spring 要生成目标类的子类：

```text
class XxxService$$CGLIB extends XxxService
```

对应源码位置是 [[AOP源码实现及分析#2.6 CGLIB 生成 AopProxy 代理对象]] 里设置 superclass 和创建代理对象的地方。

如果目标类是 `final class`，CGLIB 不能继承它，代理创建可能直接失败。这个不是“调用时失效”，而是“代理创建阶段就出问题”。

但 JDK 代理不一样。JDK 代理不继承目标类，它生成的是接口代理：

```text
$Proxy0 implements XxxInterface
```

所以如果 `final class OrderService implements OrderServiceApi`，并且 Spring 走 JDK 代理，那么 `final class` 本身不是问题。

#### final method 发生在哪里

CGLIB 会检查 final 方法，对应 [[AOP源码实现及分析#2.6 CGLIB 生成 AopProxy 代理对象]] 中的 class 校验和代理创建过程。

源码里对 final 方法会提示：

```text
Final method ... cannot get proxied via CGLIB
```

原因是 CGLIB 靠子类重写方法：

```text
proxy.b()
  -> CGLIB 子类重写的 b()
    -> intercept(...)
```

但 `final method` 不能被子类重写，所以这个方法不会进入 `intercept(...)`。你 md 里对应的是 [[AOP源码实现及分析#3.2 CglibAopProxy 的 intercept() 拦截]]。

JDK 代理这里不一样：如果这个 final 方法是接口方法的实现，那么外部调用的是 `$Proxy0.interfaceMethod()`，仍然可以进入 JDK 的 `InvocationHandler.invoke(...)`。所以 `final method` 对 CGLIB 是硬伤，对 JDK 接口代理不一定是问题。

#### private method 发生在哪里

这里有两层问题。

第一层：事务属性匹配阶段就可能被过滤。

Spring 事务默认是 public 方法才支持事务语义。核心逻辑可以对应到 [[Spring-transaction#TransactionInterceptor]] 背后的 `TransactionAttributeSource` 读取过程：

```java
if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
    return null;
}
```

`return null` 的意思是：这个方法没有事务属性，事务 Advisor 不匹配它。

第二层：代理调用阶段也拦不住。

CGLIB 子类看不到 private 方法，不能重写。JDK 代理也只能代理接口实例方法，private 方法不会在接口代理调用链里。

所以 private 方法不是单点失败，而是：

```text
事务属性可能不匹配
  + 代理也没有办法拦截 private 方法调用
```

#### static method 发生在哪里

`static` 方法不是对象实例方法，不走：

```text
proxy.method()
```

这种对象虚方法调用链。

CGLIB 靠子类重写实例方法，static 方法不能被重写。JDK 代理靠接口实例方法回调，static 方法也不是这种调用。所以它本质上不进入 Spring AOP 的代理方法调用链。

最重要的区别：

```text
CGLIB：
  靠继承目标类 + 重写方法
  所以 final class / final method / private method / static method 都很敏感

JDK 动态代理：
  靠生成接口代理
  不继承目标类
  所以 final class 不一定有问题
  final method 如果是接口方法实现，也不一定有问题
  但不在接口上的方法，JDK 代理无法暴露和拦截
```

所以不是一句“这些都会导致失效”。更准确是：

```text
final class：
  CGLIB 代理创建阶段失败
  JDK 接口代理不一定受影响

final method：
  CGLIB 调用阶段进不了 intercept
  JDK 接口代理不一定受影响

private method：
  事务属性匹配阶段通常就不认
  代理调用阶段也拦不住

static method：
  不属于对象代理调用链
  Spring AOP 代理拦不住
```

### JDK 动态代理只代理接口方法

JDK 动态代理的直觉可以看 [[JDK动态代理的实现原理解析]]。

核心是：

- 外部拿到的是 `$Proxy0`
- 调用 `$Proxy0.method()`
- `$Proxy0` 回调 `InvocationHandler.invoke(...)`

所以 JDK 动态代理天然围绕接口方法工作。如果方法不在接口上，或者调用没有经过接口代理对象，就不适合用 JDK 动态代理拦截。

### CGLIB 代理依赖子类重写方法

CGLIB 的直觉可以看 [[AOP源码实现及分析#3.2 CglibAopProxy 的 intercept() 拦截]]。

核心是：

- Spring 生成目标类的子类代理
- 外部调用 `proxy.method()`
- 子类重写的方法先进入 `intercept(...)`
- 再决定是否调用目标方法和增强链

所以这些情况会影响代理：

- `final class`
  - 不能被继承
  - CGLIB 没法生成可用的子类代理
- `final method`
  - 子类不能重写
  - CGLIB 拦不住这个方法
- `private method`
  - private 方法对子类不可见
  - 子类不能重写
  - CGLIB 拦不住
- `static method`
  - 属于类，不属于对象实例的虚方法调用
  - 代理对象拦不住

实际开发里先按这个规则记：

> [!tip]
> `@Transactional` 尽量放在 Spring Bean 的 `public` 方法上，并且由其他 Bean 通过注入的代理对象来调用。

## 事务进去了但没有回滚

有时事务不是没有生效，而是已经进入了 `TransactionInterceptor`，只是最后没有触发回滚。

### 异常被吃掉

```java
@Transactional
public void b() {
    try {
        dao.insert();
        int x = 1 / 0;
    } catch (Exception e) {
        // 异常被吃掉
    }
}
```

`TransactionInterceptor` 看不到异常，会认为方法正常结束，于是提交事务。

### 抛的是受检异常

```java
@Transactional
public void b() throws Exception {
    throw new Exception();
}
```

默认情况下，Spring 事务主要回滚 `RuntimeException` 和 `Error`。如果希望受检异常也回滚，需要显式配置：

```java
@Transactional(rollbackFor = Exception.class)
public void b() throws Exception {
    throw new Exception();
}
```

## 多数据源和事务管理器不一致

多数据源场景里，还要看 `PlatformTransactionManager` 是否和实际操作的数据源一致。

典型问题：

- `@Transactional` 使用的是 A 数据源的事务管理器
- 实际 DAO 操作的是 B 数据源
- 结果看起来像事务没生效

排查时要确认：

- 当前方法命中的事务管理器是谁
- 当前数据库操作用的是哪个数据源
- 两者是否匹配

## 排查事务失效的顺序

- [00] 事务能力有没有开启
  - 是否有 `@EnableTransactionManagement`
  - Spring Boot 是否自动配置了事务能力

- [01] 当前对象是不是 Spring Bean
  - 不能是自己 `new` 出来的对象

- [02] 当前调用有没有经过代理对象
  - 外部 Bean 注入调用：通常可以
  - `this.xxx()` 自调用：通常不行

- [03] 方法是否适合被代理
  - 优先放在 `public` 方法上
  - 避免 `private` / `final` / `static`
  - 注意 final 类和 CGLIB 代理冲突

- [04] 是否进入了 `TransactionInterceptor`
  - 进入了才说明事务拦截器开始工作
  - 对应：[[Spring声明式事务处理#2.3 事务处理拦截器的设计与实现]]

- [05] 异常是否传到了事务拦截器
  - 异常被 catch 吃掉就不会触发回滚

- [06] rollback 规则是否匹配
  - 默认主要回滚 `RuntimeException` 和 `Error`
  - 受检异常需要 `rollbackFor`

- [07] 事务管理器和数据源是否一致
  - 多数据源项目尤其要查这一点

## 这一篇最终要记住什么

- `@Transactional` 生效的关键是进入代理对象。
- 进入代理对象后，才有机会进入 `TransactionInterceptor`。
- `this.b()` 是原始对象内部调用，不会重新进入代理。
- `private` / `final` / `static` 影响的是代理能不能拦住这个方法。
- 事务没回滚不一定是没进事务，也可能是异常被吃掉、rollback 规则不匹配、事务管理器不对。
