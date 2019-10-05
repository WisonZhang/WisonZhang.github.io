---
title: View触摸事件分发机制
date: 2018-03-10 18:24:11
tags: Android
---

View的触摸事件很多文章都写过了，但是大体都是做了流程的分析，自己对于各种状态对于触摸事件的影响觉得还要总结一下，于是写了这篇文章。

### 事件的开始
View的触摸事件分发是从DecorView开始的，我们先看一下DecorView的代码。
``` Java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```
可以看到，当Callback不为空时，调用的是callback的dispatchTouchEvent，那Callback是什么呢？从代码可知，mWindow是PhoneWindow，而PhoneWindow是在Activity的attch方法中初始化的，再看下Activity的attach关于PhoneWindow初始化相关的代码。
``` Java
mWindow = new PhoneWindow(this, window, activityConfigCallback);
mWindow.setWindowControllerCallback(this);
mWindow.setCallback(this);
```
可以看到 mWindow.setCallback(this) ，这个函数的作用是在Window类中设置了回调，当DecorView的dispatchTouchEvent方法中mWindow.getCallback()时也就是调用了Activity的dispatchTouchEvent，从Activity我们也可以看到其实现了Window.Callback的接口。接下来看下Activity的dispatchTouchEvent代码。
``` Java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
从代码可知，当事件为Down时，会先回调onUserInteraction()，该方法是个空方法，当我们需要在事件分发前收到回调，可以复写该方法。之后会调用superDispatchTouchEvent，如果返回是false，则回调Activity的onTouchEvent。自然的，如果我们重写Activity的dispatchTouchEvent且不调用super.dispatchTouchEvent(ev)，事件是不会往下分发的。再来看getWindow().superDispatchTouchEvent(ev)，该方法是Window的抽象方法，实现在PhoneWindow中。
``` Java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
可以看到调用了DecorView的superDispatchTouchEvent，再到DecorView代码中查看。
``` Java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

### ViewGroup 事件分发
DecorView是ViewGroup，super.dispatchTouchEvent(event)就是调用了ViewGroup的dispatchTouchEvent。因此我们可以总结下，触摸事件会先由DecorView处理，再回到DecorView并交由给ViewGroup的dispatchInputEvent，从ViewGroup开始了事件的分发。

#### onInterceptTouchEvent
ViewGroup是View的子类，ViewGroup重写了View的dispatchTouchEvent，同时ViewGroup提供了onInterceptTouchEvent方法供外部使用，用于判断是否拦截事件，当拦截时将不会往下分发事件，且回调ViewGroup的onTouchEvent。由于dispatchInputEvent代码比较多，因此将省略与拦截无关的代码。
``` Java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    final boolean intercepted;
    if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        if (!disallowIntercept) {
            intercepted = onInterceptTouchEvent(ev);
            ev.setAction(action);
        } else {
            intercepted = false;
        }
    } else {
        // 不为Down事件且mFirstTouchTarget为空，分发也没有意义，所以直接拦截
        intercepted = true;
    }
    ...
    // 当手势不是取消且不拦截时则执行分发，稍后再介绍
    if (!canceled && !intercepted) {
        // 注释1
        ...
    }
    
    // mFirstTouchTarget不为空的条件是，进入上面分发方法且有View消耗时
    if (mFirstTouchTarget == null) {
        // 此时将由ViewGroup接收事件，注：不是消耗
        handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
    } else {
        ...
    }
}

// 外部可以重写拦截事件的方法决定拦截
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
            && ev.getAction() == MotionEvent.ACTION_DOWN
            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
            && isOnScrollbarThumb(ev.getX(), ev.getY())) {
        return true;
    }
    return false;
}

// 修改标志位
@Override
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        return;
    }
    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }
    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}

```
由代码我们可以看到，在ViewGroup的dispatchInputEvent中会判断是否是Down事件且标志位是否允许拦截，我们可以使用ViewGroup的requestDisallowInterceptTouchEvent方法来设置标志位。当都满足时会判断onInterceptTouchEvent的返回值来决定是否拦截，如果拦截则会调用dispatchTransformedTouchEvent，最终会调用父类，即View的dispatchTouchEvent方法进行分发，最后会调用View/ViewGroup的onTouchEvent方法。

#### dispatchTransformedTouchEvent
这是一个很重要的方法，在分发过程中会频繁调用到该方法，该方法的作用主要是进行事件的传输转换，根据传入的参数，再调用View的dispatchTouchEvent方法，实现了事件的递归调用。
``` Java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
    ...
    // 传入空时调用ViewGroup的super.dispatchTouchEvent，即View.dispatchTouchEvent
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        // 子View的dispatchTouchEvent
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    
    transformedEvent.recycle();
    return handled;
}
```
可以看到当传入参数child为null时会直接调用super.dispatchTouchEvent，此时代表着将直接回调ViewGroup的onTouchEvent，由ViewGroup接收事件。如果不为空则调用child.dispatchTouchEvent，child为View时则调用View的dispatchTouchEvent，为ViewGroup则调用ViewGroup的dispatchTouchEvent，然后再重新走dispatchTouchEvent的逻辑继续分发事件，实现了触摸事件的传递。

#### dispatchTouchEvent
不拦截时会进入在onInterceptTouchEvent介绍时注释1的代码，这段代码会比较长，我们先看看代码。
``` Java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    // 当手势不是取消且不拦截时则执行分发
    if (!canceled && !intercepted) {
        //把事件分发给所有的子视图，寻找可以获取焦点的视图
        View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                ? findChildWithAccessibilityFocus() : null;
                
        if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                    : TouchTarget.ALL_POINTER_IDS;
          
            removePointersFromTouchTargets(idBitsToAssign);
            final int childrenCount = mChildrenCount;
            if (newTouchTarget == null && childrenCount != 0) {
                final float x = ev.getX(actionIndex);
                final float y = ev.getY(actionIndex);
                final ArrayList<View> preorderedList = buildTouchDispatchChildList
                final boolean customOrder = preorderedList == null
                        && isChildrenDrawingOrderEnabled();
                        
                // 获取子View，并倒序遍历
                final View[] children = mChildren;
                for (int i = childrenCount - 1; i >= 0; i--) {
                    final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                    final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);
                   
                    // 是否获取用户焦点
                    if (childWithAccessibilityFocus != null) {
                        if (childWithAccessibilityFocus != child) {
                            continue;
                        }
                        childWithAccessibilityFocus = null;
                        i = childrenCount - 1;
                    }
                    
                    // 是否可见，是否在点击区域
                    if (!canViewReceivePointerEvents(child)
                            || !isTransformedTouchPointInView(x, y, child, null)) 
                        ev.setTargetAccessibilityFocus(false);
                        continue;
                    }
                    
                    // 已经在目标中
                    newTouchTarget = getTouchTarget(child);
                    if (newTouchTarget != null) {
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                        break;
                    }
                    
                    resetCancelNextUpFlag(child);
                    // 分发事件
                    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAs
                        mLastTouchDownTime = ev.getDownTime();
                        if (preorderedList != null) {
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
                        // mFirstTouchTarget赋值的地方
                        newTouchTarget = addTouchTarget(child, idBitsToAssign);
                        // 标记消费Down事件
                        alreadyDispatchedToNewTouchTarget = true;
                        break;
                    }
                    
                    ev.setTargetAccessibilityFocus(false);
                }
                if (preorderedList != null) preorderedList.clear();
            }
            if (newTouchTarget == null && mFirstTouchTarget != null) {
                newTouchTarget = mFirstTouchTarget;
                while (newTouchTarget.next != null) {
                    newTouchTarget = newTouchTarget.next;
                }
                newTouchTarget.pointerIdBits |= idBitsToAssign;
            }
        }
    }
    
    if (mFirstTouchTarget == null) {
        ...
    } else {
        TouchTarget predecessor = null;
        TouchTarget target = mFirstTouchTarget;
        while (target != null) {
            final TouchTarget next = target.next;
            // 在Down事件消费了
            if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                handled = true;
            } 
            // MOVE,UP等事件的回调
            else {
                final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
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
}
```
ViewGroup会从最底层的Child，即最后加入的子View，开始遍历并调用dispatchTransformedTouchEvent。像从FrameLayout来讲的话，就是最顶层的View会先收到触摸事件的分发。在遍历过程中，会判断是否在焦点中，canViewReceivePointerEvents判断是否可见，isTransformedTouchPointInView判断是否在点击区域内。getTouchTarget判断是否已经开始接收了触摸事件，是则跳出循环。经过这些判断后才使用dispatchTransformedTouchEvent进行事件的传输，当消耗事件时，即dispatchTransformedTouchEvent返回true，调用addTouchTarget将View记录下来，此时也是mFirstTouchTarget的赋值，在接下来的逻辑中，将不是Down事件的传递给mFirstTouchTarget，再进行事件的回调，这也是我们常说的，只有消耗了Down事件，才会接收到其余事件的回调。

### View 事件分发
我们总说ViewGroup比View多了onInterceptTouchEvent，从上面分析可知，是因为ViewGroup在分发事件时提供了这个方法控制拦截，子View可以进行重写。在View中的dispatchTouchEvent中则没有这部分的逻辑。当ViewGroup拦截或者没用子View消耗时最终也会调用View的dispatchTouchEvent，因此View的事件分发也很重要。在这里我们还要探讨各种状态，如enable，onTouchListener对View分发事件的影响。

#### dispatchTouchEvent
View的dispatchTouchEvent不是很长，先直接贴上代码。
``` Java
public boolean dispatchTouchEvent(MotionEvent event) {
    if (event.isTargetAccessibilityFocus()) {
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        event.setTargetAccessibilityFocus(false);
    }
    boolean result = false;
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }
    final int actionMasked = event.getActionMasked();
    
    // Down事件停止滚动
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        stopNestedScroll();
    }
    
    // 过滤安全触摸事件
    if (onFilterTouchEventForSecurity(event)) {
    
        // 回调onTouchListener
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        
        // 回调onTouchEvent
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }
    
    // 处理抬起或取消的事件
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }
    return result;
}
```
从代码我们可以知道，onTouchListener会优先于onTouchEvent先收到回调。View的dispatchInputEvent方法中，会判断当前View是否处于Enable状态且onTouchListener不为空，符合则回调onTouchListener，而只有当onTouchListener回调返回True，onTouchEvent才不会接收到事件，否则依然能接收到回调。

#### onTouchEvent
View的onTouchEvent默认做的事情很简单，就是判断各种状态是否符合，如果符合则消费事件。举个例子，当我们设置View.onClickListener的时候，默认情况下就等于消耗了事件了，具体我们可以在以下的代码看到。当然，如果重写了View的onTouchEvent，且没有调用super.onTouchEvent直接返回了false，是不会消耗事件的。
``` Java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();
    
    // 点击状态
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
            
    // 当为Disable时，只要点击状态true还是会消费事件
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        return clickable;
    }
    
    // 触摸事件代理
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    
     // 当为Enable时，只要点击状态true还是会消费事件
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                    handleTooltipUp();
                }
                if (!clickable) {
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
                }
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }
                    if (prepressed) {
                        setPressed(true, x, y);
                    }
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        removeLongPressCallback();
                        if (!focusTaken) {
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            // 调用onClickListener
                            if (!post(mPerformClick)) {
                                performClickInternal();
                            }
                        }
                    }
                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }
                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        mUnsetPressedState.run();
                    }
                    removeTapCallback();
                }
                mIgnoreNextUpEvent = false;
                break;
            case MotionEvent.ACTION_DOWN:
                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                }
                mHasPerformedLongPress = false;
                if (!clickable) {
                    checkForLongClick(0, x, y);
                    break;
                }
                if (performButtonActionOnTouchDown(event)) {
                    break;
                }
                
                // 是否处于可滚动的视图内
                boolean isInScrollingContainer = isInScrollingContainer();
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    // 当处于可滚动视图内，则延迟TAP_TIMEOUT，再反馈按压状态，
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout())
                } else {
                    // 立即反馈
                    setPressed(true, x, y);
                    checkForLongClick(0, x, y);
                }
                break;
            case MotionEvent.ACTION_CANCEL:
                if (clickable) {
                    setPressed(false);
                }
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                break;
            case MotionEvent.ACTION_MOVE:
                if (clickable) {
                    drawableHotspotChanged(x, y);
                }
                if (!pointInView(x, y, mTouchSlop)) {
                    removeTapCallback();
                    removeLongPressCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        setPressed(false);
                    }
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                }
                break;
        }
        return true;
    }
    return false;
}
```

### 总结

- 触摸事件是以一个U型的责任链模式来分发的，从Activity的dispatchTouchEvent开始往ViewGroup分发，再以ViewGroup的子View以倒序遍历的方式进行分发。当没有任何View接收事件时，则往回传递，最终回到Activity。
- ViewGroup提供了onInterceptTouchEvent来拦截事件，外部可用requestDisallowInterceptTouchEvent来设置标志位阻止拦截。
- 当一个ViewGroup没有子View消耗事件时，即mFirstTouchTarget为null时，会分发到ViewGroup的onTouchEvent，如果ViewGroup也不消耗，则继续抛回上一层。
- 当一个View消耗事件后，其余View接收不到Move和Up等其他事件，因为mFirstTouchTarget已经不为null，所以事件会直接给到mFirstTouchTarget。但是ViewGroup的dispatchTouchEvent和onInterceptTouchEvent在Move和Up事件依然会接收到回调。
- 当View在onTouchEvent返回true消耗事件后，事件停止往下传递，且上层的onTouchEvent也不会收到回调。
- 在当前View是Enable的情况下，onTouchListener会优先于onTouchEvent收到回调，只有onTouchListener返回true，onTouchEvent才不会收到回调。
- 在View默认onTouchEvent方法中（ViewGroup没有对onTouchEvent进行重写），当View.setClickable(true)或者View.setOnClickListener()等表示可点击时，会消耗触摸事件。

