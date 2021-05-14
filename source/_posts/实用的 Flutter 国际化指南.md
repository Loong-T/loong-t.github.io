---
title: 实用的 Flutter 国际化指南
date: 2019-02-30
updated: 2019-04-16 17:11:56
tags: [Flutter, International]
---

作为一个 Android 开发者，Flutter 上来就让我把各类字符串写在 widget 里，其实我心里是拒绝的。硬编码是不可能硬编码的。国际化又不会，就是只能去看看文档，才能学点新姿势这样子。看了文档之后，觉得国际化这部分，还是有点麻烦的，我觉得有必要拎出来单独写写。

<!-- more -->

个人希望能把应用的字符串资源独立出来，以方便管理。至于支持多语言这种，反而是顺带完成的结果。本文以实用优先，因为我认为这部分内容是每个应用都需要使用的。

## 切入点

首先简单认识一下 Flutter 国际化相关的知识点。

添加 `flutter_localizations` 依赖，让 Flutter 知道我们需要使用国际化相关的包。Flutter 自带的 widget 中，也用到了一些字符串资源，比如，`showSearch()` 方法打开的搜索栏提示。而这个包可以提供英文之外的，被 Flutter 内部默认使用的国际化字符串资源。

```yaml
dependencies:
flutter:
  sdk: flutter
flutter_localizations:
  sdk: flutter
```

然后在创建 App 时，加入 `LocalizationsDelegate`，国际化的内容就由这些类来提供。`GlobalMaterialLocalizations.delegate` 提供了 Material 组件库所使用的字符串资源；`GlobalWidgetsLocalizations.delegate` 则定义了在当前的语言中，文字默认的排列方向。

之后我们定义了自己的国际化内容后，也需要加入到这个列表的头部。

还要声明要支持什么语言，`supportedLocales` 这里添加了英文和中文两种。如果说用户的语言不在这个列表内，则会默认使用列表第一项指定的语言。假如你对这个规则不满意，可以使用 `localeResolutionCallback` 参数来自定义自己想要的规则。

```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';

class ThisApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      localizationsDelegates: [
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
      ],
      supportedLocales: [
        const Locale('en', 'US'),
        const Locale('zh', 'CN'),
      ],
      title: 'App Title',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: HomePage(),
    );
  }
}
```

现在，我们将一些自带的国际化资源加入到了应用中，Flutter 自身已经能够使用它们了。但我们要怎么使用它们呢？

通过 `MaterialLocalizations.of(context)` 获取到 `MaterialLocalizations` 的实例，然后访问里面的字符串。比如上面的 title 一行，可以替换为：

```dart
onGenerateTitle: (context) => MaterialLocalizations.of(context).closeButtonLabel,
```

注意这里将 title 替换成 onGenerateTitle 了，因为此时还在初始化 App 中，无法获取到 context，更无法通过 context 获取字符串了。

## 自定义的国际化内容

现在来考虑怎么将我们自己的国际化加入到其中。也就是，需要在 `localizationsDelegates` 中加入自己的 `LocalizationsDelegate`。

查看文档，`LocalizationsDelegate` 需要一个泛型参数。参考官方的文档，可知这里指定的类型就是我们存放字符串的类。在这里，有两种选择：第一是基于 map 的，非常简单的实现；第二个则是通过 Dart 语言中专门负责国际化的 intl 包来实现。接下来我们按次来看看。

### 基于 Map

```dart
class SimpleLocalizations {
  SimpleLocalizations(this.locale);

  final Locale locale;

  static SimpleLocalizations of(BuildContext context) {
    return Localizations.of<SimpleLocalizations>(context, SimpleLocalizations);
  }

  static Map<String, Map<String, String>> _localizedValues = {
    'en': {
      'app_name': 'App Name',
      'hello_world': 'Hello World',
    },
    'zh': {
      'app_name': '应用名',
      'hello_world': '你好世界',
    },
  };

  Map<String, String> get _stringMap {
    return _localizedValues[locale.languageCode];
  }

  String get helloWorld {
    return _stringMap['hello_world'];
  }

  String get appName {
    return _stringMap['app_name'];
  }
}
```

从上面的代码可以看到，这种方法的原理非常简单，就是将所有字符串放进 map，然后通过应用的 `Locale` 来取出对应语言的字符串。使用时，就是 `SimpleLocalizations.of(context).helloWorld` 这样来引用字符串。

其对应的 `LocalizationsDelegate` 如下：

```dart
class SimpleLocalizationsDelegate
    extends LocalizationsDelegate<SimpleLocalizations> {
  const SimpleLocalizationsDelegate();

  @override
  bool isSupported(Locale locale) => ['en', 'zh'].contains(locale.languageCode);

  @override
  Future<SimpleLocalizations> load(Locale locale) {
    return SynchronousFuture<SimpleLocalizations>(SimpleLocalizations(locale));
  }

  @override
  bool shouldReload(SimpleLocalizationsDelegate old) => false;
}
```

只要将这个 `SimpleLocalizationsDelegate` 加入到上面的 delegates 列表中，国际化就算完成了。

回看一下整个流程，并不算复杂，需要经手部分的原理也非常简单，只是一个 map 的使用。使用这个方法可以将整个应用的字符串都集中到一起管理。但是，维护起来还是很不方便。

### 基于 intl

接下来看看基于 intl 包的实现方法是怎么样的。

第一步，添加依赖：

```yaml
dependencies:
  intl: ^0.15.7

dev_dependencies:
  intl_translation: ^0.17.3
```

通过查看官方的例子，可以知道 `Intl.message()` 方法是我们管理字符串的关键。于是去看相关的文档，会发现——嗯，没有卵用（甚至没解释每个参数有什么作用）。

接下来还是一样添加一个跟 `SimpleLocalizations` 差不多类：

```dart
class IntlLocalizations {
  static IntlLocalizations of(BuildContext context) {
    return Localizations.of<IntlLocalizations>(context, IntlLocalizations);
  }

  String get appName {
    return Intl.message('App Name');
  }

  String get helloWorld {
    return Intl.message('Hello world');
  }
}
```

从命令行中运行 `flutter pub pub run intl_translation:extract_to_arb --output-dir=你想要的输出目录 IntlLocalizations所在文件`。这一操作将会在指定目录里生成一个名为 intl_messages.arb 的文件，内容大致如下：

```json
{
  "@@last_modified": "2019-02-17T15:57:00.554988",
  "App Name": "App Name",
  "@App Name": {
    "type": "text",
    "placeholders": {}
  },
  "Hello world": "Hello world",
  "@Hello world": {
    "type": "text",
    "placeholders": {}
  }
}
```

将这个文件复制一份，命名为 intl_en.arb，作为英文版本使用。接着再复制一份，命名为 intl_zh.arb 作为中文版本使用。将 intl_zh.arb 的内容修改为对应中文的内容：

```json
{
  "@@last_modified": "2019-02-17T15:57:00.554988",
  "App Name": "应用名",
  "@App Name": {
    "type": "text",
    "placeholders": {}
  },
  "Hello world": "你好世界",
  "@Hello world": {
    "type": "text",
    "placeholders": {}
  }
}
```

如果需要其他语言的版本，请自行添加并修改。

再来输入一段长长的命令行：`flutter pub pub run intl_translation:generate_from_arb --output-dir=输出目录 --no-use-deferred-loading IntlLocalizations所在文件 所有arb文件`。这样会生成几个 `messages_` 开头的 dart 文件。可以自行查看一下里面的内容，我现在的 Dart 水平还比较菜，就先不分析其中的原理了。

其中名为 messages_all.dart 的文件里，生成了 `initializeMessages(String localeName)` 这个方法，将会在下面的步骤中使用到。

在 `IntlLocalizations` 中添加如下的方法：

```dart
static Future<IntlLocalizations> load(Locale locale) {
  final name =
    locale.countryCode.isEmpty ? locale.languageCode : locale.toString();
  final localeName = Intl.canonicalizedLocale(name);
  return initializeMessages(localeName).then((_) {
    Intl.defaultLocale = localeName;
    return IntlLocalizations();
  });
}
```

`IntlLocalizations` 就准备完毕了。然后，开始实现 delegate，内容很简单：

```dart
class IntlLocalizationsDelegate
    extends LocalizationsDelegate<IntlLocalizations> {
  const IntlLocalizationsDelegate();

  @override
  bool isSupported(Locale locale) => ['en', 'zh'].contains(locale.languageCode);

  @override
  Future<IntlLocalizations> load(Locale locale) {
    return IntlLocalizations.load(locale);
  }

  @override
  bool shouldReload(IntlLocalizationsDelegate old) => false;
}
```

之后就是正常使用流程了，这里不赘诉。回想整个流程，真正国际化的内容在 arb 文件中，对于集中管理字符串来说，比使用 map 还是好一点。但是整个流程还是显得异常麻烦，尤其是两次长得过分的命令行，明显应该由工具来改进。我相信 Flutter/Dart 团队应该会在这一点上做出优化。

## flutter_i18n

那么，有没有一款工具可以解救我们呢？您好，有的。

Android Studio（IDEA）上有一款名为 [flutter_i18n](https://github.com/long1eu/flutter_i18n) 的插件，可以帮助简化这个过程。其原理是通过 arb 文件来自动生成所需要的代码。

插件的使用非常简单，安装后会出现一个新的按钮。一旦你按下这个按钮——boom——插件就会根据 `res/values` 文件夹（Android 开发者觉得很亲切）中的 arb 文件，在 `lib/generated` 中生成 Dart 代码。

那么我们的重心就放在了 arb 文件上。Arb 文件全称是 Application Resource Bundle，是基于 JSON 的 balabala 接下去的我也不想接着说了，因为并不实用。还是来看下 Flutter 国际化中切实相关的部分。

虽然我们知道了 arb 文件是类 JSON 格式，但我们还并不清楚文件里具体需要什么样的内容。这里我们通过 `Intl.message()` 方法再重新认识一下。

```dart
String get appName {
  return Intl.message(
    'App Name',
    desc: 'Name for the application',
    name: 'IntlLocalizations_appName',
  );
}

String hello(String name) {
  return Intl.message(
    'Hello $name',
    name: 'IntlLocalizations_hello',
    desc: 'Say hello to someone',
    args: [name],
    locale: 'en',
    examples: const {'name': 'Someone'},
    meaning: 'What is this?',
    skip: false,
  );
}
```

这里有两个更为详细的实现，其中 `hello` 方法将全部的参数都赋值了，以方便观察通过 `intl_translation` 包处理后的 arb 文件会是什么样的。

不过这之前简单介绍一下 `Intl.message()` 的部分参数。

* `name` 参数必须与函数名一致，或者是`类名_方法名`这个形式——建议使用后者避免冲突；
* `args` 就是重复一遍参数；
* 如果方法没有参数，那么 `name` 和 `args` 可以省略；
* `desc` 参数就是描述这个字符串的字符串，必须是一个字符串字面量；
* `examples` 是参数的示例；
* `desc` 和 `examples` 在运行时不会被使用，但会被提取出来作为额外的信息提供给翻译人员作为参考；
* `skip` 如果为 true，那么这条记录就不会被提取出来；
* 其他的文档里并没有提。

然后我们再运行一下那个很长的命令行，将其处理成 arb 文件看看：

```json
{
  "@@last_modified": "2019-02-18T21:31:28.750455",
  "IntlLocalizations_appName": "App Name",
  "@IntlLocalizations_appName": {
    "description": "Name for the application",
    "type": "text",
    "placeholders": {}
  },
  "IntlLocalizations_hello": "Hello {name}",
  "@IntlLocalizations_hello": {
    "description": "Say hello to someone",
    "type": "text",
    "placeholders": {
      "name": {
        "example": "Someone"
      }
    }
  }
}
```

首先，`meaning` 似乎没有用处。其核心就是 `"IntlLocalizations_appName": "App Name"` 这样的一条一条的记录。以 @ 开头的部分，并不会真正在程序中使用，而是给翻译人员作为参考使用的。

这么一来，我们接下来就可以在 `res/values` 文件夹中创建需要的 arb 文件了。这个插件还提供了快捷创建 arb 文件的功能，只需要在 `res/values` 目录右键选择 New -> Arb File 就可以选择这个 arb 文件的 locale 了。

需要注意的是，在这个插件中，如果字符串内需要包含变量，使用的语法是 `$var_name`，而不是上面例子里使用大括号的形式。

这里我创建了两个 arb 文件：

```json
// strings_en.arb
{
  "appName": "App Name",
  "hello": "Hello $name"
}
// strings_zh_CN.arb
{
  "appName": "应用名",
  "hello": "你好${name}"
}
```

使用插件生成代码后，将 delegate 加入到应用的列表中，使用时也只要直接利用 `S` 这个类名来引用就好：

```dart
MaterialApp(
    localizationsDelegates: [
        S.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
    ],
    supportedLocales: [
        const Locale('en', 'US'),
        const Locale('zh', 'CN'),
    ],
    onGenerateTitle: (context) => S.of(context).appName,
    ...
```

使用了这个插件之后，国际化就算得上方便了。生成的代码也可以稍微看一眼，或许有你用得到的其他方法。

最后提醒一句，由于生成代码是由插件完成的，所以依赖中的 `intl_translation` 可以删掉了。

## 其他与总结

可能有的人会问，不使用 IDEA 的开发者，有没有什么更好的选择呢？或许有。现在还有一个名为 [rosetta](https://pub.dartlang.org/packages/rosetta) 的库，致力于解决 Flutter 国际化太过复杂的问题。我尝试过，但并没有跑通正常的流程，无法更多评价。有兴趣的朋友可以试试看。

到此，这篇指南就结束了，希望能对一些人有帮助。
