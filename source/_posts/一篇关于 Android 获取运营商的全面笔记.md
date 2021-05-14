---
title: 一篇关于 Android 获取运营商的全面笔记
date: 2019-10-20
tags: [Android, 运营商]
---


发表时间：2019-10-20

## 内容总览

本文会给出在 Android 上获取运营商的方法，几个相近方法结果的差异，以及在多卡情况下有效的获取方式。最后额外提一下一种不需要请求设备识别码获取运营商信息的方法。提供可运行的 demo 源码。

<!-- more -->

## MCC 和 MNC

首先介绍一下这两个码，也是获取运营商所必须的。

MCC，Mobile Country Code，移动设备国家代码。MNC，Mobile Network Code，移动设备网络代码。MCC 和 MNC 串在一起后，可以用来表示唯一的移动设备运营商。我国的 MCC 是 460，MNC 则会出现一个运营商拥有多个的情况，比如联通有 01、06、09。当前的码表可以在[这个维基页面](https://en.wikipedia.org/wiki/Mobile_Network_Codes_in_ITU_region_4xx_(Asia)#China_-_CN)找到。

于是可以先根据码表来构建这么一个类：

```kotlin
enum class NetworkOperator(val opName: String) {

    Mobile("移动"),
    Unicom("联通"),
    Telecom("电信"),
    Tietong("铁通"),
    Other("其他");

    companion object {
        /**
         * 根据 [code]（MCC+MNC) 返回运营商
         */
        fun from(code: Int) = when (code) {
            46000, 46002, 46004, 46007, 46008 -> Mobile
            46001, 46006, 46009 -> Unicom
            46003, 46005, 46011 -> Telecom
            46020 -> Tietong
            else -> Other
        }

        fun fromOpName(name: String) = when (name) {
            "移动" -> Mobile
            "联通" -> Unicom
            "电信" -> Telecom
            "铁通" -> Tietong
            else -> Other
        }
    }
}
```

你很可能也知道存在一个叫做 IMSI 的识别码（注意并不是 IMEI 哦）。国际移动用户识别码（International Mobile Subscriber Identity）可以在蜂窝网络中区分不同用户，它是由 MCC、MNC 和 MSIN（移动订户识别码，Mobile Subscription Identification Number）组成的。即 IMSI = MCC + MNC + MSIN。

## 如何获取

在 Android 上首先需要请求 `READ_PHONE_STATE` 权限，然后通过 TelephonyManager 相关方法来获取信息。TelephonyManager 有若干相近的方法，也让人挺困惑的。Demo 里会列出各个方法的结果，有兴趣的可以自行尝试一下。

```kotlin
val tm = getSystemService<TelephonyManager>()
val text = """
    TelephonyManager.getSimOperator(): ${tm.simOperator}
    TelephonyManager.getSimOperatorName(): ${tm.simOperatorName}
    TelephonyManager.getNetworkOperator(): ${tm.networkOperator}
    TelephonyManager.getNetworkOperatorName(): ${tm.networkOperatorName}
    TelephonyManager.getSubscriberId(): ${tm.subscriberId}
    Operator name: ${NetworkOperator.from(Integer.valueOf(tm.simOperator)).opName}
""".trimIndent()
```

在我的设备上得到的结果如下：

```
TelephonyManager.getSimOperator(): 46009				// 当前流量卡，联通
TelephonyManager.getSimOperatorName(): CMCC				// 联通
TelephonyManager.getNetworkOperator(): 46000			// 移动，卡一
TelephonyManager.getNetworkOperatorName(): CHINA MOBILE  // 移动
TelephonyManager.getSubscriberId(): 46002326951xxxx    	 // 移动，这就是 IMSI，xxxx 部分是由我隐去的
Operator name: 联通
```

那么得到了 MCC+MNC，就可以判断出运营商了。

### 双卡情况

如果设备有双卡，该怎么样呢？切换流量卡后再来看一下结果：

```
TelephonyManager.getSimOperator(): 46002				// 当前流量卡，移动
TelephonyManager.getSimOperatorName(): CMCC				// 联通，跟 getSimOperator() 结果不符
TelephonyManager.getNetworkOperator(): 46000			// 移动
TelephonyManager.getNetworkOperatorName(): CHINA MOBILE  // 移动
TelephonyManager.getSubscriberId(): 46002326951xxxx		 // 移动
Operator name: 移动
```

通过尝试可以发现，`TelephonyManager.getSimOperator()` 方法获得的结果是根据当前设置的流量卡变化的。而其他的方法结果都没有变化，甚至得到的信息也是不统一的，可以说没什么用处。

不过我还无法确定这些方法的结果在不同设备不同 ROM 上的效果，仅供参考。可以下载 demo 来查看在自己设备上的结果。

## 不需要权限的方法

我们都知道 Android 会根据设备设置的不同，去加载不同的资源文件夹。最典型的，会根据系统的语言去加载不同语言的字符串资源。而 Android 也可以根据 MCC 和 MNC 加载不同的资源。

于是我灵光一闪，给 `values` 文件夹加上 `-mcc460-mnc00` 后缀，然后在里面放上对应运营商的名字字符串，就可以绕开权限获取到运营商了。

不过等等，这么一来要给每个 MNC 都写一个文件夹，还挺麻烦的，还是用代码来获取吧：

```kotlin
val mcc = resources.configuration.mcc
val mnc = resources.configuration.mnc
val operator = NetworkOperator.from(mcc * 100 + mnc)
```

这个方法也是有缺点的，不然我就把这个方法写在前了——无法获取到当前流量卡的信息，获取到的信息是固定的。

## Demo 源代码

运行以下命令来获得 demo 的源代码：

```
git clone -b sim-operator https://github.com/Loong-T/demo.git
```
