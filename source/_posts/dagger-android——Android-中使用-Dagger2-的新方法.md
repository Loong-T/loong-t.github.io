---
title: dagger.android——Android 中使用 Dagger2 的新方法
date: 2017-07-09
updated: 2017-07-15
tags: [Android, Dagger, 依赖注入, DI]
---

不打算讲解 Dagger2 的基础知识。

原先在 Android 项目中使用 Dagger2，准备好了 Component 和 Module 之后，在 Activity 或者 Fragment 里大概是这么使用的：

<!-- more -->

```java
// AppComponent
@Singleton
@Component(modules = {
        AppModule.class,
})
public interface AppComponent {

    MainActivityComponent.Builder mainComponentBuilder();

    void inject(ThisApplication application);
}

// MainActivityComponent
@ActivityScope
@Subcomponent(modules = {
        MainActivityModule.class,
})
public interface MainActivityComponent {

    void inject(MainActivity mainActivity);

    @Subcomponent.Builder
    interface Builder {
        Builder mainModule(MainActivityModule mainModule);

        MainActivityComponent build();
    }
}

// 在 MainActivity 中
@Override
protected void setupActivityComponent(AppComponent appComponent) {
    mainComponent = appComponent.mainComponentBuilder()
            .mainModule(new MainModule(this))
            .build();
    mainComponent.inject(this);
}
```

这段代码需要在生命周期方法中调用，之后才能正常使用被注入的对象。可以看到 MainActivity 需要知道自身是被 AppComponent 所注入，而被注入的类实际上不该了解（持有）注入者。`dagger.android` 就是为了在这一点上进行改进。

## 先行知识

在展示 `dagger.android` 之前，需要先了解一下 dagger 的 multibinding 功能，对于理解背后的运作机制有好处。

```java
@Module
class MultibindingModule {
    @Provides
    @IntoMap
    @StringKey("a")
    int provideAValue() {
        return "a value";
    }

    @Provides
    @IntoMap
    @StringKey("b")
    int provideBValue() {
        return "b value";
    }
}
```

上面的代码，能够产生一个 `Map<String, String>`，包含了两个 entry，值分别是 `a value` 和 `b value`。这个 Map 可以当成构造参数传入某个地方，也可以像下面的代码一样直接在 Component 中使用。

```java
@Component(modules = MultibindingModule.class)
interface MultibindingComponent {
    Map<String, String> aMap();
}
```

看到这里应该大致会明白 `@IntoMap` 的作用了。搭配 `@StringKey` 注解，`@IntoMap` 可以将方法的返回值加入一个 Map。假如需要的 key 的类型不是 `String`，也可以使用其他的注解，或者自定义一个注解。

有了 multibinding 功能之后，实际上已经可以将最初的注入方案进行改进了。具体可以参考这篇文章[Activities Subcomponents Multibinding in Dagger 2][article-multibinding]。

`dagger.android` 在原理上是接近的，算是一种标准化。

## 使用 `dagger.android`

以撰文时的最新版本 `2.11` 为例说明。假定我们的应用组织层次是 `Appliction <-- MainActivity <-- MainFragment`，这样方便说明。

第一件事是添加依赖。`dagger.android` 需要在原来的基础上再添加依赖：

```groovy
implementation 'com.google.dagger:dagger-android:2.11'
implementation 'com.google.dagger:dagger-android-support:2.11' // 如果使用 support library
annotationProcessor 'com.google.dagger:dagger-android-processor:2.11'
// kapt 'com.google.dagger:dagger-android-processor:2.11' // 使用 Kotlin kapt
```

在 AppComponent 这一级，将 `AndroidInjectionModule` 加入 modules：

```java
@Component(modules = {
        AndroidInjectionModule.class,
        AppModule.class,
})
public interface AppComponent {
    void inject(ThisApplication application);
}
```

在 MainActivity 这一层，首先确保使用的是 `@Subcomponent`，并且使 MainComponent 继承 `AndroidInjector<T>`。至于 `T` 的类型，这里需要被注入的是 MainActivity，所以就是 `MainActivity`。在 Subcomponent 内的 Builder，则需要继承 `AndroidInejector.Builder<MainActivity>`。由于 `AndroidInejector.Builder` 本身是抽象类，所以原本的 Builder 也需要改为抽象类。

```java
@Subcomponent
public interface MainActivityComponent extends AndroidInjector<MainActivity> {
    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<MainActivity> {
    }
}

@Module(subcomponents = MainActivityComponent.class)
public abstract class MainActivityModule {
    @Binds
    @IntoMap
    @ActivityKey(MainActivity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    bindActivityInjectorFactory(MainActivityComponent.Builder builder);
}
```

上面的代码中同时也新增了 MainActivityModule，Module 注解中将 subcomponents 指定为 MainActivityComponent。其他内容依葫芦画瓢。再将 MainActivityModule 添加到 AppComponent 中（感谢 [@xiaobailong24][xiaobailong24 github] 指出该处遗漏）：

```java
@Component(modules = {
        AndroidInjectionModule.class,
        AppModule.class,
        MainActivityModule.class,
})
public interface AppComponent {
    void inject(ThisApplication application);
}
```

到这里，依赖部分已经编写完毕，接下来是使用依赖。修改自定义的 Application：

```java
public class ThisApplication extends Application implements HasActivityInjector {
  @Inject DispatchingAndroidInjector<Activity> dispatchingActivityInjector;

  @Override
  public void onCreate() {
    super.onCreate();
    DaggerApplicationComponent.create().inject(this);
  }

  @Override
  public AndroidInjector<Activity> activityInjector() {
    return dispatchingActivityInjector;
  }
}
```

Application 实现了 HasActivityInjector，方法的实现仅是返回一个 dagger 为我们注入的对象。

最后在 Activity 中，在 `onCreate` 方法前，调用一个方法即可：

```java
public class MainActivity extends Activity {
  public void onCreate(Bundle savedInstanceState) {
    AndroidInjection.inject(this);
    super.onCreate(savedInstanceState);
  }
}
```

## 简单分析

最后实现注入的代码，无疑是 `AndroidInjection.inject(this)` 这一句，那么来看看这个方法的实现：

```java
public static void inject(Activity activity) {
    checkNotNull(activity, "activity");
    Application application = activity.getApplication();
    if (!(application instanceof HasActivityInjector)) {
        throw new RuntimeException(
                String.format(
                        "%s does not implement %s",
                        application.getClass().getCanonicalName(),
                        HasActivityInjector.class.getCanonicalName()));
    }

    AndroidInjector<Activity> activityInjector =
        ((HasActivityInjector) application).activityInjector();
    checkNotNull(
            activityInjector,
            "%s.activityInjector() returned null",
            application.getClass().getCanonicalName());

    activityInjector.inject(activity);
}
```

代码很简单，就是从 Application 得到 ActivityInjector，调用 `inject` 方法注入这个 activity。Application 所持有的 `DispatchingAndroidInjector`，内部维护了一个 map。可以注意到，这个 map 的类型是 `Map<Class<? extends T>, Provider<AndroidInjector.Factory<? extends T>>>`，并且构造方法使用了 `@Inject` 注解来注入这个 map。可知这个 map 便是我们使用 `@ActivityKey` 和 `@IntoMap` 所产生的。

AndroidInjector.Factory 的实现则是我们所写的 Subcomponent.Builder。通过 MainActivity.class 在 map 里得到对应的 AndroidInjector.Factory 后，就可以创建对应的 Component——也就是 AndroidInjector——接着就可以将 Component 所拥有的类型注入进 Activity 了。

## 补充内容

观察上面例子里的 MainActivityComponent，会发现这个接口的实际内容只有指定了 MainActivity 的类型，其他的部分都是固定的。减少 boilerplate code 是 dagger 的目的之一，所以针对这种情况，dagger 添加了一个 `@ContributesAndroidInjector` 注解，用来生成这样的 Subcomponent。

`@ContributesAndroidInjector` 使用在 module 中的抽象方法上。该方法不该有参数，返回类型必须是 Activity、Fragment 和 Service 等 Android Framework 的类型。dagger 将会为这样的一个方法生成一个对应的 Subcomponent。这个注解还能接受一系列 Module 作为值，这些 module 将会成为所生成的 Subcomponent 的 module。

对应上面的例子，可以将 MainActivityComponent 文件删除，然后将 MainActivity 改为如下：

```java
@Module
public abstract class MainActivityModule {
    @ContributesAndroidInjector
    abstract MainActivity contributeMainActivity();
}
```

使用 ProGuard 时，需要加入如下规则：

```proguard
# dagger
-dontwarn dagger.android.**
```

如果你想看实际使用的例子，可以参考我的 [Android Showcase][android-showcase]。

## 跋

`dagger` 的基础使用并不是很难。但在结合实际项目后，需要合理组织层级，考虑 Scope 的使用，配合上额外的 `Subcomponent` 等等，再加上 Android 独特的环境，往往会变得复杂而繁琐。而 `dagger.android` 很好地简化了 DI 的使用，能够将更多精力放到实际的应用上，个人推荐使用。

[article-multibinding]: https://medium.com/azimolabs/activities-subcomponents-multibinding-in-dagger-2-85d6053d6a95
[android-showcase]: https://github.com/Loong-T/Android-Showcase
[xiaobailong24 github]: https://github.com/xiaobailong24