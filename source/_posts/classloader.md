---
title: Java类的定义加载器 vs 初始加载器
date: 2018-4-9 15:11:04
tags:
  - Java
  - 插件化
---

在阿里的插件化方案[atlas](https://github.com/alibaba/atlas)中，有个细节是在宿主中用Class.forName()加载bundle中的类会报ClassNotFoundException异常，这很奇怪：从现象来看，atlas中宿主无疑可以加载bundle中的类（宿主启动bundle组件肯定需要先加载bundle中的组件类），可是Class.forName()抛出异常又说明无法加载，这不是矛盾么？这两者有什么区别呢？

我们都知道，类是由ClassLoader加载而来，ClassNotFoundException说明该类不在当前ClassLoader的classpath中。那么，Class.forName()抛异常很可能是因为用了跟组件加载不同的ClassLoader，具体是什么不同呢？这就引出了类加载器的两个概念：初始加载器和定义加载器。为了解释，先从使用atlas后ClassLoader的组织关系说起：

![img](https://pic4.zhimg.com/80/v2-445fe83b62dc93a9de5694408b724aef_hd.jpg)

其中DelegateClassLoader顶替了系统的PathClassLoader，成为了LoadedApk的专职ClassLoader，这意味着DelegateClassLoader接管了四大组件的类加载行为：先遵循双亲委托机制委托PathClassLoader加载，若加载失败，再分派给各插件的BundleClassLoader尝试加载。对于通过ActivityManagerService等调起的四大组件来说，组件类的加载都会从LoadedApk的ClassLoader（也就是DelegateClassLoader）的loadClass方法进入，开始类加载流程，所以DelegateClassLoader就叫做这些组件类的`初始加载器`。随着流程进行，属于宿主的组件类，最终被PathClassLoader的defineClass方法加载成功，所以把PathClassLoader叫做这些类的`定义加载器`。而属于bundle的组件类，最终被各自的BundleClassLoader加载成功，BundleClassLoader就是它们的定义加载器。

了解了两种加载器的概念，再回过头看Class.forName()跟组件类加载的区别，见源码：

```java
    /**
     * where {@code currentLoader} denotes the defining class loader of
     * the current class.
     */
    @CallerSensitive
    public static Class<?> forName(String className)
                throws ClassNotFoundException {
        return forName(className, true, VMStack.getCallingClassLoader());
    }

```

```java
public final class VMStack {
    /**
     * Returns the defining class loader of the caller's caller.
     */
    native public static ClassLoader getCallingClassLoader();
}
```

注释明确说明了Class.forName()使用的classloader是当前类的`定义加载器`，而非初始加载器。所以，在宿主中用Class.forName()加载一个类，使用的是最终定义当前类的PathClassLoader，而非DelegateClassLoader，而从PathClassLoader到DelegateClassLoader不存在委托关系（反过来才有），所以Class.forName()无法加载bundle中的类。

明白了原因，解决方法很简单：使用宿主类的初始加载器加载bundle类即可，其中初始加载器可通过getClassLoader()@ContextWrapper获得。

关于类加载过程，[这里](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/index.html)有详细的解释，还介绍了两种加载器的关联之处：

> 一个类的定义加载器是它引用的其它类的初始加载器。如类 `com.example.Outer`引用了类 `com.example.Inner`，则由类 `com.example.Outer`的定义加载器负责启动类 `com.example.Inner`的加载过程。

可以看出，随着引用链的深入，定义加载器会在类加载器的继承树上不可逆的向上委托。这种设计的背后理念是：引用关系应该是单向的、由表及里的，一个底层模块不应该依赖一个上层模块。