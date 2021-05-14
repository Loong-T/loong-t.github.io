---
title: 在 SwipeRefreshLayout 中加入多个子 View
date: 2014-09-29
updated: 2017-02-23 17:43:56
tags: [Android, View]
---

2017-02-23 更新：旧文搬运。

SwipeRefreshLayout 是由官方提供的下拉刷新 Widget。最低在 v4 中可用。最近使用了一下，发现虽然是官方出品，但也还是不够理想。

原先尝试使用了 Android L 中提供的新支持库 RecyclerView，彼此之间的兼容性就不够好。（RecyclerView 在那是也是新库，bug 多多，现在当然没有这种情况了。）

后来因为需要实现滑动到底部自动加载更多数据的功能，把 RecyclerView 换回了 ListView。在打算添加 [FloatingActionButton][fab lib] 在同一画面上时，发现 SwipeRefreshLayout 不够方便。根据 FloatingActionButton 这个库的说明，需要将 FloatingActionButton 与 ListView 放在同一 ViewGroup 下。

<!-- more -->

但是 SwipeRefreshLayout 只能有一个子视图，不然就会丢异常。于是自然就会在 SwipeRefreshLayout 下加一个 ViewGroup 包装一层来解决这个问题。这样一来，抛出异常的问题是解决了。但是运行后发现，ListView 只能上滑，而不能下拉。一旦下拉，就会触发 SwipeRefreshLayout 的下拉刷新。

可想而知，这是在事件派发上出了问题。下拉的事件在通常情况下应该由 ListView 来进行处理；当 ListView 滚动位置位于顶部时，再由 SwipeRefreshLayout 来进行处理。而现在的情况是，所有下拉手势全都由 SwipeRefreshLayout 处理的。

查阅关于事件派发的资料后，也没有想到比较可行的解决方案。接下来去查看了一下 SwipeRefreshLayout 的源码，结果不怎么麻烦的就解决了。

根据事件派发的知识，在 SwipeRefreshLayout 中找到相关的方法 `onInterceptTouchEvent(MotionEvent ev)` 和 `onTouchEvent(MotionEvent ev)`。查看它们的代码，发现在onIntercreptTouchEvent中有这么一段：

```java
if (!isEnabled() || mReturningToStart || canChildScrollUp()) {
    // Fail fast if we're not in a state where a swipe is possible
    return false;
}
```

显然，`canChildSrollUp()` 方法跟我的问题有密切关联。再追看 `canChildSrollUp()`：

```java
/**
 * @return Whether it is possible for the child view of this layout to
 *         scroll up. Override this if the child view is a custom view.
 */
public boolean canChildScrollUp() {
    if (android.os.Build.VERSION.SDK_INT < 14) {
        if (mTarget instanceof AbsListView) {
            final AbsListView absListView = (AbsListView) mTarget;
            return absListView.getChildCount() > 0
                    && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                            .getTop() < absListView.getPaddingTop());
        } else {
            return mTarget.getScrollY() > 0;
        }
    } else {
        return ViewCompat.canScrollVertically(mTarget, -1);
    }
}
```

注释中也说了，如果子视图是自定义的，那么重写这个方法即可。mTarget 就是 SwipeRefreshLayout 中默认的唯一的子视图。现在根据我的要求，继承 SwipeRefreshLayout 后，将这个方法改为如下：

```java
@Override
public boolean canChildScrollUp() {
    if (android.os.Build.VERSION.SDK_INT < 14) {
        if (mScrollableChild instanceof AbsListView) {
            final AbsListView absListView = (AbsListView) mScrollableChild;
            return absListView.getChildCount() > 0
                    && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                    .getTop() < absListView.getPaddingTop());
        } else {
            return mScrollableChild.getScrollY() > 0;
        }
    } else {
        return ViewCompat.canScrollVertically(mScrollableChild, -1);
    }
}
```

其中，`mScrollableChild` 是由自己定义的，可以滚动的 View。

现在，下拉滚动的问题已经解决了。接下来就是要方便地指定这个 `mScrollableChild`。我希望在我自定义的 SwipeRefreshLayout 中，用 xml 的属性来指定一个id，然后将这个指定 id 的 View 加载到 `mScrollableChild` 上。

在 values 文件夹中新建一个 attrs.xml，内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="ImprovedSwipeLayoutAttrs">
        <attr name="scrollableChildId" format="reference" />
    </declare-styleable>
</resources>
```

于是就可以如下代码一样使用了：

```xml
<in.nerd_is.inactive_weibo.ui.ImprovedSwipeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:fab="http://schemas.android.com/apk/res-auto"
    xmlns:isl="http://schemas.android.com/apk/res-auto"
    android:id="@+id/swipe_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/md_blue_grey_50"
    isl:scrollableChildId="@+id/list_statuses"
    tools:context="in.nerd_is.inactive_weibo.ui.StatusesFragment" >
 
    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">
 
        <ListView
            android:id="@+id/list_statuses"
            android:minHeight="?android:attr/listPreferredItemHeight"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:paddingTop="12dp"
            android:paddingBottom="12dp"
            android:paddingLeft="8dp"
            android:paddingRight="8dp"
            android:clipToPadding="false"
            android:divider="@android:color/transparent"
            android:dividerHeight="12dp"/>
 
        <com.melnykov.fab.FloatingActionButton
            android:id="@+id/button_floating_action"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="bottom|right"
            android:layout_margin="16dp"
            android:src="@drawable/ic_md_create"
            fab:fab_colorNormal="@color/md_blue_400"
            fab:fab_colorPressed="@color/md_blue_grey_500"/>
    </FrameLayout>
 
</in.nerd_is.inactive_weibo.ui.ImprovedSwipeLayout>
```

ImprovedSwipeLayout 全部代码如下，很简单：

```java
public class ImprovedSwipeLayout extends SwipeRefreshLayout {

    private static final String TAG = ImprovedSwipeLayout.class.getCanonicalName();
    private int mScrollableChildId;
    private View mScrollableChild;

    public ImprovedSwipeLayout(Context context) {
        this(context, null);
    }

    public ImprovedSwipeLayout(Context context, AttributeSet attrs) {
        super(context, attrs);

        TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.ImprovedSwipeLayoutAttrs);
        mScrollableChildId = a.getResourceId(R.styleable.ImprovedSwipeLayoutAttrs_scrollableChildId, 0);
        mScrollableChild = findViewById(mScrollableChildId);
        a.recycle();
    }

    @Override
    public boolean canChildScrollUp() {
        ensureScrollableChild();

        if (android.os.Build.VERSION.SDK_INT < 14) {
            if (mScrollableChild instanceof AbsListView) {
                final AbsListView absListView = (AbsListView) mScrollableChild;
                return absListView.getChildCount() > 0
                        && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                        .getTop() < absListView.getPaddingTop());
            } else {
                return mScrollableChild.getScrollY() > 0;
            }
        } else {
            return ViewCompat.canScrollVertically(mScrollableChild, -1);
        }
    }

    private void ensureScrollableChild() {
        if (mScrollableChild == null) {
            mScrollableChild = findViewById(mScrollableChildId);
        }
    }

}
```

[fab lib]: https://github.com/makovkastar/FloatingActionButton