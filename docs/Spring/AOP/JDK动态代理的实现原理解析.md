# JDK动态代理的实现原理解析

## 先建立边界：这篇不是直接讲 Spring AOP

> [!note]
> 这篇先解决一个前置问题：JDK 动态代理生成出来的代理对象，为什么能接住方法调用。
> 
> 它还没有进入 Spring AOP 的 `Advisor`、`Pointcut`、`ProxyFactory`、`JdkDynamicAopProxy` 主线，但后面理解 Spring AOP 的 JDK 代理时，会用到这里的直觉。

## 全局导图：从 target 到 $Proxy0.invoke

- [00] 准备真实业务对象
  - `TargetObject target = new TargetObject()`
    - 关系：`TargetObject implements MyInterface`
    - 作用：它是真正干活的原始对象
    - 方法：`play()`

- [01] 准备回调处理器
  - `ProxyFactory proxyFactory = new ProxyFactory()`
    - 关系：`ProxyFactory implements InvocationHandler`
    - 作用：它不是业务对象，而是代理对象的方法回调处理器
    - 内部持有：
      - `private Object target`
        - 指向 [00] 里的原始对象
    - 关键方法：
      - `getInstanse(target)`
        - 保存 `this.target = target`
        - 调用 [02] 生成代理对象
      - `invoke(proxy, method, args)`
        - 后面 [05] 代理对象被调用时会回到这里

- [02] 生成代理对象
  - `proxyFactory.getInstanse(target)`
    - 内部调用：
      - `Proxy.newProxyInstance(...)`
        - 输入：
          - `target.getClass().getClassLoader()`
            - 类加载器：用来在运行时生成代理类
          - `target.getClass().getInterfaces()`
            - 接口列表：决定代理对象能被强转成哪些接口
          - `this`
            - 回调处理器：这里就是 [01] 的 `proxyFactory`
        - 输出：
          - `Object proxy`
            - 真实类型：运行时生成的 `$Proxy0`
            - 实现接口：`MyInterface`
            - 内部持有：`InvocationHandler h = proxyFactory`

- [03] 业务代码拿到代理对象
  - `MyInterface mi = (MyInterface) proxyFactory.getInstanse(target)`
    - 表面类型：`MyInterface`
    - 真实对象：`$Proxy0`
    - 关键点：
      - 外部代码以为自己拿到的是一个 `MyInterface`
      - 实际拿到的是一个实现了 `MyInterface` 的代理对象

- [04] 调用代理对象的方法
  - `mi.play()`
    - 实际进入：
      - `$Proxy0.play()`
        - `$Proxy0 extends Proxy implements MyInterface`
        - `Proxy` 父类中有字段：`InvocationHandler h`
        - `h` 指向 [01] 的 `proxyFactory`
        - 所以 `$Proxy0.play()` 内部不是直接调用 `target.play()`
        - 而是调用：
          - `super.h.invoke(this, $Proxy0.m3, null)`
            - `this`：当前代理对象 `$Proxy0`
            - `$Proxy0.m3`：`MyInterface.play()` 对应的 `Method`
            - `null`：`play()` 没有参数

- [05] 回到 `InvocationHandler.invoke(...)`
  - `ProxyFactory.invoke(proxy, method, args)`
    - `proxy`：当前代理对象 `$Proxy0`
    - `method`：`MyInterface.play()` 对应的 `Method`
    - `args`：方法参数，这里是 `null`
    - 执行顺序：
      - 打印前置增强
      - `method.invoke(target, args)`
        - 这里的 `target` 指向 [00] 的 `TargetObject`
        - 真正执行：`TargetObject.play()`
      - 打印后置增强

## 这段代码具体在干嘛

这篇代码其实是在演示一件事：调用代理对象的方法时，为什么能自动绕到增强逻辑里。

如果没有代理，调用链很简单：

- `target.play()`
  - 直接进入 `TargetObject.play()`

有了 JDK 动态代理之后，调用链变成：

- `mi.play()`
  - `mi` 的真实类型是 `$Proxy0`
  - 进入 `$Proxy0.play()`
    - 调用 `super.h.invoke(...)`
      - `h` 是 `ProxyFactory`
      - 进入 `ProxyFactory.invoke(...)`
        - 执行前置增强
        - 通过反射调用 `TargetObject.play()`
        - 执行后置增强

所以 JDK 动态代理的核心不是“改了目标对象”，而是“让你拿到的对象变成代理对象”。业务代码以为自己在调用 `MyInterface.play()`，实际先进入 `$Proxy0.play()`，再由 `$Proxy0` 回调 `InvocationHandler.invoke(...)`，最后才调用原始对象。

> [!tip]
> 这就是后面 Spring AOP 里 `JdkDynamicAopProxy.invoke(...)` 能生效的基础。
> 
> Spring AOP 不是让业务代码直接调用原始 Bean，而是尽量让外部拿到代理 Bean。调用代理 Bean 的方法时，才进入增强链。

## 原文代码：从 ProxyFactory 到 $Proxy0

最近在看 Spring AOP 部分的源码，所以对 JDK 动态代理具体是如何实现的这件事产生了很高的兴趣，而且能从源码上了解这个原理的话，也有助于对 spring-aop 模块的理解。话不多说，上代码。

```java
/**
 * 一般会使用实现了 InvocationHandler接口 的类作为代理对象的生产工厂，
 * 并且通过持有 被代理对象target，来在 invoke()方法 中对被代理对象的目标方法进行调用和增强，
 * 这些我们都能通过下面这段代码看懂，但代理对象是如何生成的？invoke()方法 又是如何被调用的呢？
 */
public class ProxyFactory implements InvocationHandler {

    private Object target = null;

    public Object getInstanse(Object target){

        this.target = target;
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {

        Object ret = null;
        System.out.println("前置增强");
        ret = method.invoke(target, args);
        System.out.println("后置增强");
        return ret;
    }
}

/**
 * 实现了 接口MyInterface 和接口的 play()方法，可以作为被代理类
 */
public class TargetObject implements MyInterface {

    @Override
    public void play() {
        System.out.println("妲己，陪你玩 ~");
    }
}

/**
 * 测试类
 */
public class ProxyTest {

    public static void main(String[] args) {
        TargetObject target = new TargetObject();
        // ProxyFactory 实现了 InvocationHandler接口，其中的 getInstanse()方法 利用 Proxy类
        // 生成了 target目标对象 的代理对象，并返回；且 ProxyFactory 持有对 target 的引用，可以在
        // invoke() 中完成对 target 相应方法的调用，以及目标方法前置后置的增强处理
        ProxyFactory proxyFactory = new ProxyFactory();
        // 这个 mi 就是 JDK 的 Proxy类 动态生成的代理类 $Proxy0 的实例，该实例中的方法都持有对
        // invoke()方法 的回调，所以当调用其方法时，就能够执行 invoke() 中的增强处理
        MyInterface mi = (MyInterface) proxyFactory.getInstanse(target);
        // 这样可以看到 mi 的 Class 到底是什么
        System.out.println(mi.getClass());
        // 这里实际上调用的就是 $Proxy0代理类 中对 play()方法 的实现，结合下面的代码可以看到
        // play()方法 通过 super.h.invoke() 完成了对 InvocationHandler对象(proxyFactory)中
        // invoke()方法 的回调，所以我们才能够通过 invoke()方法 实现对 target对象 方法的
        // 前置后置增强处理
        mi.play();
        // 总的来说，就是在 invoke()方法 中完成 target目标方法 的调用，及前置后置增强，
        // JDK 动态生成的代理类中对 invoke()方法 进行了回调
    }

    /**
     * 将 ProxyGenerator 生成的动态代理类的输出到文件中，利用反编译工具 luyten 等就可
     * 以看到生成的代理类的源码咯，下面给出了其反编译好的代码实现
     */
    @Test
    public void generatorSrc(){
        byte[] bytesFile = ProxyGenerator.generateProxyClass("$Proxy0", TargetObject.class.getInterfaces());
        FileOutputStream fos = null;
        try{
            String path = System.getProperty("user.dir") + "\\$Proxy0.class";
            File file = new File(path);
            fos = new FileOutputStream(file);
            fos.write(bytesFile);
            fos.flush();
        } catch (Exception e){
            e.printStackTrace();
        } finally{
            try {
                fos.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
}

/**
 * Proxy 生成的代理类，可以看到，其继承了 Proxy，并且实现了 被代理类的接口MyInterface
 */
public final class $Proxy0 extends Proxy implements MyInterface {
    private static Method m1;
    private static Method m0;
    private static Method m3;
    private static Method m2;

    static {
        try {
            $Proxy0.m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            $Proxy0.m0 = Class.forName("java.lang.Object").getMethod("hashCode", (Class<?>[])new Class[0]);
            // 实例化 MyInterface 的 play()方法
            $Proxy0.m3 = Class.forName("com.shuitu.test.MyInterface").getMethod("play", (Class<?>[])new Class[0]);
            $Proxy0.m2 = Class.forName("java.lang.Object").getMethod("toString", (Class<?>[])new Class[0]);
        }
        catch (NoSuchMethodException ex) {
            throw new NoSuchMethodError(ex.getMessage());
        }
        catch (ClassNotFoundException ex2) {
            throw new NoClassDefFoundError(ex2.getMessage());
        }
    }

    public $Proxy0(final InvocationHandler invocationHandler) {
        super(invocationHandler);
    }

    public final void play() {
        try {
        	// 这个 h 其实就是我们调用 Proxy.newProxyInstance()方法 时传进去的 ProxyFactory对象(它实现了
            // InvocationHandler接口)，该对象的 invoke()方法 中实现了对目标对象的目标方法的增强。
        	// 看到这里，利用动态代理实现方法增强的实现原理就全部理清咯
            super.h.invoke(this, $Proxy0.m3, null);
        }
        catch (Error | RuntimeException error) {
            throw new RuntimeException();
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }

    public final boolean equals(final Object o) {
        try {
            return (boolean)super.h.invoke(this, $Proxy0.m1, new Object[] { o });
        }
        catch (Error | RuntimeException error) {
            throw new RuntimeException();
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }

    public final int hashCode() {
        try {
            return (int)super.h.invoke(this, $Proxy0.m0, null);
        }
        catch (Error | RuntimeException error) {
            throw new RuntimeException();
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }

    public final String toString() {
        try {
            return (String)super.h.invoke(this, $Proxy0.m2, null);
        }
        catch (Error | RuntimeException error) {
            throw new RuntimeException();
        }
        catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }
}
```
