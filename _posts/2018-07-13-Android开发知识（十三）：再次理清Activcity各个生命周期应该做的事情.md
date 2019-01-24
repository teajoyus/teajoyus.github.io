---
tags: Android 
read: 622   
---
@[toc]

从一开始接触Android，我们最早就接触到Activity，接触到Activity的onCreate()。后来我们会去学习Activity的各个生命周期，但是在实际项目中却还是有很多生命周期的问题导致在某些环境条件下会产生和预期不一致的结果。这里笔者整理出几个生命周期的知识点，希望对纠结于Activity生命周期如何做处理的你有些许帮助。

问题1：什么情况下Activity只调用onPause不调用onStop？
-------------------------------------
答：在跳转的目标Activity是透明状态的时候（就是有一点透明也算），也就是说不能完全去遮挡住前一个Activity的显示的话（比如跳转的Activity是Dialog的形式），那么前一个Activity只会调用onPause()，这个期间不会调用onStop()。

问题2：onPause()、onStop()方法有什么区别？保存数据到底放在哪个好？
-------------------------------------
答：首先要明白，在做跳转的时候，原来的Activity会先调用onPause()方法，知道被启动的Activity获取焦点后（onResume()）后原来的Activity才会调用onStop()方法（如果被启动的Activity会完全遮挡住原先的Activity的话）。
从这个过程我们可以看得出来，onPause()是在跳转前被调用的，onStop()是跳转后再调用的。很显然，如果你需要执行很多耗时操作去保存数据的话，那么放在onPause()方法就会导致跳转缓慢，所以onPause()其实是不适合做耗时操作的（比如持久化数据到DB、File等），对于一些耗时不多的操作倒是无所谓（比如清理某个变量）。那么onPause()方法适合做什么呢，它适合释放一些资源数据，比如使用到Camera的时候，在onPause()就可以mCamera.release();了。另外的还有解除注册的BroadCast、解除Service绑定等等。而需要做持久化的可以放在onStop()方法里，就不会影响到Activity跳转的速度。

问题3：调用了finish()方法后为什么没调用onDestory()方法，什么情况下Activity销毁时不会调用onDestory()方法。
------------------------------------
答：首先这个问题不是很严谨，调用了finish()方法后是会调用onDestory()方法的，但不是马上被调用。当一个Activity要销毁的时候，onDestory()方法不一定能即时被调用。某种情况下，在startActivity()之后做finish，可能Activity销毁时还没来得及走onDestory()方法就又重新创建走onCreate()方法了。所以首先要明确一点就是调用了finish()方法后不一定会马上调用onDestory()方法。那么什么情况下Activity销毁时不会调用onDestory()方法，这个问题看上去有种钻牛角尖的感觉。在Activity被调用finish()时、或者系统原因主动kill掉Activity的时候都会走onDestory()方法，只有在程序被外部强制关闭进程（比如在设置-应用-强制关闭）、或者在recent Task 最近任务关闭掉这个进程、或者是程序发生异常后崩溃了，这些都不会走onDestory()方法。

问题4：Activity销毁时在onDestory()方法释放资源有什么弊端？有什么更好的方法？
------------------------------------
答：这个问题是延续问题3的，问题3提及到onDestory()不是马上被调用，甚至是不会被调用。所以其实把资源释放、数据清理放在这个生命周期里在某些情况下是行不通的。可能把某些有锁数据（在某个地方没释放掉、另外一个地方一定用不了）在onDestory()方法里做清理，再其他Activity的onCreate()或者onResume()再次使用，则可能存在这个问题。那么可以用什么更靠谱的方法呢？首先，onDestory()方法里做了数据清理资源释放这个绝逼没有问题，所以并不是说我们不用这个onDestory()了，只是考虑到上面说的问题后，我们可以在onPause()或者onStop()方法结合isFinish()方法来判断是否Activity是处于销毁状态，如果是的话则做出数据清理资源释放。有同学要问了，在onPause()或者onStop()方法做了判断后为什么还要onDestory()也做相同操作呢？我举个例子，主动调用isFinish()的时候，确实走到onPause()或者onStop()方法的时候就判断是finish状态，那么onDestory()的清空操作显得不需要了。但是考虑一种情况：当内存紧张的时候，系统kill掉这个Activity呢？此时不是finish状态，那么就要靠onDestory()来做这个事情了。


问题5：onRestart()方法怎么使用？
------------------------------------
答：一个Activity的onStop() 对应 onRestart()，如果调用了onStop()而没有被销毁，那么再进去的时候则是一个重启的状态，当activity从Stopped状态回到前台时，会调用onRestart()，再调用onStart()方法。onRestart()方法只在activity从stopped状态恢复时才会被调用，因此我们可以使用它来执行一些特殊的恢复。但是建议onStop()方法去对应onStart()。也就是我们在onStop()里面做了哪些清除的操作，就该在onStart()里面重新把那些清除掉的资源重新创建出来。

问题6：onSaveInstanceState()的调用时机是什么？怎么使用它？
------------------------------------
答：onSaveInstanceState()方法，注意不是看固定什么操作就会调用它，而是只要系统主动要杀死它的时候就会调用，如果自己按下back键或者finish是用户主动行为，就不会被调用。所以系统什么时候想要主动杀死掉这个Activity，我们不能明确，只能说它要主动杀死的时候就会调用这个方法，而用户的主动销毁行为则不会调用这个方法。所以一个好的应用我们应该是要考虑到这一点，所以重写onSaveInstanceState()为我们的应用做一些被主动销毁时的数据保存工作。而注意当Activity被系统主动kill掉的时候，事实上栈上还是存在这个记录的，所以返回到这个Activity的时候，会重新创建这个实例。这个过程就会调用onRestoreInstanceState()，我们可以利用这个方法来恢复现场，或者在onCreate()方法里面传进来的参数Bundle对象，在重新创建的时候它是不为空的，在这里来恢复现场也可以。注意onRestoreInstanceState()是在onStart()方法之后的，所以它发生在onCreate()之后。其次需要注意的是onSaveInstanceState()和onRestoreInstanceState()并不是什么生命周期方法，而且onSaveInstanceState()和onRestoreInstanceState()不一定是对应的。比如发生了onSaveInstanceState()，但是栈上的记录没有了，那么下次也只会创建，并不是重新创建，当然不会再走onRestoreInstanceState()；

问题7：onCreate()、onResume()适合做什么工作？
------------------------------------
答：很多同学可能觉得，有很多数据初始化的工作，貌似放在onCreate()、或者onResume()方法都没啥问题。但事实上，区别是有的，而且很大。在启动的过程，onCreate()执行完之后会迅速到onStart()被用户可见后再到onResume()获取焦点。然后一直保持在Resumed状态直到Pause状态。也就是说onCreate()方法没执行完，用户是看不到什么东西的。所以onCreate()里面尽量少做事情，尽量只做UI初始化的工作，而注册资源等放在onResume()方法，避免程序启动太久都看不到界面。onCreate()只会调用一次，而每次Activity恢复可见后又会调用一次onResume()方法，所以根据自己的业务逻辑，有些业务数据需要在页面恢复可见的时候重新做什么事情，那么就适合放在onResume()，而onCreate()尽量只做UI初始化和一些不太耗时数据处理。

问题8：调用System.exit(0)为什么不能退出应用，还重启应用了？
------------------------------------
答：如果直接调用System.exit(0)、killProcess，如果栈上面存在没有finish过的Activity的话，那么退出会重新创建启动进程，如果当前没有其他没有finish的activity的话就能退出，所以调用这个要确保没有其他未finish的activity才行。可以利用一个记录Activity的列表，在需要完全退出应用的时候，遍历这些Activity依次调用finish()方法后再用System.exit(0)。

