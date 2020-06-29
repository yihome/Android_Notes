## 绘制之外

在了解View绘制流程之前，我们要谨记一点，Android系统的整体架构为C/S架构，作为应用程序，最终输出的只是数据，而非页面，即最终页面的显示是由系统来完成的，我们需要做的就是在给定的时间内，将我们要显示的页面通过特定数据的方式描述给`WMS` 和 `SurfaceFlinger` 。

这个数据一般是以 `Surface` 为载体，整个应用层的View体系就是一种方便开发者理解和构造的数据，这些数据通过一定的处理会把最总的数据通过 `Draw	` 方法生成到 `Surface` 内。

`WMS` 会把所有 `window` 中的 `Surface` 的内容安装 `Window` 层级进行整合，最终通过 `SurfaceFlinger` 服务进行渲染，显示在屏幕上。

在简单了解Android真正的绘制原理之后，不难发现页面的显示是与系统服务紧密相关的，但是实际开发过程中我们很少需要了解系统级的知识，只需要专注于View体系即可，这是因为Android系统通过几个中间类帮我们把系统服务级API和应用API进行了隔离，比较有代表性的有如下两个：

* WindowManger 主要任务是完成应用内 `Window` 的创建和管理工作
* ViewRootImpl 和某个特定的 `Window` 绑定，负责这个 `Window` 下的数据生成和管理

我们通常的View体系行为就是在一个特定的 `Window` 下由 `ViewRootImpl` 统筹管理的。

### ViewRootImpl的创建

从Activity的startActivity开始，最终调用到ActivityThread的handleLaunchActivity方法来创建Activity，相关核心代码如下：

```
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {

    ....
    // 创建Activity，会调用Activity的onCreate方法
    // 从而完成DecorView的创建
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        handleResumeActivity(r.tolen, false, r.isForward, !r.activity..mFinished && !r.startsNotResumed);
    }
}

final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;
    // 调用Activity的onResume方法
    ActivityClientRecord r = performResumeActivity(token, clearHide);
    if (r != null) {
        final Activity a = r.activity;
        ...
        if (r.window == null &&& !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            // 得到DecorView
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            // 得到了WindowManager，WindowManager是一个接口
            // 并且继承了接口ViewManager
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                // WindowManager的实现类是WindowManagerImpl，
                // 所以实际调用的是WindowManagerImpl的addView方法
                wm.addView(decor, l);
            }
        }
    }
}

public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    ...

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }
    ...
}
```

在了解View绘制的整体流程之前，我们必须先了解下ViewRoot和DecorView的概念。ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联，相关源码如下所示：

```java
// WindowManagerGlobal的addView方法
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    View pannelParentView = null;
    synchronized (mLock) {
        ...
        // 创建ViewRootImpl实例
        root = new ViewRootImpl(view..getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }
    try {
        // 把DecorView加载到Window中
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        synchronized (mLock) {
            final int index = findViewLocked(view, false);
            if (index >= 0) {
                removeViewLocked(index, true);
            }
        }
        throw e;
    }
}
```

## ViewRootImpl

`ViewRootImpl` 作为一个中间类，在整个 `View` 体系中发挥者巨大的作用，主要有如下几个方面

* 直接持有 `IWindowSession` 和 `IWindow.Stub` 的实现类，实现了和 `WMS`  双向通讯，前者是 `Proxy` 用于向 `WMS` 发送消息，后者是 `stub` 用于接收 `WMS` 的消息。

*  和`ViewGroup` 一样都实现了 `ViewParent` 接口，但是本身不是 `View` 实现类，但是作为`View` 体系的顶层类，相应和处理整个View树的绘制流程
* 通过 `Choreographer` 使用 `Vsync` 同步刷新过程

三个方面中第一方面更多倾向于 `WMS` ，不多做介绍，第二个方面稍后再详细说明。

这里简单介绍一下 `Vsync` ，`Vsync` 是为了保证Android60 fps引入的一种机制，通过 `Choreographer` 我们可以接受到系统每16.6ms一次的屏幕刷新事件，通过源码我们可以看到为了保证在应用UI线程能尽快处理这个事件，`ViewRootImpl` 在开始等待 `Vsync` 信号时会开启 `MQ` 的同步屏障，用于暂时过滤 `MQ` 中的非同步消息。

```java
 void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

> Android使用了 `Vsync` 来进行刷新同步，但是没有办法保证App 和 SurfaceFlinger 把每一帧处理的时间控制在 16.6 ms之内，为了系统流程度，系统还引进了TripleBuffer机制，提前处理下两帧需要显示的画面，即使某些帧处理时间超过16.6 ms，也不一定会掉帧

## 绘制的触发

在了解View层的绘制流程之前我们需要了解一下整个绘制过程是怎么触发的，由谁来触发，目前主要有如下两种方式

* 外部触发，通过 `WMS` 的远程调度，一般是 `Window` 发生变化，如果`Activity`、`Dialog` 等组件的显示隐藏。
* 内部触发，`View` 体系内因为需要，通过 `invalidate` , `requestLayout` ,`postInvalidate ` 触发

不管是通过内部触发还是外部触发都会通过上面的 `scheduleTraversals` ，最终由系统的 `Vsync` 信号开启新一轮的绘制流程。

## 绘制流程

`View` 体系的绘制流程由 `performTraversals` 方法开始。这个方法由上面的 `Vsync` 信号触发，这个方法在开始绘制流程之前会获取当前 `Window` 的显示状态、显示尺寸、横竖屏方向、等信息，同时会触发View体系中三个重要的方法

* `performMeasure`  测量
* `performLayout` 布局
* `performDraw` 绘制

其中`performLayout` 和`performDraw` 两个方法在一次绘制流程中至多只会调用一次，但是 `performMeasure` 在整个 `performTraversals` 中过程中可能会调用根据不同情况可能会调用多次，用于准确获取最终 `View` 显示的大小，其中一个原因是 `ViewRootImpl` 虽然不是`ViewGroup` ，但是在处理`Window` 大小和根`View` 大小时需要发挥一个类似 `ViewGroup` 的作用 。

### onMesure

上面的 `performMeasure` 方法最终会调用`View.measure` 方法，`measure` 方法中会根据状态判断是否可以已缓存的测量尺寸，最终实际的测量过程在  `onMesure` 方法内由子类实现，

`onMesure` 方法的目的是为了让 `View` 结合自身内容大小、子 `View`大小 （ViewGroup）与实际 `MeasureSpec`（包括size和mode，由 `LayoutParams` 和上级 `ViewGroup` 给出）给出一个合适的显示尺寸。

#### View的测量

在实际情况中，这个过程对于普通 `View` 而言相对简单，只需要根据内容大小和给定的 `MeasureSpec` 计算最终的尺寸即可，唯一需要主要的就是 `onMesure` 在不同的场景下可能会被多次调用。

`onMesure` 的参数是两个`int` 型变量，`Java` 下`int` 为32位，这个两个变量中前两位表示的`mode`，后30位表示的才是`size` ，两位的mode分别是。

* `UNSPECIFIED` 0x00 未指定，`View` 可以完全按照内容的大小设置，需要注意的就是需要计算实际内容大小、padding以及背景大小

* `EXACTLY` 0x01 精确大小，View大小完全由父类指定，一般是设置了 `match_parent` 或者固定的大小

* `AT_MOST` 0x02 最大取值，View大小可以根绝内容大小和size取小，一般是设置了 `wrap_content` 

  这三种测试模式中 `UNSPECIFIED` 相对比较特殊，这个mode和 `LayoutParams` 没有太多关联，一般只用于 `ViewGroup` 在自身大小未确定的时候用于测量子 `View` 的大小。

#### ViewGroup的测量

`ViewGroup` 相比于普通 `View` 测量过程更为复杂，一方面更难确定内容大小，需要根据子 `View` 大小和子 `View` 的的排版方式才能确定内容大小。另一方面在确认子 `View` 需要根据自身的 `MeasureSpec` 和子 `View` 的 `LayourParams` 确定子 `View` 的 `MeasureSpec` 。

关于确认子 `View`  的 `MeasureSpec` 的一般性代码如下

```java
//ViewGroup自身的mode
switch (specMode) {
case MeasureSpec.EXACTLY:
    if (childDimension >= 0) {
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
    } else if (childDimension == LayoutParams.MATCH_PARENT) {
        resultSize = size;
        resultMode = MeasureSpec.EXACTLY;
    } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
    }
    break;

case MeasureSpec.AT_MOST:
    if (childDimension >= 0) {
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
    } else if (childDimension == LayoutParams.MATCH_PARENT) {
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
    } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
    }
    break;

case MeasureSpec.UNSPECIFIED:
    if (childDimension >= 0) {
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
    } else if (childDimension == LayoutParams.MATCH_PARENT) {
        resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
        resultMode = MeasureSpec.UNSPECIFIED;
    } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
        resultMode = MeasureSpec.UNSPECIFIED;
    }
    break;
}
```

这里主要考虑两个维度，一个是子 `View` 在 `LayoutParams` 中设置的参数，一个是 `ViewGroup` 自身的`MeasureSpec` 

* 当View设置固定大小时，一般子 `View` 的 `MeasureSpec` 的 `mode` 都是 `EXACTLY` 
* 当View设置 `MATCH_PARENT` 时，则和`ViewGroup` 的mode一致
* 当View设置为 `WRAP_CONTENT` 时，除非`ViewGroup`为`UNSPECIFIED` ,否则均为`AT_MOST`

这里简单介绍一下常用的几种 `ViewGroup` 的测量过程，这个过程和`ViewGroup` 本事的测量mode有关， 暂时不考虑 `UNSPECIFIED` 的情况。

* `FrameLayout`

  `FrameLayout` 的测量过程相对简单，最多会调用两次子View的 `measure` 方法，

  当 `FrameLayout` 大小确定即为 `EXACTLY` 则会按照上面的方式进行一次测量，如果是其他两种mode，则会在测量完成后获取最大的子 `View` 尺寸（可能是背景的尺寸）作为自己的最终尺寸，同时会将`match_parent` 的子View 重新测量一次

* LinearLayout

  `LinearLayout` 根据情况，可能会进行2-3次测量，这个只分析横向测量过程

  1. 第一个分支在于 `LinearLayout` 本身宽度为 `EXACTLY` 时，如果没有`baselineAligned` 的属性可跳过第一次的测量，否则会将有 `weight` 的View的 `LayoutParams`设置为`WRAP_CONTENT` 进行第一次测量，第一次测量主要确认最大高度和总宽度（分为有`weight` 的View的总宽度和无 `weight` 的 View的总宽度）
  2. 第二个分支在于 `useLargestChild` ，这个属性主要针对 `weight` ，在此属性下有`weight` 的最小宽度为其他最大的子 `View` ,

默认情况下 `onMesure`  

会通过 `getDefaultSize` 计算一个默认尺寸

```java
 protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

