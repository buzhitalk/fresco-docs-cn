---
docid: writing-custom-views
title: 自定义 View
layout: docs
permalink: /docs/writing-custom-views.html
---

### DraweeHolders

总有一些时候，`DraweeViews`是满足不了需求的，在展示图片的时候，我们还需要展示一些其他的内容，或者支持一些其他的操作。在同一个View里，我们可能会想显示一张或者多张图。

在自定义View中，Fresco 提供了两个类来负责图片的展现:

* `DraweeHolder` 单图情况下用。
* `MultiDraweeHolder` 多图情况下用。

`DraweeHolder` 是持有 DraweeHierarchy 的一个类，它和 DraweeController 紧密相关。它允许你在自定义View中也能够使用所有Drawee的特性！ 你只需要通过 `mDraweeHolder.getTopLevelDrawable()` 就能够获取Drawable。你需要知道 Android Drawable 需要一些特殊照顾，所以请仔细阅读下面的内容。

`MultiDraweeHolder` 仅仅是 `DraweeHolder` 的一个列表，它其实是一个语法糖：）

### 自定义View需要完成的事情

Android 呈现View对象，只有View对象才能得到一些系统事件的通知。`DraweeViews`
处理这些事件通知，高效地管理内存。使用`DraweeHolder`时，你需要自己实现这几个方法。

#### 处理 attach/detach 事件

**如果没按照以下步骤实现的话，很可能会引起内存泄露**

当图片不再在View上显示时，比如滑动时View滑动到屏幕外，或者不再绘制，图片就不应该再存在在内存中。Drawees 监听这些事情，并负责释放内存。当图片又需要显示时，重新加载。

这些在`DraweeView`中是自动的，但是在自定义View中，需要我们自己去操作，如下:

```java
DraweeHolder mDraweeHolder;

@Override
public void onDetachedFromWindow() {
  super.onDetachedFromWindow();
  mDraweeHolder.onDetach();
}

@Override
public void onStartTemporaryDetach() {
  super.onStartTemporaryDetach();
  mDraweeHolder.onDetach();
}

@Override
public void onAttachedToWindow() {
  super.onAttachedToWindow();
  mDraweeHolder.onAttach();
}

@Override
public void onFinishTemporaryDetach() {
  super.onFinishTemporaryDetach();
  mDraweeHolder.onAttach();
}
```

在View获取到 attach/detach 时间时通知 `Holder` 是一件非常重要的事情！如果 Holder 错过了一个attach事件，那么图片很可能不会被正常加载，因为它认为此时不是加载图片的时机。同理如果 Holder 错过了一个detach事件，那么不需要显示的图片依然会被保留在内存中，因为它认为此时图片仍然显示。

#### 处理触摸事件

如果你启用了[点击重新加载](placeholder-failure-retry.html)，你需要告诉它用户点击了屏幕，像这样：

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
  return mDraweeHolder.onTouchEvent(event) || super.onTouchEvent(event);
}
```

#### 自定义 onDraw

你必须调用

```java
Drawable drawable = mDraweeHolder.getTopLevelDrawable();
drawable.setBounds(...);
...
drawable.draw(canvas);
```

否则图片将不会出现。

* 不要向下转换这个Drawable，底层实现可能会在无任何通知的情况下发生改变。
* 不要变换这个Drawable，只需要正确设置下边界。
* 如果你需要对Canvas做变换，请确认你在 invalidate 的时候正确刷新了图片所在区域。

#### 其他应该做的

* 设置[Drawable.Callback](http://developer.android.com/reference/android/graphics/drawable/Drawable.Callback.html)

```java
// 当设置Holder时，不要忘记将顶层Drawable的Callback设置为它。
mDraweeHolder = ...
mDraweeHolder.getTopLevelDrawable().setCallback(this);

// 当不使用Holder时，不要忘记将顶层Drawable的Callback设置为null。
mDraweeHolder.getTopLevelDrawable().setCallback(null);
mDraweeHolder = ...
```

* 复写 `verifyDrawable:`

```java
@Override
protected boolean verifyDrawable(Drawable who) {
  if (who == mDraweeHolder.getTopLevelDrawable()) {
    return true;
  }
  // 对其他Drawable的验证逻辑
}
```

> 确保 `invalidateDrawable` 处理了图片占用的那块区域。如果你在图片显示前对Canvas应用了一些变换逻辑，那么invalidate 的时候需要考虑到这些变换。最简单的就是参考 Android ImageView 的[做法](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.4.4_r1/android/widget/ImageView.java#192)

### 初始化 View 和 DraweeHolder

这同样需要非常小心和细致

#### 构造函数

我们推荐如下实现构造函数:

* 重写所有构造函数，共 3 个
* 在每个构造函数中调用同等签名的父类构造函数，和一个私有的 `init` 方法。
* 在 `init` 方法中执行初始化操作。

即，不要在构造函数中用 `this` 来调用另外一个构造。

这样可以保证，不管调用哪个构造，都可以正确地执行初始化流程。holder 在 `init` 方法中被创建。

#### 创建 Holder

如果有可能，只在 View 创建时，创建 Drawees。创建 DraweeHierarchy 开销较大，最好只做一次。更重要的是，Holder 的生命周期应该与 View 的相一致。最好的方式是在 View 构造的时候来创建 Holder。

```java
class CustomView extends View {
  DraweeHolder<GenericDraweeHierarchy> mDraweeHolder;

  // constructors following above pattern

  private void init() {
    GenericDraweeHierarchy hierarchy = new GenericDraweeHierarchyBuilder(getResources());
      .set...
      .set...
      .build();
    mDraweeHolder = DraweeHolder.create(hierarchy, context);
  }
}
```

#### 设置要显示的图片

使用 [controller 构建](using-controllerbuilder.html) 创建DraweeController，然后调用holder的 `setController` 方法，而不是设置给自定义 View。

```java
DraweeController controller = Fresco.newControllerBuilder()
    .setUri(uri)
    .setOldController(mDraweeHolder.getController())
    .build();
mDraweeHolder.setController(controller);
```

### MultiDraweeHolder

和 `DraweeHolder` 相比，`MultiDraweeHolder` 有 `add`, `remove`, `clear`
等方法可以操作 Drawees。如下:

```java
MultiDraweeHolder<GenericDraweeHierarchy> mMultiDraweeHolder;

private void init() {
  GenericDraweeHierarchy hierarchy = new GenericDraweeHierarchyBuilder(getResources());
    .set...
    .build();
  mMultiDraweeHolder = new MultiDraweeHolder<GenericDraweeHierarchy>();
  mMultiDraweeHolder.add(new DraweeHolder<GenericDraweeHierarchy>(hierarchy, context));
  // repeat for more hierarchies
}
```

同样，也需要处理系统事件，设置边界等等，就像处理单个 `DraweeHolder` 那样。


