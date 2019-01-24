---
tags: Android 
read: 1056   
---
@[toc]
# 前言
----------
&emsp;&emsp;在前面的两个章节中，我们已经分析过关于Android事件处理机制的过程，特别是关于View的触摸、点击、长按之间的处理过程的分析，如果对这方面还不熟悉的读者请先阅读：
[Android开发知识（七）：Android事件处理机制：事件分发、传递、拦截、处理机制的原理分析（上）](http://blog.csdn.net/lc_miao/article/details/78251504)
[ Android开发知识（八）：Android事件处理机制：事件分发、传递、拦截、处理机制的原理分析（中）](http://blog.csdn.net/lc_miao/article/details/78263102)

&emsp;&emsp;在本章节是我们分析Android事件处理机制的<下>篇，我们将分析关于手指从触摸屏幕到离开屏幕期间，从顶级ViewGroup到View的事件传递过程。
# Activity的事件分发过程
&emsp;&emsp;实际上，**当我们手指触摸屏幕的时候，事件最先是传递给当前的Actvity，由Actvity的dispatchTouchEvent方法来分发事件，而Actvity会将事件传递给Window对象来分发，Window对象再传递给Decor View,而Decor View则是我们在Actvity中通过setContentView后所设置的布局的父容器**，通过getWindow().getDecorView().findViewById(android.R.id.content).getChildAt(0)这个方式就能获取到Activity所设置的布局。

&emsp;&emsp;我们不要单纯瞎说，还是从源码解析入手，才具有说服意义。事件从Actvity的dispatchTouchEvent开始，该方法的源码如下：

```
 public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
	        //这是一个空方法
            onUserInteraction();
        }
        //传递给Window去分发事件
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        //只有传递下去的事件返回false没有被消耗，则会调用onTouchEvent
        return onTouchEvent(ev);
    }
```
&emsp;&emsp;当ACTION_DOWN事件的时候，会先执行onUserInteraction();而onUserInteraction()方法是个空方法，事实上，当此activity在栈顶时，我们触屏、点击、或者按home，back，menu键等都会触发此方法，所以onUserInteraction()可以用于屏保。

&emsp;&emsp;接下来调用getWindow().superDispatchTouchEvent(ev)去分发事件，如果最终事件被消耗了，则直接返回true，否则Activity才会调用onTouchEvent方法自己来处理事件。
getWindow()是一个window对象。而Window类是抽象类，从Window类的注释里面说明了PhoneWindow类是Window类的唯一实现类。然而PhoneWindow存在于框架层中，在sdk中是没有源码的，因此我们看不到。想要查看源码则必须手动关联PhoneWindow，请读者自行百度。

&emsp;&emsp;PhoneWindow类的superDispatchTouchEvent方法源码如下：

```
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
  return mDecor.superDispatchTouchEvent(event);
}
```
说明window对象是把事件传给mDecor去分发，mDecor就是上面我们说的Decor View。Decor View的superDispatchTouchEvent源码如下：

```
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```
&emsp;&emsp;其实也是调用ViewGroup类的dispatchTouchEvent方法。

&emsp;&emsp;到了这里，我们已经验证了**事件的传递是：Activity->Window->decor View->ViewGroup**
其实到了Decor View这里就相当于事件传递到了我们在Activity中设置的布局父容器了。

&emsp;&emsp;接下来就是一层层的分发下来，如果遇到某一子ViewGroup设置了拦截，才会让事件分发停止，而让这个ViewGroup处理，否则的话会传递到触摸位置对应的View上（如果有View的话）。

&emsp;&emsp;而默认的ViewGroup到View的传递过程，我们在前文也讲过了，这里再贴出以一个传递的流程图，在dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent这三个重要的方法中：
![流程图](http://img.blog.csdn.net/20171018153206467?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
# View的事件传递过程
-------
&emsp;&emsp;再来回顾一下前文说的：如果View的mOnTouchListener被设置的话，则onTouch会调用，如果onTouch返回了true，则onTouchEvent不会被调用，而mOnClickListener被设置了话，只有onTouchEvent被调用了后才会被调用。而如果子View在onTouchEvent中不返回true的话，则表示不消耗事件，事件会回传给上一级ViewGroup的onTouchEvent，如果全部ViewGroup的onTouchEvent都不返回true则最终到达Activity的onTouchEvent方法。

&emsp;&emsp;好了，现在我们重点来分析顶级ViewGroup到View的事件传递过程的源码。

# ViewGroup#dispatchTouchEvent源码分析
-----
我们从dispatchTouchEvent方法分析，该方法源码比较多，我们就不全部贴出来，由于Android系统版本更新，源码有所改变，但是原理是不变的。我这里是基于Android6.0的源码来分析的。

&emsp;&emsp;展开dispatchTouchEvent的源码，前面几句代码是关于辅助点击的，我们暂时可以忽略掉，直接绕过，看接下来的代码：

```
 if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                  //从拦截标志位取得是否允许拦截
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
	                //判断是否需要拦截，默认是返回false
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                //不拦截
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                /*代码会走到这里，说明当前不是ACTION_DOWN事件或者是mFirstTouchTarget为null，说明没有一个子View要去处理ACTION_DOWN事件，导致mFirstTouchTarget还是空的，没有指向要处理事件的子View，所以接下来的其他事件，都不再继续分发下去了，而且拦截了事件让自己处理。
                */
                intercepted = true;
            }
```
&emsp;&emsp;从源码得知，ViewGroup会判断是否要拦截事件只会是在ACTION_DOWN的时候，或者mFirstTouchTarget！=null。mFirstTouchTarget其实要才从后面的代码才能得知其作用，在这里先说明一下他的作用：当事件被这个ViewGroup的某个子View处理时，mFirstTouchTarget会指向这个子View。也就是说如果这个ViewGroup不拦截事件的话，则事件交给了子View处理了后mFirstTouchTarget也就不为null了。

&emsp;&emsp;在if里面第一句代码：

```
final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
```
&emsp;&emsp;FLAG_DISALLOW_INTERCEPT是一个标记位，通过 requestDisallowInterceptTouchEvent(boolean disallowIntercept) 可以设置它，一般是用在子View里面。如果FLAG_DISALLOW_INTERCEPT被设置了后，ViewGroup就无法拦截ACTION_DOWN以外的其他事件，这是因为ACTION_DOWN事件会重置FLAG_DISALLOW_INTERCEPT标记位，我们还是来看看源码吧，在我们分析dispatchTouchEvent的这段代码的前面还有这么几句：

```
  // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                //重点代码：重置触摸状态
                resetTouchState();
            }
```

&emsp;&emsp;首先cancelAndClearTouchTargets方法会遍历清除所有的target，导致mFirstTouchTarget=null，然后在调用resetTouchState，重置触摸状态，点击进去看看resetTouchState的源码：

```

    /**
     * Resets all touch state in preparation for a new cycle.
     */
    private void resetTouchState() {
        clearTouchTargets();
        resetCancelNextUpFlag(this);
        //重置标志位
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        mNestedScrollAxes = SCROLL_AXIS_NONE;
    }
```
&emsp;&emsp;从源码得知resetTouchState会重置掉FLAG_DISALLOW_INTERCEPT标志位，所以说，当子View调用requestDisallowInterceptTouchEvent方法是不会影响到ViewGroup对ACTION_DOWN事件的拦截处理。

&emsp;&emsp;分析到这里我们可以得出一个结论：**当ViewGroup要拦截事件后，那么后续到来的事件会直接交给他处理，而不会再调用onInterceptTouchEvent询问是否拦截。**

&emsp;&emsp;因为onInterceptTouchEvent方法会被调用的话有两个重要条件：要么当前是ACTION_DOWN事件要么有子View处理了事件，同时FLAG_DISALLOW_INTERCEPT标记位没有被设置。而ViewGroup要拦截事件的话，则子View就没处理这个事件，所以后面的其他事件也就满足不了这个条件而不会再走onInterceptTouchEvent方法。

&emsp;&emsp;另外我们还有一个结论就是：**当调用requestDisallowInterceptTouchEvent方法设置了不允许拦截的标记位后，在ACTION_DOWN事件的时候会被重置掉而不起作用，也就是说requestDisallowInterceptTouchEvent方法针对的是ACTION_DOWN以外的其他事件，并且是在不拦截ACTION_DOWN事件的情况下才会起作用。**

&emsp;&emsp;接下来我们来看一下当ViewGroup不拦截事件的时候，事件的分发，源码如下：

```
// Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL; // 此事件是否被取消了

            // Update list of touch targets for pointer down, if needed.
            // 是否要拆分事件，在Android 3.0之后才引入的，默认是拆分
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null; // 用来记录一个要处理事件的子View
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) { // 如果事件没被取消也不拦截的话
                if (actionMasked == MotionEvent.ACTION_DOWN 
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN) // 接下来其他手指的ACTION_DOWN
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) { 
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final View[] children = mChildren;

                        final boolean customOrder = isChildrenDrawingOrderEnabled();
                        // 遍历ViewGroupde所有子View
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder ?
                                    getChildDrawingOrder(childrenCount, i) : i;
                            final View child = children[childIndex];
                            /*第一个条件是判断事件的坐标是不是落在这个子View里面，第二个是判断子View是否在播放动画
                            */
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue; 
                            }
                            


                            newTouchTarget = getTouchTarget(child);// 查找child对应的TouchTarget
                            if (newTouchTarget != null) { 
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break; // newTouchTarget已经有了,则跳出
                            }

                            resetCancelNextUpFlag(child);
                            // 将此事件交给child处理
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                mLastTouchDownIndex = childIndex;
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                // 如果子View处理掉这个事件的话，将此child添加到touch链的头部
                                // 会更新 mFirstTouchTarget
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true; // 记录ACTION_DOWN事件已经被处理了。
                                break; 
                            }
                        }
                    }
```
&emsp;&emsp;从上面的源码，结合我做的注释，基本就可以理解这个流程。其中有个重要的代码，就是GroupView遍历查找目标子View的时候是根据事件是否落在该子View的区域内已经子View是否在播放动画来决定是否要传递事件到该子View。

&emsp;&emsp;然后当判断出了一个目标子View来处理事件的时候，调用了dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)方法，

&emsp;&emsp;我们来看一下这个方法的部分重要源码：

```

            if (child == null) {
	            //XXX省略
                handled = super.dispatchTouchEvent(event);
            } else {
            //XXX省略
                handled = child.dispatchTouchEvent(event);
            }
        }

```
&emsp;&emsp;从中我们得出，到这里会调用子View的dispatchTouchEvent进行下一轮的事件分发，因为我们在GroupView里面遍历查找满足条件的子View，所以在这里child不为null，调用的是child自身的dispatchTouchEvent方法，使得事件到了子View手里，再继续分发。

&emsp;&emsp;另外一点，当dispatchTransformedTouchEvent方法返回了true，也就是child的dispatchTouchEvent方法了true后，则会调用：

```
 newTouchTarget = addTouchTarget(child, idBitsToAssign);
```
addTouchTarget的源码：

```
   private TouchTarget addTouchTarget(View child, int pointerIdBits) {
        TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }

```
&emsp;&emsp;可以发现根据坐标点找到了目标子View，并且赋值给了mFirstTouchTarget!=null。

&emsp;&emsp;到这里就完成了这一轮的事件分发，然后还没有结束，加入遍历完所有子View后，没有找到合适的子View来处理呢？

&emsp;&emsp;这里有两种情况导致这种结果：第一种是ViewGroup根本就没有子View，第二种是子View的确是有处理了，但是在dispatchTouchEvent返回了false(当然默认是返回true，只有重写View的这个方法后才可能返回false)

&emsp;&emsp;这种情况就看接下来的ViewGroup处理ACTION_DOWN的源码了：

```
 // Dispatch to touch targets.
 //如果遍历完没有找到合适的子View处理事件的话
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                //注意第三个参数为null，而前面传的是child
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } 
```
&emsp;&emsp;当没有找到合适的子View处理事件的话，ViewGroup则只能自己处理事件了，同样会调用dispatchTransformedTouchEvent，不同的是第三个参数传的不是child了 而是null。

&emsp;&emsp;使得dispatchTransformedTouchEvent里面，child为null，调用了handled = super.dispatchTouchEvent(event);事实上，super.dispatchTouchEvent(event)就是View的默认处理方式了。

而如果mFirstTouchTarget！=null，则else里面的代码：

```
while (target != null) { // 遍历TouchTarget形成的链表
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true; // 已经处理过的子View不再让其处理事件
                    } else {
                        // 取消child标记
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        // 如果ViewGroup拦截了touch事件则把touch链上的child发送cancel事件，也就是参入cancelChild为true
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true; // TouchTarget链中任意一个处理了则设置handled为true
                        }
                        if (cancelChild) { // 如果是cancelChild的话，则回收此target节点
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next; // 删除一个节点
                            }
                            target.recycle(); // 回收
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target; // 访问下一个节点
                    target = next;
                }
```
&emsp;&emsp;当mFirstTouchTarget不为空则表示有子View要处理事件，则接下来的其他事件，因为不是ACTION_DOWN则不会走ACTION_DOWN事件的逻辑，直接到达这里，处理接下来的其他事件。

&emsp;&emsp;最后一个事件是ACTION_UP，就是ACTION_UP处理完之后，会调用resetTouchState();来重置状态。

&emsp;&emsp;到这里，我们就分析完ViewGroup的dispatchTouchEvent方法了。

&emsp;&emsp;可能有同学会问，ViewGroup的dispatchTouchEvent是走完了，但是好像没看到ViewGroup如果自己拦截的话则调用onTouchEvent啊。

&emsp;&emsp;事实上，ViewGroup并没有调用onTouchEvent，ViewGroup也没有去重写onTouchEvent。此话怎讲呢？

&emsp;&emsp;在上文我们分析到了当ViewGroup拦截了事件后，会调用dispatchTransformedTouchEvent方法，只是第三个传入的参数是null，如果不为空则会再继续下一轮分发了，所以要拦截事件的话，这里就会传如null，而传入null后，注意在dispatchTransformedTouchEvent方法里面走的是handled = super.dispatchTouchEvent(event);。注意super，ViewGroup直接继承自View，所以super.dispatchTouchEvent(event);直接变成是View类对事件分发的处理了。其实View类本身因为已经是child了，不是group，所以事件到达了View后，View只有决定要不要处理这个事件，而没有所谓得继续像下级分发。

&emsp;&emsp;由于View不需要完成分发的操作，所以dispatchTouchEvent方法显得非常简单。

# View#dispatchTouchEventd的源码分析
---
&emsp;&emsp;接下来我们来看一下View类的dispatchTouchEvent源码：

```
public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            //没有焦点则不处理这个事件，返回了false
            //当focusable=false的话则会在这里被返回false
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }
//返回值，默认为 false 代表不消费。
        boolean result = false;
	//判断是否是键盘输入事件，如果先是的话先传递给输入事件的onTouchEvent,并且还会继续执行后面的代码
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }
		//是判断窗口是否被遮挡，如果被遮挡则返回false,比如有时候两个View是会重叠的，导致其中一个被遮挡了。
        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            //此处我们也在上一篇文章分析过了，当设置了mOnTouchListener后，如果onTouch返回true则不会调用onTouchEvent
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        //判断手势来停止滚动
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
```
&emsp;&emsp;对比起ViewGroup，View的dispatchTouchEvent方法源码很简单吧。而且在上一篇文章中我们也提到过，**当设置了mOnTouchListener后，如果onTouch返回true则不会调用onTouchEvent**

# 后续
-----
关于View的onTouchEvent方法我们已经在上一篇文章[Android开发知识（八）：Android事件处理机制：事件分发、传递、拦截、处理机制的原理分析（中）](http://blog.csdn.net/lc_miao/article/details/78263102)分析完了，若有同学对这个方面不了解的请阅读上一篇文章。

如有疑问或者分析有误的话，烦请读者留下评论，谢谢。

