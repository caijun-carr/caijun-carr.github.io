---
layout: post
title: 触摸事件传递分析
date: 2017-06-11
categories: blog
tags: [android,触摸事件,原理分析]
description: 分析Android在View和ViewGroup上触摸事件传递的原理

---

### MotionEvent

MotionEvent是一个触摸动作的封装，里面包含了触摸动作的类型，如：ACTION_DOWN、ACTION_MOVE、ACTION_UP等
当前触摸动作的坐标 X、Y

### 事件序列
一个事件序列一般由一个按下事件，0个或者N个滑动事件，一个抬起事件组成
正常情况下一个事件序列所有事件都只分发给一个View处理

某个View一旦决定拦截某次事件序列（ViewGroup 返回onInterceptTouchEvent为true），那么这个事件序列都只能由它来处理
相反的一个view如果开始处理事件（执行onTouchEvent），如果它在处理ACTION_DOWN时返回为false即不消费掉，那么同一个事件序列中的其他事件也不会给这个view处理了

### 事件传递的主要函数

先大致了解下需要自己制定传递和执行触摸的规则需要重写使用的方法，后面会详细分析其原理：
* dispatchTouchEvent 主要用于分发事件给子view
* onInterceptTouchEvent 当前的ViewGroup是否拦截掉（不继续向子View传递）事件
* onTouchEvent 主要用于处理触摸事件

除了这三个比较常用来重写的函数外，还有一个函数requestDisallowInterceptTouchEvent();

当这个函数的参数为true的时候代表调用这个函数的view告诉他所有的父view，你不允许拦截掉这个事件序列

### 传递的原理分析

先上个图：
![触摸事件传递的原理图](http://oogbkd3ln.bkt.clouddn.com/touch_principle.png)
这个图表达的不够完善，比如就没有上图所说的requestDisallowInterceptTouchEvent方法对应的原理，因为加在图上比较难表现出来，所以先分析这个方法。

先上源码：
``` java
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) { // 这里的mFirstTouchTarget是一个典型的单向链表结构的第一个节点（触摸事件后续传递给谁等信息就是靠这个对象来传递的）
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
```
如果按下或者已经确定事件将传递给谁，就会首先检查FLAG_DISALLOW_INTERCEPT标记的值。requestDisallowInterceptTouchEvent方法的实际就是改变这个标记的值的。如果子view确认不允许父view拦截（调用requestDisallowInterceptTouchEvent传入true的参数）则直接让intercepted = false;父view不会再调用onInterceptTouchEvent方法检查当前view是否需要拦截事件。
当这个flag没有被标记，则调用onInterceptTouchEvent方法检查当前view是否拦截掉事件不再往子view传递，onInterceptTouchEvent返回为true的时候 intercepted = true；
```java
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        // We're already in this state, assume our ancestors are too
        return;
    }

    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

    // Pass it up to our parent
    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```
requestDisallowInterceptTouchEvent方法的原理直接递归调用所有父view的FLAG_DISALLOW_INTERCEPT对应的值。

得到intercepted的值以后跟着源码继续往下看：
```java
// Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
                    ...
            if (!canceled && !intercepted) {
                    ... // 处理传递到子view的逻辑
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                  ... // 处理有对应的mFirstTouchTarget的nextTouchTarget
            }
```
如果检测到不需要调用cancel而且当前view不拦截掉哦事件则事件会尝试往子view传递（里面的逻辑后面再看），这里先假设上面的intercepted的值为true 则不会尝试传递到子view。如果当前是ACTION_DOWN事件则mFirstTouchTarget为null。
则会执行dispatchTransformedTouchEvent方法（注意这里的第三个参数传递null）。继续看源码：
```java
/**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
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

        ...

    }
```
这个函数可以看到当child传递过来为null时调用了 handled = super.dispatchTouchEvent(event);
而ViewGroup的super就是View。调用super.dispatchTouchEvent(event)函数则把当前的ViewGroup当做View来处理这次的点击事件。View里的dispatchTouchEvent函数相对比较简单就简单贴源码了。大致流程是 先检测是否有onToucListener的监听 如果有切OnTouch方法返回的为true则直接返回true。不返回true或者没有监听则调用改View的onTouEvent方法。如果其返回true则dispatchTouchEvent函数返回true，否则返回false。 简单贴下源码，如下：
```java
/**
     * Pass the touch screen motion event down to the target view, or this
     * view if it is the target.
     *
     * @param event The motion event to be dispatched.
     * @return True if the event was handled by the view, false otherwise.
     */
    public boolean dispatchTouchEvent(MotionEvent event) {

      ...

      ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }

        ...

        return result;
    }

    public boolean onTouchEvent(MotionEvent event) {

      ...

      if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {

                  ... // view默认的处理touch事件的逻辑包裹执行onClickListener的onCLick事件  和处理
                      // 触摸时View的反馈（5.0以下的按下背景。5.0以上水波纹）

              return true;
        }
        return false;
    }
```

上面分析了intercepted = true的情况。一般默认的情况下是不会拦截的，下面分析下如果不为true的情况。
```java
// Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {

              ...

              final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                                    ...

                                    newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
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
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }
                }
            }
```
如果不拦截且不cancle则进入这个if里。这里面会遍历所有子view，如果这个子view可以接受当前touch事件则会调用
dispatchTransformedTouchEvent函数。这个函数上面分析过，第三个参数为child不为空时会调用child的dispatchTouchEvent方法这样就将事件传递给了子View，子View就会重复执行上面的操作事件也就传递下去了。继续执行上面的循环。如果某个子View的函数dispatchTouchEvent返回为true（表示该子View会消费掉当前事件）则会执行newTouchTarget = addTouchTarget(child, idBitsToAssign);
```java
  newTouchTarget = addTouchTarget(child, idBitsToAssign);


  /**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
    private TouchTarget addTouchTarget(View child, int pointerIdBits) {
        TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }
```
mFirstTouchTarget单链表结构得到传递，后续事件得到传递。
