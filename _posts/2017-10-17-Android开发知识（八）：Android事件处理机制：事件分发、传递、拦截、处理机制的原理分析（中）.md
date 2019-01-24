---
tags: Android 
read: 1056   
---
@[toc]
# 前言
-----
&emsp;&emsp;在上一章节中，我们谈到了Android中的事件处理机制，展示了事件传递过程的基本性质，如果读者从未接触过关于Android事件处理机制的知识，可以先阅读[Android开发知识（七）：Android事件处理机制：事件分发、传递、拦截、处理机制的原理分析（上）](http://blog.csdn.net/lc_miao/article/details/78251504)
# onTouch、onClick、onLongClick的调用顺序
-----
&emsp;&emsp;在本章节中，我们重点谈论一下onTouch、onClick、onLongClick三个方法被回调的过程。

&emsp;&emsp;在上一篇文章中，我们谈到关于为View添加一个点击事件SetOnClickListener后，就可以通过回调onClick方法来实现事件的响应。而另外还有一个setOnTouchListener方法，通过设置监听后可以在触摸的时候回调onTouch方法。而我们又说到onTouchEvent方法是处理事件的。那么这三个方法究竟有什么区别呢？

&emsp;&emsp;我们在上篇文章的源码中，在Actvity里来分别设置OnClick和onTouch的监听，通过log来打印出他们的调用：

```
   MyView view = (MyView) findViewById(R.id.Button);
        view.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.i("lc_miao","MyView :  onClick");
            }
        });
        view.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.i("lc_miao","MyView :  onTouch");
                return false;
            }
        });
```
&emsp;&emsp;我们来看一下关于onClick、onTouch、onTouchEvent三者的调用顺序，运行后点击按钮，log如下：

![log](http://img.blog.csdn.net/20171017165245133?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;从log上看，当事件被View接收后，在ACTION_DOWN的时候View会分别触发onTouch和onTouchEvent方法，而在ACTION_UP的时候会分别触发onTouch和onTouchEvent、onClick方法。

&emsp;&emsp;这说明了，在日常中我们给View设置点击事件其实响应优先级是最低的，因为他需要同时接收到ACTION_DOWN和ACTION_UP事件后才会触发，而onTouch方法则是在设置监听后，只要有事件到来，则会触发一次，它比onTouchEvent优先被响应。

&emsp;&emsp;事实上：
&emsp;&emsp;1、onTouch比onClick方法多了一个返回值，其返回值也表示了是否消耗事件，如果返回了true则不会再调用onTouchEvent方法
&emsp;&emsp;2、onClick方法是在onTouchEvent里面被回调的，如果onTouch返回了true，onTouchEvent不会被调用，那么onClick也就不会被调用。
我们来查看一下View的源码。从View接收到事件开始，也就是dispatchTouchEvent方法的源码。
#  onTouch、onClick、onLongClick的源码藏身之处
----
&emsp;&emsp;由于Android系统版本的更新，我查看的是android-23的源码，可能与较低版本的源码会有差异，但是整体的核心流程并不会有改变。

&emsp;&emsp;在源码中，我们关注方法中这部分重要的源码：

```
//如果设置了mOnTouchListener且onTouch返回true，则不走onTouchEvent
if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
```
&emsp;&emsp;源码中先判断li != null && li.mOnTouchListener != null的条件，我们不用看源码也清楚mOnTouchListener 就是我们调用setOnTouchLister的时候赋值进去的。所以如果设置了OnTouchListener的话这里条件就成立，其次(mViewFlags & ENABLED_MASK) == ENABLED，要View是enable状态条件才成立，事实上默认就已经是enable状态，除非调用setEnable(false)让控件变为不可选取状态。满足了上面两个条件后，则onTouch便会触发，同时会把onTouch返回值作为条件。到这里，我们也就清楚了onTouch的为什么会被优先响应。

&emsp;&emsp;然后，假如onTouch消费了事件，也就是返回了true，则result变量则为true，导致下面的：

```
  if (!result && onTouchEvent(event)) {
                result = true;
            }
```
&emsp;&emsp;从!result就已经条件不成立了，所以就不会调用onTouchEvent方法，除非onTouch返回false，这验证了前面我们说的：
&emsp;&emsp;onTouch比onClick方法多了一个返回值，其返回值也表示了是否消耗事件，如果返回了true则不会再调用onTouchEvent方法

&emsp;&emsp;接下来我们再来查看onTouchEvent的方法源码，同样方法源码很多，我们挑重点的来看：

```
 // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                //performClick()里面会走onClick
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
```
&emsp;&emsp;从这个代码中mPerformClick 确认是不为空的，然后调用post(mPerformClick)，我们直到View里面的post方法其实也就是相当于把runnable任务放进主线程的消息队列来处理，如果post进去失败，则会直接处理performClick();我们看一下这个mPerformClick的实现：

```
 private final class PerformClick implements Runnable {
        @Override
        public void run() {
            performClick();
        }
    }
```

```
public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            //调用onClick
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }
```

&emsp;&emsp;发现PerformClick中其实主要的是执行一个performClick方法，而我们在performClick方法中可以看出，首先代码判断li != null && li.mOnClickListener != null，这里我们也不难看出mOnClickListener 就是我们调用setOnClickListener的时候设置进去的对象，当设置了点击监听事件后此处条件便成立，然后会调用一个播放点击音效，然后调用li.mOnClickListener.onClick(this);这正是我们设置点击监听事件的时候，回调的onClick方法，由此可以验证我们的第二条说法：
&emsp;&emsp;onClick方法是在onTouchEvent里面被回调的，如果onTouch返回了true，onTouchEvent不会被调用，那么onClick也就不会被调用。


&emsp;&emsp;另外一个我们还可能会用到一个长按的监听，我们也给view增加一个长按事件并在onLongClick打印出来，，发现log如下：

![log2](http://img.blog.csdn.net/20171017174840684?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;从log上看，当View接收到ACTION_DOWN的时候，并且不松开大概0.5s的时候（log从onTouchEvent到onLongClick的执行时间差大概就是0.5s）会执行onLongClick，当接收到ACTION_UP的时候再执行onClick，而如果onLongClick方法中返回了true，则onClick就不会再执行。

&emsp;&emsp;我们再来看一下关于这个说法的源码依据，并且分析源码中是如何确定长按事件并且回调onLongClick的。

&emsp;&emsp;当手指开始按下的时候，执行了onTouchEvent方法。我们从onTouchEvent关于对ACTION_DOWN的执行逻辑代码如下：

```
 case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    //判断是不是一个滚动视图
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    //如果是滚动视图的话
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        //则添加一个延迟消息，大概是getTapTimeout()的时间，实际上就是100ms，如果100ms里面没有滚动则判断为是长按的过程
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        //如果不是可滚动布局的话，则直接就是代表长按了
                        setPressed(true, x, y);
                        //注意传入的是0，代表一个延迟偏移值，如果值越大则等待形成长按的事件会更短
                        checkForLongClick(0);
                    }
                    break;
```
&emsp;&emsp;从代码上看首先执行performButtonActionOnTouchDown(event)，为true则直接跳出，源码如下：

```
/**
     * Performs button-related actions during a touch down event.
     *
     * @param event The event.
     * @return True if the down was consumed.
     *
     * @hide
     */
    protected boolean performButtonActionOnTouchDown(MotionEvent event) {
        if ((event.getButtonState() & MotionEvent.BUTTON_SECONDARY) != 0) {
        //如果是鼠标右键的话会弹出菜单之类的
            if (showContextMenu(event.getX(), event.getY(), event.getMetaState())) {
                return true;
            }
        }
        return false;
    }
```
&emsp;&emsp;MotionEvent的BUTTON_SECONDARY其实对应的就是鼠标中的右键，事实上，在手机设备中只要我们用手指触摸的都是返回false，而比如是接入了鼠标，那么鼠标点击右键了就会有展开菜单之类的功能，则会消费掉这个事件，所以在这里我们条件不成立会接着走下面的代码。接着执行：

```
 // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();
```
isInScrollingContainer方法的源码如下：

```
  /**
     * @hide
     */
    public boolean isInScrollingContainer() {
        ViewParent p = getParent();
        while (p != null && p instanceof ViewGroup) {
            if (((ViewGroup) p).shouldDelayChildPressedState()) {
                return true;
            }
            p = p.getParent();
        }
        return false;
    }
```
&emsp;&emsp;说明该方法是用来遍历View树判断当前按下的View是不是在一个滚动的视图容器中，
&emsp;&emsp;如果是在一个可以滚动的容器中，比如（ListView，ScorllView）那么先设置PFLAG_PREPRESSED标记位，表示用户准备点击，
&emsp;&emsp;随后发出一个延迟的消息来确定用户到底是要滚动还是点击.，通过记录用户按下的坐标和ViewConfiguration.getTapTimeout()指定的时间(源码被设置为100ms)，来延迟100ms后判断出用户是滚动了还是点击的

&emsp;&emsp;而这个消息的执行任务是mPendingCheckForTap，点击查看：

```
  private final class CheckForTap implements Runnable {
        public float x;
        public float y;

        @Override
        public void run() {
            mPrivateFlags &= ~PFLAG_PREPRESSED;
            setPressed(true, x, y);
    //注意这里区别与非滚动容器，因为滚动容器要花getTapTimeout()这个事件去判断是不是滚动，所以形成长按的话要带上这个偏移量来减去这个时间 checkForLongClick(ViewConfiguration.getTapTimeout());
        }
    }
```
&emsp;&emsp;不难看出其实在延迟消息后再调用checkForLongClick，并且参数是ViewConfiguration.getTapTimeout()。
，而如果不是在一个滚动的容器中，则执行：setPressed(true, x, y);标记PFLAG_PRESSED为true表示标记一个按下的状态，再调用
checkForLongClick，不同的是参数是0.

&emsp;&emsp;实际上这两种情况，都是确定了是保持按下状态，不是滚动屏幕。然后再调用checkForLongClick，只不过给出的参数不同罢了。

&emsp;&emsp;到这里也不难看出，checkForLongClick会在检查确定是满足长按条件后执行长按监听的回调。我们看checkForLongClick的源码：

```
  private void checkForLongClick(int delayOffset) {
        if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
            mHasPerformedLongPress = false;

            if (mPendingCheckForLongPress == null) {
                mPendingCheckForLongPress = new CheckForLongPress();
            }
            mPendingCheckForLongPress.rememberWindowAttachCount();
            //形成长按的过程，如果这个消息到执行的时候依旧满足条件，则代表长按事件成立，注意延迟时间会减去一个delayOffset偏移量
            postDelayed(mPendingCheckForLongPress,
                    ViewConfiguration.getLongPressTimeout() - delayOffset);
        }
    }
```
&emsp;&emsp;从代码上看，是把一个mPendingCheckForLongPress 的任务延迟执行，而执行的时间正是ViewConfiguration.getLongPressTimeout()-delayOffset，这里也就直到了如果不是滚动视图则会马上进入这个方法，传了参数0，所以长按延迟是整个ViewConfiguration.getLongPressTimeout()的时间，如果是在滚动容器的话则因为消耗了100ms的时间去判断是否是滚动，所以在这里就会减掉那个时间。

&emsp;&emsp;我们重点看下mPendingCheckForLongPress是个什么任务，查看下他的源码：

```
private final class CheckForLongPress implements Runnable {
        private int mOriginalWindowAttachCount;

        @Override
        public void run() {
            if (isPressed() && (mParent != null)
                    && mOriginalWindowAttachCount == mWindowAttachCount) {
                    //performLongClick里面调用onLongClick，并且返回true则记录mHasPerformedLongPress，而不会再让onClick被调用
                if (performLongClick()) {
                    mHasPerformedLongPress = true;
                }
            }
        }

        public void rememberWindowAttachCount() {
            mOriginalWindowAttachCount = mWindowAttachCount;
        }
    }
```
&emsp;&emsp;因为手指按下后不松开到形成长按的这段等待过程之中,界面是可能发生一些变化的，比如Activity被暂停了或者被重启了,或者这个时候,长按的事件就不应该被响应了

&emsp;&emsp;在View中有一个mWindowAttachCount记录了View的attach次数.他的作用是：当检查长按时的attach次数与长按到形成时的attach一样则处理,否则就不应该形成长按事件. 所以在将检查长按的消息添加时队伍的时候,要记录下当前的windowAttachCount.

&emsp;&emsp;而当满足上面说的条件后，则performLongClick()会被调用，它的源码如下：

```

    /**
     * Call this view's OnLongClickListener, if it is defined. Invokes the context menu if the
     * OnLongClickListener did not consume the event.
     *
     * @return True if one of the above receivers consumed the event, false otherwise.
     */
    public boolean performLongClick() {
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_LONG_CLICKED);

        boolean handled = false;
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLongClickListener != null) {
            handled = li.mOnLongClickListener.onLongClick(View.this);
        }
        if (!handled) {
            handled = showContextMenu();
        }
        if (handled) {
            performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
        }
        return handled;
    }
```
&emsp;&emsp;可以看出当我们设置了长按监听事件了之后会回调li.mOnLongClickListener.onLongClick(View.this);，
&emsp;&emsp;而如果方法返回了了true，则mHasPerformedLongPress = true;

&emsp;&emsp;我们在回过头来看onTouchEvent的ACTION_UP的处理逻辑部分代码：

```
 if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }
```
&emsp;&emsp;可以看到如果mHasPerformedLongPress 为true了，则不会再走performClick()方法回调onClick了。这里验证了我们在上文的说法：如果onLongClick方法中返回了true，则onClick就不会再执行。

&emsp;&emsp;到了这里，我们就已经明白了当一个事件序列被View接收后，onTouch、onClick、onLongClick被回调的原理过程。
# 后续
------
&emsp;&emsp;我将在下一章节中，我们继续谈论Android事件处理机制。主要是剖析出从顶级ViewGroup到最低级的View的事件分发处理的原理过程。要理解这个事件处理机制的过程还是有一些难度的，毕竟源码的流程也比较多。不过全部贴出代码来一句句深究其含义并不是我们从源码上理解Android系统机制的办法，源码太多，我们挑出重要的代码部分就已经足够我们来理解这个过程了。
