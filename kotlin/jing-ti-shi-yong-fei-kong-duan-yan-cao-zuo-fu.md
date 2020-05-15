---
description: >-
  原文链接：https://medium.com/@igorwojda/kotlin-combating-non-null-assertions-5282d7b97205
---

# 警惕使用非空断言操作符（!!）

我最近一直在审查Kotlin项目，开发人员使用很多**非空断言操作符**（`!!`）的现象令我非常不安。这个操作符应当很少被用到，因为它是一个潜在的**空指针异常**（ NullPointerException）的信号。每次在Kotlin代码中看到这两个叹号的时候，你的脑海里应该跳出一个大大的黄色警号。让我说得清楚一点，我并不是说要完全不使用非空断言操作符，而是要确保100%正确使用了这个操作符，即使你认为是正确使用了它，最好也在[Kotlin Slack 频道](http://slack.kotlinlang.org/)中确认一下。这个操作符适宜的使用环境是非常有限的，双重感叹号意味着“双重警号”或者“嘿，小心这里”，但一些开发者只是简单地把它当作另一个操作符来使用。让我们深入地研究一些示例并通过删除一些不必要的非空断言操作符来重构代码。

## 警示1

永远不要使用这样的代码，因为使用多个非空断言会增加潜在空指针异常的风险。 如果在此行中遇到异常，则将无法判断公司或者地址是否为空。

```kotlin
person.company!!.address!!.country
```

安全调用操作符将确保代码安全，并确保空指针异常永远不会发生。

```kotlin
person.company?.address?.country
```

如果你确实确定这些变量将不会为空，则只需将它们声明为非空类型即可，这样也不需要安全调用操作符了。

## 警示2

这是一个与集合过滤有关的示例。

```kotlin
class Foo(val name: String)
val list = listOf(Foo("Big"), null, Foo("Small")) //推断类型 List<Foo?>
list.filter { it != null }
    .map { it!!.name.toUpperCase() }
```

代码至少在目前为止工作良好。 当你决定在将来某个时候更改过滤器标准时代码可能会崩溃。 我们可以在此处使用与前面示例相同的安全调用操作符，但`filterNotNull`方法将是更好的解决方案。

```kotlin
class Foo(val name: String)
val list = listOf(Foo("Big"), null, Foo("Small")) //推断类型 List<Foo?>
list.filterNotNull()
    .map { it.name.toUpperCase() }
```

`filterNotNull`方法会移除空值并将可空类型`Foo?`强制转换为非空类型`Foo`。 现在它是非空类型`Foo`，不再需要非空断言。

## 警示3

这是用于存储从服务器加载的数据的类的示例。 我们解析JSON并将数据放到一个类中，这没什么花哨的。

```kotlin
class ProfileData {

    var name: String? = null
    var thumbnail: Image? = null

    val imageUrl: String
        get() = thumbnail!!.url
}
```

假设我们想返回不可为空的`imageUrl`，因为我们想在UI中显示它。 让我们在这里考虑两种情况：

如果`thumbnail`属性可以为空，则将其定义为可空类型并且使用非空断言操作符，这将在某些时候导致应用程序崩溃。 要解决此问题，我们可以将安全调用操作符与**猫王**（ elvis）操作符结合使用并返回默认值。

```kotlin
class ProfileData {

    var name: String? = null
    var thumbnail: Image2? = null

    val imageUrl: String
        get() = thumbnail?.url ?: ""
}
```

如果`thumbnail`属性不能为空，那么我们可以简单地将lateinit修饰符与非空数据类型一起使用。

```kotlin
class ProfileData {

    lateinit var name: String
    lateinit var thumbnail: Image

    val imageUrl: String
        get() = thumbnail.url
}
```

## 警示4

这个例子就有点极端了，但它来自真实的项目。 我真的惊讶于一些（Java）程序员的创造力:-\)我可以对这段代码说很多……

```kotlin
open class BasePresenter<V> {
    private var weakRef: WeakReference<V>? = null

    val view: V?
        get() = if (weakRef == null) null else weakRef!!.get()

    val isViewAttached: Boolean
        get() = weakRef!= null && weakRef!!.get() != null
}
```

现在不是花时间指责的时候，而是要找到解决方案，因此我们可以使用一个安全调用操作符和一些重构来使此代码更安全，更简单。

```kotlin
open class BasePresenter<V> {
    private var weakRef: WeakReference<V>? = null

    val view: V?
        get() = weakRef?.get()

    val isViewAttached: Boolean
        get() = (view != null)
}
```

实际上，我甚至因为这样的代码产生了做这个[Kotlin插件](https://youtrack.jetbrains.com/issue/KT-18372)的想法，这样以后我们将不会再看到这样的代码（如果愿意，请尝试一下）。

## 警示5

提出者：[Jean-Michel Fayard](https://medium.com/@jm_fayard/one-important-trick-missing-are-delegated-properties-https-kotlinlang-org-docs-reference-b31d89ada6)

此示例来自Android平台，我们把`adapter`用作`RecyclerView`的数据源。

```kotlin
var recyclerView: RecyclerView? = null
val adapter : Adapter? = null
fun onCreate() {
setContentView(...)
   recyclerView = ....
   adapter = createAdapter()
   recyclerView!!.adapter = adapter!!
}
```

要修复此代码，我们可以结合使用`bindView`代理属性（kotterknife库）和`Lazy`代理（Kotlin stdlib）。

```kotlin
val recyclerView: RecyclerView by bindView(R.id.recycler)
val adapter : Adapter by lazy {
    createAdapter()
}
fun onCreate() {
   setContentView(...)
   reyclerview.adapter = adapter
}
```

更新后的代码除了安全性外，还为我们带来了其他好处。 通过使用只读变量，我们可以使代码更具表现力，并在一行中执行变量声明和初始化。

## 总结

每次使用非空断言操作符时都相当于对Kotlin编译器说：“我100％肯定它不会为空”，事实是，在大多数情况下，都有更好和更安全的方法来实现相同的目的。 

我希望这篇文章可以让你更深入地了解什么情况下不应该使用非空断言，并且每次你在代码中看到这两个字符（!!）时，脑海中都会出现大黄色警告标志。

