---
tags: Android 
read: 1056   
---
&emsp;&emsp;在上一篇博文中，我们已经学习过了消息处理机制的基础，对这个过程所涉及到的几个重要的类也有了一定的了解，如果在这方面不清楚也没看过上一篇博文的读者，请点击先阅读[《 Android开发知识（五）消息处理机制Handler+Looper+MessageQueue的原理分析（上）》](http://blog.csdn.net/lc_miao/article/details/77504343)

&emsp;&emsp;在这篇博文中，我们来分析这个通信的过程。
&emsp;&emsp;长话短说，我们直接从handler的sendMessage()一步步说起。毕竟我们的消息处理来源于handler的一系列send方法。

&emsp;&emsp;我们调用的sendMessage方法源码如下：
```
 public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```
&emsp;&emsp;实际上是调用了sendMessageDelayed方法，sendMessageDelayed方法在发送消息的时候允许我们设置消息的延迟发送，第二个参数则是延迟的毫秒数，这里传入了0，表示不延迟。细心的读者还看到有返回值boolean，代表的则是消息是否发送成功。再点进去：

```
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```
&emsp;&emsp;实际上调用了sendMessageAtTime方法，该方法则是指明在特定的时间发送消息，可以看到sendMessageDelayed指定的延迟参数delayMillis也是加上当前时间戳，形成一个特定时间值传给sendMessageAtTime方法，再点进去：
```
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
&emsp;&emsp;到这里，首先判断mQueue是否为空，为空的话则抛出异常提示没有队列，那么这个是在什么时候赋值的呢？找啊找，发现mQueue 赋值的有两个地方，分别是Handler的两个构造方法，第一个是：

```
public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        //把该线程的looper的消息队列赋值过来
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
&emsp;&emsp;该构造方法我们好像基本没这么调用，基本都是Handler的无参构造方法，来看看它的无参构造方法：

```
  public Handler() {
        this(null, false);
    }
```
&emsp;&emsp;调用的是两个参数的构造方法，只是默认第一个参数为null，因为我们没有传入CallBack，CallBack实际上也是一个消息处理接口：
```
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
```
&emsp;&emsp;第二个参数则默认是false，表示默认是同步消息，也就是说消息是一个执行完接着执行下一个了，而异步消息则不一定是这样，也可能在上一个同步消息还没处理就已经轮到这个异步消息，为什么这么说呢？这部分的解析我们放到后面的源码来分析。在这里也可以先忽略这个同步异步的概念。

&emsp;&emsp;通过调用Message对象中的setAsynchronous(true)可以设置该消息是异步消息，值得注意的是当你调用这个党法时编译器会提示你这个方法是在API 22之前是被隐藏的。

&emsp;&emsp;所以我们new出来的无参构造方法调用了这个二参构造方法，从而走了二参构造方法的流程。

&emsp;&emsp;从代码而知，首先if (FIND_POTENTIAL_LEAKS) {XXX｝这部分代码只是检查Handler对象是否为静态，不为静态的话编译器会给出提示。

&emsp;&emsp;下面的代码就尤为重要：
```
mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
```
&emsp;&emsp;首先，利用Looper.myLooper();来得到looper，该方法实现如下：

```
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
```
&emsp;&emsp;发现这不就是用到上一篇我们说到的ThreadLocal么，可以看出来mLooper = Looper.myLooper();就是拿到这个线程的消息循环。如果拿不到，则抛出异常，异常也告诉我们没有调用Looper.prepare()，那我们就看看该方法的代码：

```
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
&emsp;&emsp;实际上该方法也只是调用了ThreadLocal的set方法，在当前线程创建一个消息循环，传入给Looper一个参数为true，表示运行消息被打断，这部分代码可以追溯到MessageQueue中：
```
 // True if the message queue can be quit.
    private final boolean mQuitAllowed;
```

&emsp;&emsp;并且从中我们也可以看出，在创建Handler之前必须调用Looper.prepare()，而这个Looper.prepare()在该线程也只允许被调用一次，调用多次会抛出异常（当然这里排除人为的直接删除掉ThreadLocal的值，这并没有什么意义）

&emsp;&emsp;那么问题来了，按照上面的逻辑，在创建Handler的时候会判断是否有looper，但是我们在创建的时候好像也没有调用这个Looper.prepare()方法就能正常使用。

&emsp;&emsp;说到这里，考虑一种情况，读者是否思考过，线程总会终止运行后被回收，而为什么主线程可以一直运行着，就像我们启动应用时Activity能一直显示着，并不会说主线程执行完了要退出了。原因是因为主线程对于我们应用来说是特殊线程，也只有它才能更新UI，系统在启动我们应用入口的时候，已经对主线程初始化了消息循环，也正式因为这个消息循环不断循环下去，我们的应用才得以长期运行。
&emsp;&emsp;启动应用的入口来自于ActivityThread类的main方法，代码如下：

```
 public static void main(String[] args) {  
        SamplingProfilerIntegration.start();  
  
        // CloseGuard defaults to true and can be quite spammy.  We  
        // disable it here, but selectively enable it later (via  
        // StrictMode) on debug builds, but using DropBox, not logs.  
        CloseGuard.setEnabled(false);  
  
        Environment.initForCurrentUser();  
  
        // Set the reporter for event logging in libcore  
        EventLogger.setReporter(new EventLoggingReporter());  
  
        Process.setArgV0("<pre-initialized>");  
		  //创建主线程的消息循环
        Looper.prepareMainLooper();  
  
        // 创建ActivityThread实例  
        ActivityThread thread = new ActivityThread();  
        thread.attach(false);  
  
        if (sMainThreadHandler == null) {  
            sMainThreadHandler = thread.getHandler();  
        }  
  
        AsyncTask.init();  
  
        if (false) {  
            Looper.myLooper().setMessageLogging(new  
                    LogPrinter(Log.DEBUG, "ActivityThread"));  
        }  
		 //启动循环
        Looper.loop();  
  
        throw new RuntimeException("Main thread loop unexpectedly exited");  
    }  
```
&emsp;&emsp;可以发现里面有两处代码，一个是Looper.prepareMainLooper(); 查看源码如下：

```
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
&emsp;&emsp;代码是先调用 prepare(false)创建一个不可被打断的消息循环，然后再把创建出来的Looper赋值给sMainLooper 对象。

&emsp;&emsp;之后main方法调用了Looper.loop(); 开启了消息循环

&emsp;&emsp;因此，我们才可以在主线程中创建我们的Handler对象而无需调用prepare，而倘若在子线程创建Handler对象的话，则一定要先调用Looper.prepare()，之后再调用Looper.loop()启动消息循环，不开启循环的话会处理不了消息。而关于loop方法的源码我们放到下面再讲。

&emsp;&emsp;我们“递归”到前面的sendMessageAtTime这个方法，这个方法里面我们解析了关于mQueue和mLooper的赋值。

&emsp;&emsp;最终调用到了enqueueMessage方法：

```
  private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
&emsp;&emsp;从 msg.target = this;可以看出该方法首先让这个消息保持有Handler自身的引用，然后判断是否为赋值异步消息，接着调用queue对象的enqueueMessage方法将消息插入到队列中，查看MessageQueue的enqueueMessage方法源码如下：

```
 boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                //注意这句
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                     //注意这句代码
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

&emsp;&emsp;代码有点多，但是我们挑重点的来，前面的两个if只是对消息做个是否有效的判断，然后：
```
 if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
```
&emsp;&emsp;判断消息是不是被中断，通过查看mQuitting赋值为true的地方，发现是quit方法，代码如下：

```
void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```
&emsp;&emsp;quit方法用来中断一个消息，而当是一个被设置为不可中断的消息，调用此方法则抛出异常。可以看到终端消息实际上也就是设置中断标记mQuitting为true。

&emsp;&emsp;其中传入的safe参数表示了是否安全退出，也就是说如果safe为true的话，则会调用removeAllFutureMessagesLocked方法，为false调用removeAllMessagesLocked方法。
我们看看removeAllFutureMessagesLocked方法源码：
```
private void removeAllFutureMessagesLocked() {
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;
        if (p != null) {
            if (p.when > now) {
                removeAllMessagesLocked();
            } else {
                Message n;
                for (;;) {
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        break;
                    }
                    p = n;
                }
                p.next = null;
                do {
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();
                } while (n != null);
            }
        }
    }
```
&emsp;&emsp;从源码顺序读下来，意思就是，如果消息的执行时间都大于现在，那么就还是调用removeAllMessagesLocked()，如果不是的话，则遍历链表，看看从哪个消息开始就是延迟消息（即when>now）,那么就删除在此开始之后的全部消息。

&emsp;&emsp;我们看removeAllMessagesLocked的源码：
```
  private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }
```
&emsp;&emsp;发现它是直接全部清空链表了。

&emsp;&emsp;根据源码我们就可以知道了。如果调用quit(true)的话则会清空全部的延迟消息，而非延迟的消息会继续处理，如果是quit(false)则清空全部消息。

&emsp;&emsp;如果要退出消息循环队列的话，我们需要调用Looper.quit(),或者是Looper.quitSafely(),两个方法都是调用MessageQueue的quit方法，根本区别就是传入的safe参数值不同，Looper.quitSafely()传入的是true。而在调用了quit方法之后，循环队列就不会再接收消息了，因为已经表示退出消息循环队列了，这通常我们在子线程里会这样子使用到，因为子线程如果已经处理完了但是一直不退出消息循环，则会一直保持着线程而浪费资源。需要注意的是Looper.quit方法API一直都有，而Looper.quitSafely()是在API 18之后才有的。

&emsp;&emsp;我们再回到上面当mQuitting为true时，消息便直接被回收，也就不插入到队列中了，并且返回false表示没发送消息成功（因为在插入已经被中断了）。

&emsp;&emsp;接下来的代码实际上就是链表的插入代码了，首先判断链表为空则是头结点的插入，不为空则遍历到链表的结尾或者是根据when的前后（when表示执行时间）进行插入。

&emsp;&emsp;在上面的代码中，有一句代码：
```
needWake = mBlocked && p.target == null && msg.isAsynchronous();
```
&emsp;&emsp;从代码面看上去好像是说如果是阻塞的并且Message没有target，同时还是一个异步消息的话，则“需要唤醒”。而在前面我们提到的MessageQueue中的enqueueMessage方法中首先就已经判断了msg的target为空的话则抛出一个提示“Message must have a target.”的异常，也就是消息一定要有target的，我们也知道target其实就是发送消息的那个handler对象。那这里还判断p.target==null不显得多余么。

&emsp;&emsp;要解释清楚这个逻辑，这里我们还需要先来说一个关于“Barrier”的概念。

&emsp;&emsp;Barrier看上去是一个拦截器的意思。当我们设置了一个拦截器后，在后面的同步消息则暂时无法处理，除非我们解除了这个拦截器后同步消息才可以被处理。这里之所以说“同步消息“而不是“消息”，就是因为异步消息并不会受到这个限制，在设置拦截器后，异步消息还是一样会处理。这就是上问提到的关于同步消息和异步消息的区别。当然，我只是这么说的话还是有点没依据，我们来看一下源码吧。

&emsp;&emsp;在MessageQueue中，提供了一个给我们设置拦截器的方法postSyncBarrier（注意低版本的叫enqueueSyncBarrier，其代码没什么区别）。该方法源码如下：

 private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
    
&emsp;&emsp;看上去好像是创建了一条消息，然后指定arg1是一个自增的token，这里自增token只是为了得到唯一的token。然后就直接把这条消息插入到链表中了，感觉这条消息好像是少了什么？没错，它并没有指定target，也是因为如此，才与普通消息区别开来，而源码注释也提示了我们随后需要再调用removeSyncBarrier方法来去除掉一个拦截器，否则同步消息没办法没执行。

&emsp;&emsp;关于removeSyncBarrier的源码这里就不介绍了，因为这不是很重要。

&emsp;&emsp;现在，我们已经清楚了同步消息和异步消息的区别，就是设置拦截器后消息是否还能执行的区别，而拦截器消息则是target为null。

&emsp;&emsp;回到上面我们分析刚才这句代码：
```
needWake = mBlocked && p.target == null && msg.isAsynchronous();
```
&emsp;&emsp;现在就好理解了吧，这里三个条件换成中文意思就是：

needWake  = 阻塞了？设置拦截器了？是异步消息？

&emsp;&emsp;那么就need wake，当然，在enqueueMessage方法下面还有这句：
```
   if (needWake && p.isAsynchronous()) {
          needWake = false;
   }
```
&emsp;&emsp;表明了如果在此之前有异步消息的话，那么此条消息并不需要立即被唤醒的，因为需要被先执行的是前一条异步消息。

&emsp;&emsp;好了，到此我们终于讲完了从Handler的send系列方法到插入消息队列的过程，实际上，插入到的消息队列正是主线程的消息队列，因为在Handler的构造方法中，已经指明了mQueue = mLooper.mQueue; 消息队列是从looer从获取的，而looper对象又是从prepare方法中创建的，在创建的时候Looper的构造方法里也会创建一个消息队列，如下：

```
   private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
&emsp;&emsp;到这里就已经清楚从Handler的send系列方法到插入消息队列的过程过程了吧。

&emsp;&emsp;然而，这还没结束，我们只是讲了消息的发送，那消息的处理在哪里？

&emsp;&emsp;在前面我们提到的程序入口的main方法，在最后调用了Looper.loop()后开启了消息循环。如果只是调用prepare方法那么只是创建了消息循环，需要再调用loop方法后才能处理消息。实际上，把消息取出来的操作，正是在loop方法里面，我们来看看loop方法的源码：

```
 public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
&emsp;&emsp;源码看上去比较长，但其实并不难理解。

&emsp;&emsp;该方法我们主要看在for里面的代码，for(;;)是一个死循环，也只是因为死循环才有消息循环的概念。

&emsp;&emsp;首先看for循环里第一句代码：

```
Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
```
&emsp;&emsp;通过获取queue.next()；返回的消息，源码注释了might block，让我们知道这个方法是会阻塞的 我们点击查看MessageQueue类的next方法的源码：

```
   Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
&emsp;&emsp;这个next方法源码更长，同样我们也无需纠结于这里的每一句代码，毕竟这并不影响我们会原理的分析。我们挑重点代码来看。

&emsp;&emsp;可能读者也注意到了，这里面也有一个for死循环，里面有这部分代码：

```
if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
```
&emsp;&emsp;我们看到了if里面的条件判断后面是msg.target == null，这显然就是判断这条消息是不是拦截器，如果是的话，从while循环条件：msg != null && !msg.isAsynchronous()中表明，如果有拦截器的话则一直绕过同步消息，直到开始取出第一条异步消息为止（如果没有的话则msg为空）。

紧接着如果now是小于msg.when的话，说明这一条消息执行时间没到，那么通过：
```
  nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
```
&emsp;&emsp;计算出一个延迟后再来执行，如果已经到了执行的时间的话，那么 mBlocked = false;表示不阻塞了，然后取出链表中的第一条消息，也就是队头。

&emsp;&emsp;如果都msg是为空也就是说都没有消息要处理的话，那么设置nextPollTimeoutMillis =-1。如果这个时间没有消息可以返回的情况下才会执行后面关于IdleHandler的代码，这也表明IdleHandler就是在空闲的时间内没有需要执行的消息时，才会执行IdleHandler的消息，最后设置了nextPollTimeoutMillis =0；

&emsp;&emsp;其中有个相对IdleHandler来说比较重要的代码：
```
boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
```
&emsp;&emsp;也就是我们在设置的IdleHandler里的queueIdle方法返回false后才会移除掉，不然会每次空闲了都被执行。

&emsp;&emsp;好了，这里我们解释完了关于MessageQueue的next方法，现在读者也明白了为什么会阻塞了吧？正是next方法里因为是死循环，除非有新的消息已经到了执行的时间才可以返回来。

&emsp;&emsp;我们来继续看Looper中的loop方法源码，我们才说完了 Message msg = queue.next(); 的这个取消息的过程。

&emsp;&emsp;然后下面判断了msg为空就直接返回了，那msg什么时候为空呢？

&emsp;&emsp;就是上面的MessageQueue的next方法里，当没有消息时才会执行IdleHandler，而其中间还有一句：
```
 // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
```
&emsp;&emsp;mQuitting应该不用多说了吧，在MessageQueue中就提供了一个quit方法指定了mQuitting为true，表明消息队列要退出了。而在我们的代码中要退出这个消息循环队列的话，则需要调用Looper.quit()，其内部也就是调用了MessageQueue的quit方法。

&emsp;&emsp;还没完，我们继续看loop方法接下来的源码。
&emsp;&emsp;我们发现在消息被取出来后执行了这句代码
```
msg.target.dispatchMessage(msg);
```
&emsp;&emsp;我们分析过了，msg.target是指向了发送消息时的handler对象，然后调用dispatchMessage方法来分发消息，我们查看下Handler的dispatchMessage方法源码：
```
/**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
&emsp;&emsp;源码不难看出，只是做了几个逻辑判断的分发：
&emsp;&emsp;1、当msg.callback!=null

&emsp;&emsp;什么时候callback不为空呢，查看Message源码不难看出，当我们在使用Message的obtain方法时，也可以传入一个CallBack对象，而CallBack我们前面也提到只是一个接口，接口方法也是叫handlerMessage。

&emsp;&emsp;也就是说我们在发送消息的时候，可以为这个消息指定一个特定的回调处理，而不经过handler的handlerMessage方法。

&emsp;&emsp;2、mCallback != null。

&emsp;&emsp;mCallback是Handler本身的一个对象，如果我们为handler指定了一个mCallback的话，那么就会回调mCallback的handlerMessage方法。

&emsp;&emsp;比如我们在创建Handler的时候并不重写handlerMessage方法，而是实现了一个Callback接口，然后把接口对象传入Handler对象的构造方法。

&emsp;&emsp;3、 handleMessage(msg);
&emsp;&emsp;这就是我们在创建Handler对象时重写的方法。

&emsp;&emsp;可以看出这三者的优先级是，消息自身callback>Callback>Handler

&emsp;&emsp;之后再调用了msg.recycleUnchecked（）方法回收消息，实际上只是重置了消息，然后再放入到消息池，下次可以用handler的obtainMessage来从消息池中取出来而不用创建新消息。

&emsp;&emsp;这样loop方法的原理就分析完了，相信之前不了解的读者看起来应该挺绕的吧，我们再来总结一下：

&emsp;&emsp;首先我们调用Handler的一系列send方法，最终是调用了Handler的sendMessageAtTime方法，其实我们调用handler的post方法最终也是到sendMessageAtTime该方法最终把消息插入到了Handler所在线程的消息队列中，如果Handler是在子线程中的话，则我们必须先调用Looper.prepare()才能创建一个该线程的消息循环队列，然后调用Looper.loop()来启动循环，而在主线程中这两个方法已经在main入口被调用了。在插入到消息队列后，Looper中的loop方法会循环的把消息在消息的执行时间时取出来，然后调用Message指定的target也就是发送它的Handler，调用dispathMessage来处理消息。这个过程就完了。

以上就是Android的消息处理机制的原理，基于这个原理使得我们可以进行线程间通信。

