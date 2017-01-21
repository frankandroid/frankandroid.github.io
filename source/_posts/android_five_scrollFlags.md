---
title: AppBarLayout的五种ScrollFlags
date: 2017-01-21 12:15:58
update: 
category: [android5.0]
tags: [android5.0]

---

##AppBarLayout的五种ScrollFlags。

>ScrollFlags共有五种常量值供AppBarLayout的Children View使用，在xml布局文件中通过app:layout_scrollFlags设置，对应的值为：scroll，enterAlways，enterAlwaysCollapsed，snap，exitUntilCollapsed；

###先上代码。
```xml
<root>

<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    tools:context="com.hhly.collapesingdemo.ScrollingActivity">

    <android.support.design.widget.AppBarLayout
        android:id="@+id/app_bar"
        android:layout_width="match_parent"
        android:layout_height="@dimen/app_bar_height"
        android:fitsSystemWindows="true"
        android:theme="@style/AppTheme.AppBarOverlay">

        <android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/toolbar_layout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:minHeight="30dp"
            android:fitsSystemWindows="true"
            app:contentScrim="?attr/colorPrimary"
            app:layout_scrollFlags="scroll">

            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_collapseMode="pin"
                app:popupTheme="@style/AppTheme.PopupOverlay"/>

        </android.support.design.widget.CollapsingToolbarLayout>
    </android.support.design.widget.AppBarLayout>

    <include layout="@layout/content_scrolling"/>

    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="@dimen/fab_margin"
        app:layout_anchor="@id/app_bar"
        app:layout_anchorGravity="bottom|end"
        app:srcCompat="@android:drawable/ic_dialog_email"/>

</android.support.design.widget.CoordinatorLayout>


</root>
```


###scroll

>Child View 伴随着滚动事件而滚出或滚进屏幕。注意两点：第一点，如果使用了其他值，必定要使用这个值才能起作用；第二点：如果在这个child View前面的任何其他Child View没有设置这个值，那么这个Child View的设置将失去作用。


###enterAlways
>快速返回模式。其实就是向下滚动时Scrolling View和Child View之间的滚动优先级问题。对比scroll和scroll | enterAlways设置，发生向下滚动事件时，前者优先滚动Scrolling View，后者优先滚动Child View，当优先滚动的一方已经全部滚进屏幕之后，另一方才开始滚动。


###enterAlwaysCollapsed

>enterAlways的附加值。这里涉及到Child View的高度和最小高度，向下滚动时，Child View先向下滚动最小高度值，然后Scrolling View开始滚动，到达边界时，Child View再向下滚动，直至显示完全。

示例代码：

<pre><code>
...
android:layout_height="@dimen/dp_200"
android:minHeight="@dimen/dp_56"
...
app:layout_scrollFlags="scroll|enterAlways|enterAlwaysCollapsed"
...
</code></pre>


###exitUntilCollapsed
>这里也涉及到最小高度。发生向上滚动事件时，Child View向上滚动退出直至最小高度，然后Scrolling View开始滚动。也就是，Child View不会完全退出屏幕。
<pre><code>
...
android:layout_height="@dimen/dp_200"
android:minHeight="@dimen/dp_56"
...
app:layout_scrollFlags="scroll|exitUntilCollapsed"
...
</code></pre>

###snap
>就是Child View滚动比例的一个吸附效果。也就是说，Child View不会存在局部显示的情况，滚动Child View的部分高度，当我们松开手指时，Child View要么向上全部滚出屏幕，要么向下全部滚进屏幕，有点类似ViewPager的左右滑动。
