---
tags: Android 
read: 576   
---
@[toc]
# 前言
---------
&emsp;&emsp;在我们刚开始学习安卓的时候，总会一开始就接触到Button，也就是对按钮进行一个事件监听的事件，当我们点击屏幕上的按钮时就可以触发一个点击事件。那么，从我们点击屏幕到按钮触发事件这个过程，是什么样子的呢？本文我们就来谈一下关于事件拦截处理机制的基本知识。

&emsp;&emsp;我们知道，在Android中，View视图是以树形结构来展示的，也就是说，一个ViewGroup既可以可以装入若干个View，也可以在ViewGroup里面再嵌套若干个ViewGroup，那么对于一个事件，子View或者父ViewGroup都可能要处理，因此就须有有一些“规则”来定义这个事件处理机制。

&emsp;&emsp;为了让初学者可以更容易理解这个事件处理机制的过程，本文先讲最基础的方面，把事件处理机制的分析过程放在后一章节再讲。
# MotionEvent
-------------
&emsp;&emsp;在我们触摸屏幕的过程中，可以分为三种情况，分别是按下、滑动、弹起。Android中为我们封装好了一个MotionEvent类，使得我们对屏幕的一系列操作事件都可以记录在这个MotionEvent里面。
&emsp;&emsp;这三种情况分别对应MotionEvent的：
&emsp;&emsp;MotionEvent.ACTION_UP
&emsp;&emsp;MotionEvent.ACTION_MOVE
&emsp;&emsp;MotionEvent.ACTION_DOWN
在我们手指按下的时候触发MotionEvent.ACTION_UP事件，可能我们手指不小心一动，会触发MotionEvent.ACTION_MOVE滑动事件，接着手指离开屏幕，会触发MotionEvent.ACTION_DOWN事件，我们把这个过程成为一个事件序列，也就是说一个事件序列里面必然包含有ACTION_UP事件和ACTION_DOWN事件，如果有伴随着滑动的话则就有包含ACTION_MOVE事件。MotionEvent还封装了其他很多事件的信息，比如坐标、时间等等。

# dispatchTouchEvent、onTouchEvent、onInterceptTouchEvent的作用
----------
&emsp;&emsp;在ViewGroup中，涉及到事件处理过程的有三个重要的方法，分别是：

```
public boolean dispatchTouchEvent(MotionEvent ev) 

public boolean onTouchEvent(MotionEvent event) 

public boolean onInterceptTouchEvent(MotionEvent ev) 
```
&emsp;&emsp;其中，从方法名来看也不难看出，dispatchTouchEvent方法是用来进行事件分发，onTouchEvent是对事件的处理，onInterceptTouchEvent是对事件的拦截。

&emsp;&emsp;在本文中，我们先不要去分析这几个方法的源码，我们只需要了解这个过程：
&emsp;&emsp;当事件传递到一个ViewGroup上面时，ViewGroup会触发dispatchTouchEvent方法，随后调用onInterceptTouchEvent方法确认是否拦截此事件，最后如果事件是自己来处理的话，则调用onTouchEvent方法。
&emsp;&emsp;在ViewGroup类中，onInterceptTouchEvent方法总是返回false，表示默认是不拦截事件的，除非去重写ViewGroup类来返回true。而onTouchEvent方法的返回值表示是否消费(返回true则消费)此事件，消费的意思就是说ViewGroup自己处理了这个事件，不再传递到上一层的onTouchEvent去。

&emsp;&emsp;而在View中，与ViewGroup相比，同样有dispatchTouchEvent方法和onTouchEvent方法。但是没有onInterceptTouchEvent这个方法，因为在一个View中，已经是View树的叶子节点，它没有下一级的视图嵌套，所以不需要决定是否拦截事件，它自己就可以处理事件了。

&emsp;&emsp;在View类中，只要该View是可以点击的，那么默认都会在onTouchEvent返回true，表示自己消费了这个事件，不再传递到上一级ViewGroup去。

&emsp;&emsp;事件传递的过程其实非常好理解，想像一下这么一个情景：在公司里BOSS给总监下达了一个任务，总监把这个任务派给了经理，而经理把这个任务派给了你，你把这个任务干完了，向经理汇报，随后经理签名确认后向总监汇报，总监签名确认后再汇报给BOSS，这个任务就算完成了。

&emsp;&emsp;在这里，总监就相当于是一个ViewGroup，经理也是一个ViewGroup，而你是一个View。总监把这个任务利用dispatchTouchEvent方法分发给了经理，自己调用onInterceptTouchEvent方法返回false，表示自己不做这个任务。然后经理把这个任务利用dispatchTouchEvent方法分发给了你，同样自己调用onInterceptTouchEvent方法返回false，表示自己不做这个任务。而你则调用了dispatchTouchEvent方法后只能自己来干这个苦差事，所以调用了onTouchEvent方法，并且返回了true，表示自己完成了任务（消费这个事件）。那么经理和总监就不用去干这个件事，所以他们就不调用onTouchEvent方法。

&emsp;&emsp;假如出现了一个情况，你搞不定这个任务，只能请求经理去帮忙，那么你就在onTouchEvent方法后返回了false，表示自己没完成这个任务(不消费此事件)，那么随后经理就会调用onTouchEvent方法，如果自己能完成，那么他就返回true(消耗事件)，如果经理自己也搞不定那么就只能返回false，让总监去调用onTouchEvent方法自己去处理这个任务。

&emsp;&emsp;好了说到这里，究竟是不是这样呢，我们来用代码验证一下，我们分别创建RelativeLayoutA、RelativeLayoutB，都继承自RelativeLayout，也等同于是ViewGroup，再创建一个MyView继承自Button类，也等同于是继承View。
在RelativeLayoutA类中重写上面提到的三个方法，分别打印出他们的方法名:

```
 @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.i("lc_miao","RelativeLayoutA : dispatchTouchEvent");
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.i("lc_miao","RelativeLayoutA : onTouchEvent");
        return super.onTouchEvent(event);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        Log.i("lc_miao","RelativeLayoutA : onInterceptTouchEvent");
        return super.onInterceptTouchEvent(ev);
    }
```
&emsp;&emsp;同理，在RelativeLayoutB类中也重写这三个方法。

&emsp;&emsp;在MyView中，重写两个方法（注意View是没有onInterceptTouchEvent方法的）：

```
   @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.i("lc_miao","MyView : dispatchTouchEvent");
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.i("lc_miao","MyView :  onTouchEvent");
        return super.onTouchEvent(event);
    }
```

&emsp;&emsp;随后，我们创建一个Activity，还有一个布局文件如下：

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.example.android7.EventsActivity">
<com.example.android7.RelativeLayoutA
    android:id="@+id/RelativeLayoutA"
    android:layout_width="300dp"
    android:layout_height="300dp"
    android:layout_centerInParent="true"
    android:background="#ffff00"

    >
    <com.example.android7.RelativeLayoutB
        android:id="@+id/RelativeLayoutB"
        android:layout_width="150dp"
        android:layout_height="150dp"
        android:layout_centerInParent="true"
        android:background="#00ff00"
        >
            <com.example.android7.MyView
                android:id="@+id/Button"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="按钮"
                android:layout_centerInParent="true"
                />

    </com.example.android7.RelativeLayoutB>

</com.example.android7.RelativeLayoutA>
</RelativeLayout>

```

&emsp;&emsp;运行，界面截图如下：

![运行截图](http://img.blog.csdn.net/20171016160924319?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


&emsp;&emsp;可以看到最外层黄色的就是RelativeLayoutA，而中间那层青色的就是RelativeLayoutB，最中间的则是按钮。
我们点击一下按钮，查看log：


![BUTTON拦截](http://img.blog.csdn.net/20171016161237548?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


&emsp;&emsp;通过log的打印，其实分为两个部分，我用红线分割开来，当我们按下屏幕的时候，打印顺序则是这三个控件的事件拦截所被调用的顺序

&emsp;&emsp;因为Button默认是可以点击的（即使我们并没有设置点击监听事件），所以MyView打印出了onTouchEvent，随后返回了true，这个ACTION.DOWN事件就被MyView消耗掉了。

&emsp;&emsp;如果MyView自己不处理任务呢？我们把MyView的onTouchEvent事件返回false，编译运行后点击中间的按钮，再看下打印：

![不处理](http://img.blog.csdn.net/20171016161832095?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;从Log中我们发现，只要MyView的的onTouchEvent事件返回false，那么RelativeLayoutB的onTouchEvent就会被调用，由于默认是返回false，最终再给到最上级RelativeLayoutA去。
在这里有个疑问，log只是打印出了ACTION.DOWN的打印，并没有像上面的log一样打印出ACTION.UP，这个问题要留到下一个博文我们深入源码分析后才可以给出答案，在这里我们暂时只需要知道，也是必须重要的一个点就是：

&emsp;&emsp;如果在同一个事件序列里面，如果ACTION.DOWN事件不被这个View做出消耗，则后面陆续的事件序列则不会传递到这个View来。

&emsp;&emsp;在这个Log里面由于MyView不处理事件，而RelativeLayoutB和RelativeLayoutA其实也是不处理自己事件的，最后交由了更高级别的ViewGroup(Activity)去响应了，所以后面的ACTION.UP不会再传递到这几个控件上来了。

&emsp;&emsp;假如我们在RelativeLayoutB中让onInterceptTouchEvent返回true，表示RelativeLayoutB会拦截事件自己处理，不分发给下一级View树处理，编译运行后点击中间的按钮，我们再来看看Log：

![拦截](http://img.blog.csdn.net/20171016164602729?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;从这个Log中可以看出，MyView并没有打印出来，说明他没有接收到事件，因为RelativeLayoutB已经把事件给拦截了，就不再分发给MyView，而RelativeLayoutB把事件拦截了后自己调用onTouchEvent，默认是没有消耗事件的，所以才会再调用RelativeLayoutB的onTouchEvent方法。

&emsp;&emsp;注意事件拦截和事件消费是两回事，事件拦截说的是不把事件发给下一级View，而事件消费说的是处理完这个事件还要不要让上一级也处理，如果消费了事件那么就不会再让上一级处理这个事件。

&emsp;&emsp;关于事件拦截处理机制的基本分析，我们就讲到这里，在下一文章中我们再来深入Android源码解剖事件处理机制的实现过程。
