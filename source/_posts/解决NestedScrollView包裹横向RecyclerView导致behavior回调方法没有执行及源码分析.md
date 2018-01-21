layout: android
title: 解决NestedScrollView包裹横向RecyclerView导致behavior回调方法没有执行及源码分析
date: 2018-01-21 12:33:43
tags:
---
### 前言
如题，现在有一种behavior的使用场景：NestedScrollView下面包裹横向的RecyclerView，behavior的滚动回调方法不执行。详细可见[demo](https://github.com/fly7632785/BehaviorScrolltest),  建议最好clone下来自己试一试，因为你总有一天会用到behavior！
看看问题
![滚动下面bottomView没有跟着动](http://upload-images.jianshu.io/upload_images/1311457-7f39e4338cd244cd.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 先来看看demo的布局层级
￼![main_activity](http://upload-images.jianshu.io/upload_images/1311457-0af6bbe2a89e7c5c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
CoordinatorLayout包含两个子View: Viewpager和View(注入behavior关联滚动的view)
- 再看看viewpager_item
![viewpager_item](http://upload-images.jianshu.io/upload_images/1311457-3ee1618cb7a76f75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
里面是一层NestedScrollView，里面包含几个子Linear, Linear里面包裹横向的RecyclerView
- 最终层级图

![效果希望滚动里面的nestedscrollView然后显示和隐藏bottomView](http://upload-images.jianshu.io/upload_images/1311457-cd62403cdca36fc0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个层级还是简化后的demo的，实际开发中我们遇到的情况比这个更加复杂，但是就算层级再多再复杂，只要符合behavior的使用规则，那么一切皆可以实现。
- 再看看behavior
```
public class MyBehavior extends CoordinatorLayout.Behavior<View> {
    private boolean isHide = false;
    public MyBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean onStartNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull View child, @NonNull View directTargetChild, @NonNull View target, int axes, int type) {
        return true;
    }

    @Override
    public void onNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull View child, @NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int type) {
        Log.e("test", "onNS");
        if(dyConsumed >0 ) {
            if (!isHide) {
                child.offsetTopAndBottom(child.getHeight());
                isHide = true;
            }
        }else {
            if(isHide){
                child.offsetTopAndBottom(-child.getHeight());
                isHide = false;
            }
        }
    }
}
```
也超级简单就是判断一下滚动方向，然后显示和隐藏bottomView而已。
### 但是
我们这样简单的代码却有着问题，我们实际运行发现，貌似滚动的关联“不太灵敏”，打log发现，有时候onNestedScroll方法不会调用。这是为什么呢？
### 问题
于是提出两个问题：
**1、为什么onNestedScroll方法不会调用？
2、为什么让RecyclerView设置setNestedScrollingEnable(false)就能够正常使用？**

另外后面会进行更深层次的源码分析，附加几个问题：
**1、对于如果onIntercept返回true拦截了，交给onTouchEvent去处理，具体体现在何处？
2、判断子View是否能够接收事件从哪里体现？
3、另外一个比较重要的方法dispatchTransformedTouchEvent干什么用的？
4、viewGroup和view的dispatch返回false，会直接回溯到parent的onTouchEvent，这个又在哪里体现？
5、viewGroup重写了dispatch但是没有调用super, 那么它在哪里调用自己的onTouch的呢？**

### 正题
######  首先我们解决第一个问题，**“为什么onNestedScroll没有调用？”**
这需要大家对behavior有一定的了解，我们都知道coordinatorLayout和behavior联合使用可以实现许多花哨的效果，很牛逼。

behavior的工作原理就是:
1、coordinatorLayout下面的所有子view(包含子孙view),实现了滚动接口(包括NestedScrollingChild、NestedScrollingParent等等)的view, 如果有滑动事件的消耗，就会一层一层向上传递，直到coordinatorLayout
2、然后coordinatorLayout再对注入了behavior的子View传递滚动回调事件，这样，behavior就能拿到滚动的值，进而进行对View的一些关联滚动操作
如果用最通俗的例子来讲就是：
父亲是CoordinatorLayout，它有两个儿子，一个是NestedScrollView，一个是BottomView，behavior绑在BottomView身上（父亲比较偏爱他）。NestedScrollView发年终奖了（滚动了），发了红包给父亲（通知给了父亲），然后父亲又把钱分给了喜爱的儿子BottomView（父亲又通知了BottomView）
贴点重要代码
recycler->linear->nestedScroll->coordinator:
![传递过程，其他非滚动view“透明”](http://upload-images.jianshu.io/upload_images/1311457-8a6da7866ef19d91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![异常中返回false](http://upload-images.jianshu.io/upload_images/1311457-7890bfac9e10c40a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



为什么onNestedScroll没有回调呢？
***PS: 这里的源码是对应26的，support是26.1***
通过在代码里面打断点发现：
- ![RecyclerView中自己消费了consumedY](http://upload-images.jianshu.io/upload_images/1311457-32e7ccd4255cc480.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RecyclerView中自己消费了consumedY，uncomsumed = y - consumeY = 0 ，然后
![NestedScrollView中拿到的dyUnConsumed为0](http://upload-images.jianshu.io/upload_images/1311457-8b373d479b685fd3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

NestedScrollView中拿到的dyUnConsumed为0，调用dispatchNestedScroll方法也就传入0
![NestedScrollingChildHelper中if分支进不去](http://upload-images.jianshu.io/upload_images/1311457-b1f5a63b62fd33e0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样的话，NestedScrollingChildHelper中if分支进不去，就没法向上层传递消费的y值（相当于它并没有滚动），ViewParentCompat.onNestedScroll没法调用，所以没能传递到顶层的CoordinatorLayout，自然behavior里面也不会收到回调了。

从源码上来看是这样的，如果从宏观上来讲，其实就是RecyclerView和NestedScrollView的事件处理有冲突，RecyclerView消费了事件，从而NestedScrollView没能把自己消费的事件往上传递。
按道理，我们都知道，如果竖向的RecyclerView和NestedScrollView或者ScrollView联合使用的话（虽然，这样联合使用没有意义，也不建议这样做），会出现事件冲突。但是，横向的RecyclerView和NestedScrollView一起使用，在事件处理上面是没有问题的，没有冲突，但是，在使用到behavior，希望nestedScrollView能够把自己滚动消费的事件往上传递的时候就会出问题了。
（我们都希望behavior的使用是在没有嵌套滚动冲突的情况下，兄弟滚动，然后父亲知道，父亲通知另外一个兄弟做出相应的行为，而如果是子孙滚动，往上传给父亲，这期间出了问题，就没法正常工作了）

###### 接着第二个问题，“为什么让RecyclerView设置setNestedScrollingEnable(false)就能够正常使用？”
看看效果
![OK,可能转gif帧率不够有点卡](http://upload-images.jianshu.io/upload_images/1311457-9ffc92ff78aa70cd.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二个问题就需要大家对于事件分发机制有一定的了解，这里就大致贴张图。
![事件分发机制](http://upload-images.jianshu.io/upload_images/1311457-e3d5fe3ad2c86848.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
另外，贴几个认为比较不错的链接：
1、[图解 Android 事件分发机制](http://www.jianshu.com/p/e99b5e8bd67b)
2、[Android事件分发机制详解：史上最全面、最易懂](http://www.jianshu.com/p/38015afcdb58)
3、[Android6.0源码解读之View点击事件分发机制](http://blog.csdn.net/mynameishuangshuai/article/details/52912641)
4、[Android 事件分发机制-试着读懂每一行源码-View](http://www.jianshu.com/p/383bae6b6487)
5、[ScrollView与头+RecycleView嵌套冲突源码分析](http://www.jianshu.com/p/7e17e48e6baf)

我们看看setNestedScrollingEnabled
```
// RecyclerView
   @Override
    public void setNestedScrollingEnabled(boolean enabled) {
        getScrollingChildHelper().setNestedScrollingEnabled(enabled);
    }
```
调用了辅助类
```
// NestedScrollingChildHelper
   public void setNestedScrollingEnabled(boolean enabled) {
        if (mIsNestedScrollingEnabled) {
            ViewCompat.stopNestedScroll(mView);
        }
        mIsNestedScrollingEnabled = enabled;
    }
```
辅助类设置mIsNestedScrollingEnabled为false，并且调用了 ViewCompat.stopNestedScroll(mView);传入了自己
```
// NestedScrollingChildHelper
 public boolean isNestedScrollingEnabled() {
        return mIsNestedScrollingEnabled;
    }
```
这样isNestedScrollingEnabled返回false了，以后behavior的回调方法里面的if(isNestedScrollingEnabled())就进不去了
接着：
```
//ViewCompat
 public static void stopNestedScroll(@NonNull View view) {
        IMPL.stopNestedScroll(view);
    }
```
这里，ViewCompat就是一个兼容类，兼容各个版本api的使用，因为有一些新版本的api，实现的是NestedScrollingParent2等方法。
```
//ViewCompat
 public void stopNestedScroll(View view) {
            if (view instanceof NestedScrollingChild) {
                ((NestedScrollingChild) view).stopNestedScroll();
            }
        }
```
这里是相当于调用view的stopNestedScroll，也就是RecyclerView的。
```
//RecyclerView
  @Override
    public void stopNestedScroll() {
        getScrollingChildHelper().stopNestedScroll();
    }
```
```
//NestedScrollingChildHelper
   public void stopNestedScroll() {
        stopNestedScroll(TYPE_TOUCH);
    }
```
```
//NestedScrollingChildHelper
 public void stopNestedScroll(@NestedScrollType int type) {
        ViewParent parent = getNestedScrollingParentForType(type);
        if (parent != null) {
            ViewParentCompat.onStopNestedScroll(parent, mView, type);
            setNestedScrollingParentForType(type, null);
        }
    }
```
这里就比较重要了，这里通过getNestedScrollingParenForType获得了parent，然后调用了ViewParentCompat.onStopNestedScroll(parent, mView, type);
```
//viewParentCompat
  public static void onStopNestedScroll(ViewParent parent, View target, int type) {
        if (parent instanceof NestedScrollingParent2) {
            // First try the NestedScrollingParent2 API
            ((NestedScrollingParent2) parent).onStopNestedScroll(target, type);
        } else if (type == ViewCompat.TYPE_TOUCH) {
            // Else if the type is the default (touch), try the NestedScrollingParent API
            IMPL.onStopNestedScroll(parent, target);
        }
    }
```
这个方法会又调用IMPL.onStopNestedScroll(parent, target);这样类似的方法其实就是把事件一层一层往上传，当然，其他onPreNestedScroll、onNestedScroll这些也都是这样的。
```
//NestesScrollView
  @Override
    public void onStopNestedScroll(View target) {
        mParentHelper.onStopNestedScroll(target);
        stopNestedScroll();
    }
```
我们又看NestedScrollView里面的onStopNestedScroll
```stopNestedScroll();```是继续往上调用传递
``` mParentHelper.onStopNestedScroll(target);```就比较关键了
```
//NestedScrollingParentHelper
  public void onStopNestedScroll(@NonNull View target) {
        onStopNestedScroll(target, ViewCompat.TYPE_TOUCH);
    }
```
```
 public void onStopNestedScroll(@NonNull View target, @NestedScrollType int type) {
        mNestedScrollAxes = 0;
    }
```
这个就关键了，mNestedScrollAxes = 0
```  public int getNestedScrollAxes() {
        return mNestedScrollAxes;
    }
```
这个方法返回0了，看看它在哪被调用
```
//NestedScrollVIew#onIntercept#move
    final int yDiff = Math.abs(y - mLastMotionY);
                if (yDiff > mTouchSlop
                        && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
                    mIsBeingDragged = true;
                    mLastMotionY = y;
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                    mNestedYOffset = 0;
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }
                break;
```
在move的时候，它返回为0，那么走入分支的话，mIsBeingDragged =true
onInterceptTouchEvent就返回true, 就会拦截了。
这就说明，在move的时候，nestedScrollView就完全拦截了事件，里面的子孙view（包括横向的RecyclerView就不会有事件了，更不用谈什么它自己消费掉了consumeY，NestedScrollView自己全权处理了），这样的话它自己的滚动事件就能够再往上一直传递到coordinatorLayout，然后behavior也就肯定能够执行回到方法了！
啊，原来如此，恍然大悟！

### 5个小问题
前面两个大问题终于解决了，下面来搞清楚后面提的那5个小问题。
**1、对于如果onIntercept返回true拦截了，交给onTouchEvent去处理，具体体现在何处？
2、判断子View是否能够接收事件从哪里体现？
3、另外一个比较重要的方法dispatchTransformedTouchEvent干什么用的？
4、viewGroup和view的dispatch返回false，会直接回溯到parent的onTouchEvent，这个又在哪里体现？
5、viewGroup重写了dispatch但是没有调用super, 那么它在哪里调用自己的onTouch的呢？**
这几个问题全是关于事件分发的，大家可以把那几个链接的文章都看了，如果还不能解决，那么再往下看。

```
//ViewGroup#dispatchTouchEvent  代码有省略
 @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
      boolean handled = false;
            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
    ##### 重点
              // 这里mFirstTouchTarget置为null
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
   ##### 重要 if分支1
            if (!canceled && !intercepted) {
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

    ##### 重点
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            resetCancelNextUpFlag(child);
    ##### 重点
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
    ##### 重点
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }
            }

            // Dispatch to touch targets.
   ##### 重要 if分支2
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
    ##### 重点
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }
      ...
        return handled;
    }
```
```
//ViewGroup#dispatchTransformedTouchEvent  代码有省略
 private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
              handled = child.dispatchTouchEvent(event);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
         handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```
#### 1 、对于如果onIntercept返回true拦截了，交给onTouchEvent去处理，具体体现在何处？
如果onIntercep返回true，那么interceped变量为true，那么不会走入【重要 if分支1】（里面分发事件，设置mFirstTouchTarget等），mFirstTouchTarget依旧为null, 于是走入【重要 if分支2】的```dispatchTransformedTouchEvent(ev, canceled, null,TouchTarget.ALL_POINTER_IDS)``` 并且传入child为null, 
在dispatchTransformedTouchEvent中如果child为null,就会走super.dispatch, super就是view，这样的话，就会走view的dispatch（view本身的dispatch会调onTouch），就会走到onTouch去了
#### 2、判断子View是否能够接收事件从哪里体现？
在viewgroup的dispatch中的走入if分支之后，里面有个判断
 ```
if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
```
有个方法canViewReceivePointerEvents，里面主要是判断是否Visible
isTransformedTouchPointInView主要是判断事件的位置是否在子VIew的区域内
如果不行，就continue，后续的事件分发就不进行
#### 3、另外一个比较重要的方法dispatchTransformedTouchEvent干什么用的？
dispatchTransformedTouchEvent主要就是对于事件分发的处理，比如什么时候调用自己的super.dispatch，什么时候调用child.disaptch分发给子View, 这个判断方法的主要根据就是child是否为null, 而这个又跟mFirstTouchTarget有关联
#### 4、viewGroup和view的dispatch返回false，会直接回溯到parent的onTouchEvent，这个又在哪里体现？
这个问题也跟第1个问题有点类似，如果子View的dispatch返回false，那么dispatchTransformedTouchEvent的handled就会是false返回，然后【重要 if分支1】就走不进去，addTouchTarget这个方法也不执行（主要是给mFirstTarget赋值）
```
  private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
        final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }
```
前面说了，如果mFirstTarget为null, 【重要 if分支2】就会进入dispatchTransformedTouchEvent的时候传入为null的child, 这样就会调用super.dispatch，就是view的dispatch，然后就调用了onTouch咯

#### viewGroup重写了dispatch但是没有调用super, 那么它在哪里调用自己的onTouch的呢？
如果看到这，希望第5个问题我已经不用解释了，因为前面4个问题已经把它囊括在内了。

### 最后
为了解决这个问题，最近一直在源码的黑洞里遨游，打了N个断点来回跳，梳理逻辑。最后一句，最终想要深刻地理解事件分发机制、behavior机制这些玩意儿，RTFSC。
