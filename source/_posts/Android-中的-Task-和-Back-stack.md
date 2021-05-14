---
title: Android 中的 Task 和  Back stack
date: 2013-08-19
updated: 2017-02-20 16:16:41
tags: [Android, Task]
toc: true
---

2017-02-20 更新：旧文搬运。内容很浅显，但这部分内容实际上是需要好好掌握的，等以后会再充实一下。

在近来做的实战项目中，终于涉及到了这个方面的内容，那就趁机学一学用一用。

事实上，这方面的内容并不难理解，但是总让我感觉不知该怎么用才好。这里也就只能列一列几种模式，至于相应的解决方法，还是需要自己去琢磨试验。

<!-- more -->

下面的内容参考，都是来自官方文档。

Task 是用户在进行特定操作时调用的 activity 的一个集合。这些 activity 是由回退栈（back stack）来进行管理的，以打开的顺序来进行排列。

Home screen 是大多数 task 开始的地方。当用户在点击应用的图标时，该应用的 task 就来到了前台；如果还没有它的 task，则会新建一个。打开的 activity 则会被放在这个 stack 最底部（事实上也是在回退栈的最顶部，因为栈内只有这么一个元素）。

当在栈内的这个 activity 打开其他的 activity 时，新的 activity 会被被放置在栈的顶部，并且得到焦点。用户点击返回键后，当前的 activity 被销毁，同时也从栈中被弹出，此时在栈顶的 activity 被恢复。在回退栈中的 activity 是不可能进行重新排序的，只有栈的 push 和 pop 操作（LIFO）。

继续按返回键，task 中的 activity 将会依次被弹出，直到其中再没有 activity，这时，这个 task 也就不存在了。

用户开始一个新的 task 时，原来的 task 可以被移到后台，其中的所有 activity 都会停止，但回退栈保持完整。并且用户此时可以回到原来的 task，比如通过选择最近应用，或者是在主页再次点击开始那个应用。

需要注意的是，当运行了太多的 task 时，系统会开始销毁后台的 activity。被销毁的 activity 的状态将会丢失，但是 task 的回退栈不会丢失。

如果一个 activity 能在多个 activity 中被打开，由于回退栈不能被重新排序，此时这个回退栈中会出现这个 activity 的多个实例，甚至在多个 task 中被实例化多次。这时候再点击返回键，还是依旧将这些 activity 依次弹出，并且各自有各自的状态。如果不想 activity 被实例化多次，可以对这种行为进行更改。

## 定义启动模式

启动模式允许你定义新的 activity 实例与当前 task 的关系。有两种定义方法：

1. 使用 manifest 文件
1. 使用 Intent 的 flags

`<activity>` 中可使用的属性：

* taskAffinity
* launchMode
* allowTaskReparenting
* clearTaskOnLaunch
* alwaysRetainTaskState
* finishOnTaskLaunch

flags 可使用的值：

* FLAG_ACTIVITY_NEW_TASK
* FLAG_ACTIVITY_CLEAR_TOP
* FLAG_ACTIVITY_SINGLE_TOP

Intent 的 flags 方式优先于 manifest，但是，两个方式不能等同。某些效果只能通过在 manifest 中定义的启动模式实现，不能通过 flags 实现；反之亦然。

### 使用 manifest 文件

使用 `<activity>` 元素中的 launchMode 属性。launchMode 属性可设为四种启动模式：

*   standard
    默认值。系统在启动 activity 的 task 中创建一个新的 activity 实例，并且将 intent 传送给它。该 activity 可以被实例化多次，各个实例可以属于不同的 task，一个 task 中也可以存在多个实例。
*   singleTop
    如果 activity 已经存在一个实例并位于当前 task 的栈顶，则系统会调用已有实例的 `onNewIntent()` 方法把 intent 传递给已有实例，而不是创建一个新的实例。activity 可以被实例化多次，各个实例可以属于不同的 task，一个 task 中可以存在多个实例（但仅当栈顶的实例不是该 activity 的）。
*   singleTask
    系统将创建一个新的 task，并把 activity 实例作为根放入其中。但是，如果 activity 已经在其它 task 中存在实例，则系统会通过调用其实例的 `onNewIntent()` 方法把 intent 传给已有实例，而不是再创建一个新实例。 此 activity 同一时刻只能存在一个实例。
*   singleInstance
    除了系统不会把其它 activity 放入当前实例所在的 task 之外，其它均与 singleTask 相同。 无论 activity 是在一个新的 task 中启动，还是位于其它已有的 task 中，用户总是可以用回退键返回到前一个 activity 中。 但是，如果你启动了一个启动模式设为singleTask的 activity，且有一个后台 task 中已存在实例的话，则这个后台 task 就会整个转到前台。 这时，当前的回退栈就包含了这个转入前台的 task 中所有的 activity，位置是在栈顶。

### 使用Intent标志

这个标志可以修改的默认模式包括：

*   FLAG_ACTIVITY_NEW_TASK
    在新的 task 中启动 activity。如果要启动的 activity 已经运行于某 task 中，则那个 task 将调入前台，最后保存的状态也将恢复，activity 将在 `onNewIntent()` 中接收到这个新 intent。 这个过程与前一节所述的  singleTask 模式相同。
*   FLAG_ACTIVITY_SINGLE_TOP
    如果要启动的 activity 就是当前 activity（位于回退栈顶），则已存在的实例将接收到一个 `onNewIntent()` 调用，而不是创建一个 activity 的新实例。 这个过程与前一节所述的 singleTop 模式相同。
*   FLAG_ACTIVITY_CLEAR_TOP
    如果要启动的 activity 已经在当前 task 中运行，则不再启动一个新的实例，且所有在其上面的 activity 将被销毁，然后通过 `onNewIntent()` 传入 intent 并恢复 activity（不在栈顶）的运行。 此种模式在launchMode中没有对应的属性值。FLAG_ACTIVITY_CLEAR_TOP 常与 FLAG_ACTIVITY_NEW_TASK 一起使用。这表示先定位其它 task 中已存在的 activity，再把它放入可以响应 intent 的位置。