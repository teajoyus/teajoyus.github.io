---
tags: Android 
read: 346   
---
​​![生命周期图](https://img-blog.csdn.net/20170707175214852?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


其中，onCreate对应onDestory，onStart对应onStop，onResume对应onPause。





onCreate：activity被创建新实例的时候调用，完成一些初始化操作。



onStart：activity准备显示但还不能交互。



onResume：activity获得焦点，这个状态下才可以进行交互。



onPause ：正在运行的activity被其他activity转入前台时调用。



onStop：如果转入的activity完成遮住activity那么就会调用。



跳转activity：首先执行A的onPause，然后执行B的onCreate、onStart，如果B不是透明的话，会执行A的onStop方法，如果透明的话比如消息框就不会执行。然后再走B的onResume。



保存信息：如果内存太紧张的话，会直接走onDetory而不一定走onStop，所以保存状态信息是应该在onPause方法中处理，而不是onStop。



网上有面试题说走什么时候走onCreate不走onStart。感觉那除非是在onCreate里面就调用finish方法(这个并非无意义，可能有些特殊情况下我们需要这个activity完成一些初始化操作但是无需它显示在前台。)。



onRestart：在Activity被onStop后，但是没有被onDestroy，在再次启动此Activity时就调用onRestart（而不再调用onCreate）方法



onSaveInstanceState调用：这个并不属于生命周期，因为不一定会被调用。在按电源键、关闭屏幕、旋转屏幕、按HOME键、跳转到其他Activity，不是人为的被finish掉上一个Activity。一般发生在onPause或者onStop的时候。如果是用户主动销毁，比如说主动调用finish或者是按下返回键则不会被调用,因为在这种情况下，用户的行为决定了不需要保存Activity的状态。



通常onSaveInstanceState()只适合用于保存一些临时性的状态，而onPause()适合用于数据的持久化保存。



onRestart ：当处于onStop状态的Activity需要再次显示时调用。



onRestoreInstanceState：不属于Activity的生命周期，不是人为的被finish时下次就被调用。但是他不是与onSaveInstanceState成对的，onSaveInstanceState是可能被杀就会调用，但是onRestoreInstanceState要确定被dinish(非人为)的才有用



onNewIntent：activity有四种启动模式：standard、singleInstance、singleTop、singleTask。只有在standard模式下不会被调用。





因为启动模式跟生命周期也挂钩了，所以下面也来简单的说一下这几种启动模式：

standard:

activity默认的启动模式，只要每次startactivity都会再创建一个实例(比如在A中又启动A，返回时就又回到A)。

singleTop ：

重点在这个“top”，也就是栈顶的意思。就是在要创建的时候，如果栈顶有这个实例就重用它，此时 onNewIntent被调用，而不会调用oncreate，如果栈顶没有这个实例，但是栈中有，那还是会再创建一个，此时不再调用onNewIntent，而是调用oncreate。所以这个模式下重点突出在于栈顶，比如你的activity进入顺序是A->B->B,那么按一次返回就直接回到A，因为在B中又进入B不会有新的实例。。但是如果进入顺序是B->A->B那么按一次返回就是到A 第二次返回就到B。（如果在栈顶，则不重新创建，如果不在，那么跟standard模式一样）

singleTask  ：

重点在于”single”,与singleTop不同，singleTask  是只要在栈中有这个实例，那么就会把这个实例拿到栈顶来用(当然，走的是onNewIntent不走oncreate)。但是这个要注意的地方是，由于被推到栈顶，那么在这个实例之前的其他实例都会被移除。只有栈中不存在该实例才会走oncreate。比如，A是singleTask模式 ，B、C是standard模式，有进入顺序如下B->A->C->A,那么最后进入的这个A是复用之前的A实例，在按下返回键时，直接回到了B。

singleInstance  ：

重点突出在“Instance”，也就是说是单例的。如果没有该实例，那么这个模式会创建一个新的栈，并且栈中只存放这个实例，所有的activity启动它都会在这个实例中启动，而不会生成新的实例，在这过程中singleInstance  模式始终保证只有一个实例，而singleTask并不保证。举个例子A是standard模式，B是singleInstance  有如下进入顺序：A->B->A->B，在这个过程中是这样子的：在第一个A的时候进入B，B由于还没实例则创建新的任务栈并且生成B的实例，然后B又启动A，这个时候第二个A就会在第一个A的栈顶中(与B不是同一个栈，B独享一个栈)A启动B发现已经有实例，则不生成新实例，在我们按下返回键的时候会又B返回到A，再按一次就再返回到A，再按一次就返回到桌面了。



以上的举例过程就不给出源码与运行截图，由读者亲测。




