---
title: 来了解一下 Jetpack Compose 吧
date: 2019-11-7
tags: [Android, Jetpack, Compose]
---

发表时间：2019.11.7

在前不久的 Android Dev Summit '19 上，Jetpack Compose 终于发布了一个可直接获得的预览版。现在的版本还是 0.1.0-dev02，处于非常早期的版本，官方也再三强调非常有可能产生变化且无法用于生产环境。不过我认为这是简单了解一下 Compose 的好时机。有备而无患。

<!-- more -->

首先来了解一下现在尝试 Compose 所需要的环境：

1. Android Studio 4.0 Canary1
2. Kotlin 1.3.60-eap-25
3. minSdkVersion 21（也就是 Android 5.0）

好的，这篇文章就到此为止吧。

![告辞](https://tva1.sinaimg.cn/large/007X8olVly1g8f4evrq81j3023023jr6.jpg)



等等！写都写了，我还是再多写一点吧。

演这么一出其实也是为了说明，Compose 真的还处于很早期的阶段，估计还需要很长一段路才能真正在项目中实际使用上。但早鸟多少能尝到些甜头。希望各位能通过简单的了解，预判一下以后的开发趋势，趁早规划，适时投资。

同时由于 Compose 还处于很可能发生很多变化的状态，所以本文也不准备过多拘泥于使用的细节，而是从更为大局的层面上介绍一下 Compose。当然，笔者本人经验和眼界极为有限，真知灼见是没有的，希望起到抛砖引玉的作用。

## 开始体验

请记得下载一个[最新的 Android Studio](https://developer.android.com/studio/preview)，需要 4.0 版本，在发文时这意味着你需要使用 canary channel 版本的。

开始新建一个项目，会发现多了一个 Empty Compose Activity 的选项，选择它，各种依赖会为你配置好。那么一个 hello world 会是这个样子的：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyApp()
        }
    }
}

@Composable
fun MyApp() {
    Text("Hello world!")
}
```

好的，你已经知道了 Compose 的基础知识，现在可以尝试用它来构建各种界面了。

![打脸](http://tva1.sinaimg.cn/large/007X8olVly1g8f80fbncej301o01amx4.jpg)

请先慢着动手。如果去看官网的入门教程，你所得到的核心内容的确就是这样。由于文档及教程还很不完善，这里我建议，出门左转学习一下 Flutter。

这不是一句玩笑话。Compose 在控件的使用以及组合上，跟 Flutter 更为相近，而跟 Android 之前的那套 UI 没有太大的关系。使用 Layout Inspector 这个工具来查看 Compose 编写的应用，可以看到 Activity 内是一个 AndroidComposeView 包含了很多 RepaintBoundaryView。而没有使用原先就有的 TextView 等控件。

假如你只接触过 Android 的 UI 编程，可能需要一段时间来熟悉然后转变原先的思维。虽然 Compose 本身还很不完善，但这种声明式的编程范式（declarative programming）可以通过学习 React 或者 Flutter 等 UI 框架未雨绸缪一番。

关于声明式编程稍后再提，我还是继续谈一下一些常见的情形。

### 布局

在布局上，首先会教会你使用 Column 和 Row，能够对应代替 LinearLayout；FrameLayout 也换成了 Stack。如果你对  main axis、cross axis 还比较陌生，如上所说，可以尝试学习一下 React 和 Flutter。

视频中提到，Compose 对 ConstraintLayout 的支持还正在进行中。所以可以猜测还在对于现有的布局控件，以后应该还是会添加支持的。

### 列表

大多数应用都逃不了网络请求 + 列表展示！

Compose 提供了 VerticalScroller 和 HorizontalScroller 来生成列表，在使用上与 RecyclerView 是完全不同的体验：

```kotlin
@Composable
fun MyApp() {
    VerticalScroller {
        Column {
            repeat(20) {
                Row(mainAxisSize = LayoutSize.Expand) {
                    Container(height = 48.dp) {
                        Text("Item $it", modifier = Spacing(left = 16.dp))
                    }
                }
            }
        }
    }
}
```

事实上，Scroller 与 ScrollView 更接近，只提供了一个滚动的功能，并没提到有对 View 进行回收复用。

### Style

```kotlin
@Composable
fun Greeting(name: String) {
    MaterialTheme {
        Surface(color = Color.White) {
            Text(text = "Hello $name!", style = +themeTextStyle { h5 })
        }
    }
}
```

以上代码添加了 Material 的主题。

在我看来，这种方式比使用 XML 那套繁琐、不直观、缺乏文档的方法要好太多。当然，也可能只是因为我菜。

![我太菜](http://tva1.sinaimg.cn/large/007X8olVly1g8gdget2b3j305d04wq2u.jpg)

### Live Preview

Android Studio 4.0 还提供了一个名为 Live Preview 的功能来预览 Compose 的效果。通过给 Compose 函数添加 `@Preview` 注解来预览该函数的效果。这个函数必须是无参的，而且可以给注解传递不同的参数来预览各种不同情况下（不同主题不同屏幕大小等）的界面。

功能是好功能，但代码发生变动后，必须要重新 build 后才能更新预览，现在效率还比较低。

### 状态管理

最后谈谈当状态（数据）发生变化后，Compose 如何响应。

```kotlin
@Model
class State(var count: Int = 0)

@Composable
fun MyApp() {
    val state = State()
    MaterialTheme {
        Stack {
            aligned(Alignment.Center) {
                Button("${state.count} clicks", {
                    state.count++
                })
            }
        }
    }
}
```

上面这段代码，放置了一个按钮，每按一次按钮上的计数就会 +1。代码里并没有对 State 进行监听，然后再对 UI 进行更新。而是在 State 类上添加了 `Model` 注解，Compose 自动进行 UI 的更新（官方称之为 recompose）。

你或许会想：哦，`setState()`。

## Compose?

### 为什么需要 Compose

Android 已经十年，设备的变化非常大，也涌现出很多心的开发技术和思想，但用来开发 UI 的工具却还依旧停留在十年前。不少控件已经过时而且背负了太多历史包袱，重新开始或许是更好的选择。

开发 UI 需要编写 XML 布局，通过代码加载，可能还需要通过 XML 来定义 style，为了编写一个界面要做的工作太多了。而且考虑到 Activity 和 Fragment，需要顾及的就更多了。如果要自定义一个 View，要做什么工作？我想“自定义 View”这几个字，都能吓退一批人吧。

并且大量 UI 工具与系统的版本绑定，新功能新修复无法及时让开发者和用户受益。Material design？Who cares？那是 5.0 以上才大部分支持的东西，更别提 shape theming 了。

原先的控件使用了一些只有官方能使用的黑魔法，也就是 hidden API。Compose 将完全在公开的 API 上进行构建，官方使用的广大的普通开发者也能够使用。

插点私货。我偶尔会看着隔壁 Flutter 流下羡慕的泪水。它提供了大量官方的控件，应对各种场景，而且在各种系统版本上提供统一的行为。而我却需要满世界找非官方的实现，一个个查看是否满足我的需求，是否还在维护，是否需要自己魔改。当然，我无比感恩这些开发者的贡献，但我觉得我们应该被 Android 官方善待。

然后是 data flow。对于 UI 编程来说，分发事件和接收状态是与开发者关联最密切的事情。而现有的 android.widget 在这两个方面都做得不够好：状态的管理比较混乱；事件在分发时就已经改变了控件的状态；listener 可以跟 Kotlin 结合更紧密提供更合适的做法。

### 声明式 UI 编程

声明式编程通常是相对于命令式编程（imperative programming）来说的，不关注编程中具体的过程，而是以最后的结果为重点。在 UI 这一特定的领域来说，声明式编程则意味着：当状态发生变化时，声明式框架会自动更新视图。

声明式的 UI 框架会关注：

1. 对于给定的数据，UI 是如何被展示的；
2. 怎么对事件进行响应；
3. 不考虑 UI 之前的状态对当前的状态产生的影响。

也就是说，它只关心当前的数据（状态）会渲染出什么样的外观，而不把数据当成一个拥有上下文的状态流来看待。

### Composition

> Composition over inheritance.
>
> 组合优于继承。

Compose 通过组合的方式构成界面。这或许也是起名叫 Compose 的原因。

遵循单一责任原则，Compose 函数都只有一个单一的目的，想实现某一种效果，就使用对应的 Compose 函数。比如想要设置背景，则需要使用 `Surface`，没有其他 Compose 函数可以做到这一点。

## @Composable 

使用 `@Composable` 注解的函数只能在另一个的 Composable 函数中调用。看到这句话，你可能会想起：“ 只有替身才能打倒替身 ！”

![你在向我靠近吗](http://tva1.sinaimg.cn/large/007X8olVly1g8ohcynfdxj30f00dm7lb.jpg)

如果你对 Kotlin Coroutine 有所了解，suspend 函数也同样只能在 suspend 函数中进行调用。而 coroutine 的实现，是 Kotlin 编译器在编译时，将 suspend 函数进行了改造，会给函数添加一个 Continuation 参数，并对函数体进行一定程度的改造。详细的这里不多写了。

对于 Composable 函数，同样也是使用了 Kotlin 编译器插件将函数变形了。Composable 函数里添加的则是 [Composer](https://developer.android.com/reference/kotlin/androidx/compose/Composer)，这是一个类似于 gap buffer 的数据结构。Gap buffer 在文本编辑器里被普遍使用，有兴趣的可以多了解一下。

## 总结

再次强调，Compose 处于很早期的阶段，API 也好，具体的底层实现也好，都很可能会发生变化。官方的示例 JetNews 也存在一定的性能问题。

所以我觉得普通开发者还不需要去了解具体使用的细节。但还是有几个建议：

* 如果还没学习 Kotlin，快学吧。
* 然后考虑学习一下 Kotlin coroutine。
* 学习已经比较成熟的声明式 UI 框架，比如 React 和 Flutter。考虑了解一下它们的应用和原理，比如状态管理的最佳实践和 virtual DOM 等。这样可以快速掌握同类型的 UI 框架。

## 扩展资料

[官网 Compose 页面](https://developer.android.com/jetpack/compose)

[Codelab - Jetpack Compose Basics](https://codelabs.developers.google.com/codelabs/jetpack-compose-basics)

[官方示例 JetNews](https://github.com/android/compose-samples/tree/master/JetNews)

[Declarative UI Patterns (Google I/O'19)]( https://www.youtube.com/watch?v=VsStyq4Lzxo )

[What's New in Jetpack Compose (Android Dev Summit '19)](https://www.youtube.com/watch?v=dtm2h-_sNDQ)

[Understanding Compose (Android Dev Summit '19)](https://www.youtube.com/watch?v=Q9MtlmmN4Q0)



