---
tags: Android 
read: 1056   
---
[TOC]
# EventBus前言

本文主要讲解EventBus的源码解析，如果您未听过/使用过EventBus的话请自行百度一下，几分钟即可GET到这门技能。
如果你还不了解EventBus的Register（订阅）流程，那么我建议你先看看我的上一篇博客: [《EventBus3.1.1源码解析（上篇）》](https://blog.csdn.net/lc_miao/article/details/86065562).
本篇章讲解的是EventBus的发布流程。


同样说明下，我的源码解析是基于EventBus3.1.1的源码：
```
 implementation 'org.greenrobot:eventbus:3.1.1'
```

# EventBus#使用

通常来说，我们在通常在写订阅事件方法的时候，这样定义：
```
@Subscribe()
public void onEventRecivier(MyObject o){
    ...
}
```

当需要指定在主线程接收时，用threadmode参数：
```
@Subscribe(threadMode = ThreadMode.MAIN)
public void onEventRecivier(MyObject o){
    ...
}
```

当需要把该方法事件列为粘性事件（在发布之后注册也能收到）的时候，用sticky = true：
```
@Subscribe(threadMode = ThreadMode.MAIN,sticky = true)
public void onEventRecivier(MyObject o){
    ...
}
```
当需要指定事件优先级时，用priority来排序事件优先级（默认priority=0）：

```
@Subscribe(threadMode = ThreadMode.MAIN,sticky = true,priority = 1){
public void onEventRecivier(MyObject o){
    ...
}
```

当我们要发布事件时，简单的例子：
```
    MyObject o = new MyObject();
    EventBus.getDefault().post(o);
```

只需要调用EventBus的post方法就能把事件发布出去，然后每个订阅者会收到自己定义的事件订阅方法注解参数所规定的方式接收到事件

# EventBus#Post（发布）流程
下面我们将来剖析从调用EventBus#post方法到自动回调订阅方法的流程，直接从post入手吧。贴上post的源码解析：
```

    /** Posts the given event to the event bus. */
    public void post(Object event) {
        /*
            currentPostingThreadState看定义是： ThreadLocal<PostingThreadState>
            关于ThreadLocal不了解的同学，可以看下我之前的一篇讲解Handler的博客中有提到：https://blog.csdn.net/lc_miao/article/details/77504343
            ThreadLocal就是每个线程都拥有自己独立的数据，每个线程都有自己的一份拷贝，不相互影响。
            这里通过ThreadLocal的get方法来得到一个PostingThreadState
            PostingThreadState这个类只是一个实体类，封装了当前线程事件发布的状态

        */
        PostingThreadState postingState = currentPostingThreadState.get();
        //得到该线程的事件队列
        List<Object> eventQueue = postingState.eventQueue;
        //把需要发布的消息添加到队列尾部来
        eventQueue.add(event);
        //isPosting描述了当前是否在发布中，如果当前不是正在发布中，
        if (!postingState.isPosting) {
            //isMainThread记录了当前是否是主线程
            postingState.isMainThread = isMainThread();
            //要准备发布事件，把状态改为发布中
            postingState.isPosting = true;
            //如果该线的事件被取消了则抛出异常
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //一切都没毛病后，那么就往队列中取出消息事件发布出去直到取完
                while (!eventQueue.isEmpty()) {
                    //发布单个事件，重点方法。我们在下面追踪进来
                    //eventQueue.remove(0)相当于队头的一个出队动作
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                //发布完毕后重置状态
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

上面的代码，如果你对ThreadLoacl比较熟悉的话，相信没啥问题。currentPostingThreadState.get();会返回该线程的一份实例，并不需要进行set，因为创建ThreadLocal的时候重载了：
```

    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };
```
当get不到实例的时候，ThreadLocal内部会调用initialValue来初始化实例

PostingThreadState只是一个封装了线程发布状态的类：
```
   /** For ThreadLocal, much faster to set (and get multiple values). */
    final static class PostingThreadState {
        //事件对垒
        final List<Object> eventQueue = new ArrayList<>();
        //是否正在发布中
        boolean isPosting;
        //是否是主线程
        boolean isMainThread;
        //订阅方法
        Subscription subscription;
        //具体的事件
        Object event;
        //是否被取消
        boolean canceled;
    }
```

那么上面的post方法中重点就是继续追踪postSingleEvent方法了，源码如下：
```
  private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        //还记得上篇讲的eventInheritance的作用么
        //就是说是否支持事件继承， A extends B 那么发出事件A后 订阅了事件B的需不需要收到
        if (eventInheritance) {
            //查找出事件的继承链，也就是关于所有eventClass的父类
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            //遍历该继承链
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                //调用postSingleEventForEventType，是下一步的核心
                //传入事件、状态、事件类型
                //注意subscriptionFound取得方法返回值，用“或等于”的形式，也就是只要一次是true那么后面都是为true了
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
        //如果不支持继承，那么就不用查找出父类了，直接调用
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        //subscriptionFound字面也好理解，就是调用postSingleEventForEventType方法后得到个是否有找到订阅类的意思
        //如果没找到的话
        if (!subscriptionFound) {
            //上篇解析的logNoSubscriberMessages作用在这里
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            //sendNoSubscriberEvent这个在上篇也讲过了，是当消息事件发送出去后没有对应的处理方法时，是否发送 NoSubscriberEvent 事件
            //当然，事件类型本身也不能是NoSubscriberEvent，要不继续post到这里就死循环了
            //事件也不能是SubscriberExceptionEvent这个类，因为这个是异常类也会因为sendSubscriberExceptionEvent=true时会发送出来
            //所以要先确保事件不是这两个事件才可以发出没有被处理的消息
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }

```
上面的代码，一开始通过eventInheritance判断是否支持继承，如果支持则需要取出所有父类，然后都是调用postSingleEventForEventType
该方法返回是否找到订阅类，如果都没找到的话，则通过logNoSubscriberMessages判断是否需要打印，通过sendNoSubscriberEvent判断是否需要发送消息未处理的事件

这些eventInheritance、logNoSubscriberMessages、sendNoSubscriberEvent变量开关我们都是在上篇在构造自定义的EventBus时讲过了


接下来我们需要追踪的方法，那就是postSingleEventForEventType方法了，源码如下：
```
 private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        //同步锁，其他线程会等待，不然同时读写会有问题
        synchronized (this) {
        //subscriptionsByEventType在上篇讲过,作用是记录所有的事件以及事件对应的订阅列表，
        //通过这个集合就可以查看到具体都有哪些事件，每个事件具体都被哪些类所订阅
        //于是我们通过事件类型来获取对应的订阅列表
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        //做下判空
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                //在该线程的事件状态记录当前处理的事件
                postingState.event = event;
                //在该线程的事件状态记录当前处理的订阅类
                postingState.subscription = subscription;
                //是否打断
                boolean aborted = false;
                try {
                //拿到订阅类和要发布的事件后，通过postToSubscription来回调出订阅类的订阅方法
                //下面要讲的核心方法
                    postToSubscription(subscription, event, postingState.isMainThread);
                    //是否取消
                    aborted = postingState.canceled;
                } finally {
                    //每次对每个订阅类回调订阅方法后都重置下状态
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                //如果事件状态是取消的，那就被打断了。不继续发了
                if (aborted) {
                    break;
                }
            }
            //因为找到了订阅类，所以返回true 需要注意的是即使事件被取消，只要有订阅类那就是返回true
            return true;
        }
        //没找到订阅类返回false
        return false;
    }
```

上面的方法已经追踪到了该事件具体的订阅列表，然后遍历这个订阅列表去回调对应的事件订阅方法，也就是postToSubscription方法的作用，源码如下：
```
//这个源码是不是看上去有点熟悉，因为它的switch正是对应了注解参数中的threadmode
 private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    //subscription.subscriberMethod则是订阅方法，判断这个订阅方法所注解的线程模式
        switch (subscription.subscriberMethod.threadMode) {
        //POSTING就是默认的，因为注解类Subscribe里面定义了默认是ThreadMode.POSTING
        //POSTING是最简单的，也就是直接在该线程上发布事件，不管该线程上是啥线程
            case POSTING:
                //对订阅方法反射调用 下面将会继续追踪
                invokeSubscriber(subscription, event);
                break;
             //MAIN则对应了线程模式为主线程
            case MAIN:
                //如果当前正好是主线程 则直接调用订阅方法
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                //否则插入到mainThreadPoster中，mainThreadPoster我们在上篇讲过。它也是一个队列，记录的是当前主线程正在发布的事件队列

                //关于enqueue流程我们下面再继续追踪
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
             //MAIN_ORDERED也是在主线程 咋看跟MAIN貌似没区别，但是区别还是有
             //它指明了事件序列需要严格按照串行的顺序来执行
             //不管当前是不是主线程，都要把事件放到mainThreadPoster里面等待处理
            case MAIN_ORDERED:
             // 假如mainThreadPoster为空（在上篇讲过，
              // 用户可以自定义传入一个实现了MainThreadSupport接口的对象，接口中实现了一个createPoster方法，如果我们把该方法返回null这里就为null了）
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
              //后台线程模式，也就是非主线程发布。后台模式只是确保发布过程不在主线程，并不限定在哪个线程发布
            case BACKGROUND:
                //这个逻辑跟主线程是反过来的，当是主线程时则进backgroundPoster的队列中
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
              //事件异步模式，每个事件执行的前后没有顺序。
              //直接进asyncPoster的队列中。下面会继续追踪各个Poster的队列发布方式
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
              //如果设定其他threadmode，那是不识别的，会抛异常
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

上面的方法是比较核心的。它指定了需要在什么线程模式上发布，其中核心的方法是invokeSubscriber方法，另外核心的是几个poster类
我们可以看到有些是直接就调用invokeSubscriber方法进行反射调用，有些则是直接进队，并没有看到方法反射，那么事件在哪里调用呢，肯定enqueue这个方法会什么操作

这里我们先说下invokeSubscriber方法，源码如下：
```
 void invokeSubscriber(Subscription subscription, Object event) {
        try {
            //就是进行一个反射操作，拿到该方法载入调用
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            //异常时的处理
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
    //反射调用事件订阅方法时异常，则调用此方法
     private void handleSubscriberException(Subscription subscription, Object event, Throwable cause) {
            if (event instanceof SubscriberExceptionEvent) {
                if (logSubscriberExceptions) {
                    // Don't send another SubscriberExceptionEvent to avoid infinite event recursion, just log
                    logger.log(Level.SEVERE, "SubscriberExceptionEvent subscriber " + subscription.subscriber.getClass()
                            + " threw an exception", cause);
                    SubscriberExceptionEvent exEvent = (SubscriberExceptionEvent) event;
                    logger.log(Level.SEVERE, "Initial event " + exEvent.causingEvent + " caused exception in "
                            + exEvent.causingSubscriber, exEvent.throwable);
                }
            } else {
                if (throwSubscriberException) {
                    throw new EventBusException("Invoking subscriber failed", cause);
                }
                if (logSubscriberExceptions) {
                    logger.log(Level.SEVERE, "Could not dispatch event: " + event.getClass() + " to subscribing class "
                            + subscription.subscriber.getClass(), cause);
                }
                //在方法的最后会判断sendSubscriberExceptionEvent=true后发布一个事件异常的事件
                //还记得前面我们说的没找到订阅类时的异常么，在那里需要判断事件类型不为SubscriberExceptionEvent，正是因为这里可能会发出一个SubscriberExceptionEvent事件
                if (sendSubscriberExceptionEvent) {
                    SubscriberExceptionEvent exEvent = new SubscriberExceptionEvent(this, cause, event,
                            subscription.subscriber);
                    post(exEvent);
                }
            }
        }
```

invokeSubscriber没啥，就是反射载入方法。如果我们使用的是默认的线程模式，则到这里就调用invokeSubscriber方法后便回调了订阅方法了
重点还是mainThreadPoster、backgroundPoster、asyncPoster这几个Poster





# EventBus#  mainThreadPoster


首先，它实现了Poster接口：
```
/**
 * Posts events.
 *
 * @author William Ferguson
 */
interface Poster {

    /**
     * Enqueue an event to be posted for a particular subscription.
     *
     * @param subscription Subscription which will receive the event.
     * @param event        Event that will be posted to subscribers.
     */
    void enqueue(Subscription subscription, Object event);
}
```

Poster只有一个enqueue方法，表明一个入队操作。三个Poster实现类我们来一个一个看，首先看下mainThreadPoster，
mainThreadPoster在EventBus中创建的时候是这样的：
```
    mainThreadSupport = builder.getMainThreadSupport();
    mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
```

追踪EventBusBuilder#getMainThreadSupport方法的源码是：
```

    MainThreadSupport getMainThreadSupport() {
        if (mainThreadSupport != null) {
            return mainThreadSupport;
        } else if (Logger.AndroidLogger.isAndroidLogAvailable()) {
        //默认会返回一个AndroidHandlerMainThreadSupport类，并传入主线程的一个Looper
            Object looperOrNull = getAndroidMainLooperOrNull();
            return looperOrNull == null ? null :
                    new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
        } else {
            return null;
        }
    }
```
AndroidHandlerMainThreadSupport类是在MainThreadSupport类中定义的：
```
 class AndroidHandlerMainThreadSupport implements MainThreadSupport {

        private final Looper looper;

        public AndroidHandlerMainThreadSupport(Looper looper) {
            this.looper = looper;
        }

        @Override
        public boolean isMainThread() {
            return looper == Looper.myLooper();
        }
        默认返回的Poster是一个HandlerPoster
        @Override
        public Poster createPoster(EventBus eventBus) {
            return new HandlerPoster(eventBus, looper, 10);
        }
    }
```
 由此可看出，当我们并没有自定义一个Poster的时候，默认的mainThreadPoster对象是一个HandlerPoster。
 HandlerPoster继承自我们熟悉的Handler，并实现了Poster接口。
最重要的enqueue 的源码如下：
 ```
  public void enqueue(Subscription subscription, Object event) {
        //PendingPost在入队之前，还将事件和订阅类封装在PendingPost中，下面会讲解
         PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
         synchronized (this) {
            //实际入队是这个，封装成一个PendingPost对象后送入队列来
            //对象queue在HnadlerPoster构造方法中是被赋值了一个PendingPostQueue对象
            //篇幅原因不展开PendingPostQueue，它只是模拟了队列的入队出队的操作
             queue.enqueue(pendingPost);
             //暂时理解下，如果当前正在处理消息，则不做操作
             //如果当前Hnandler是空闲的，那么发送个消息出去，消息是一个来自Handler#obtainMessage，一个空的消息
             //这有什么用呢？那就得来看看是怎么处理消息的 也Handler的handlerMessage方法
             if (!handlerActive) {
                 handlerActive = true;
                 if (!sendMessage(obtainMessage())) {
                     throw new EventBusException("Could not send handler message");
                 }
             }
         }
     }

 ```
当我们发送消息出来，会出发Handler回调handleMessage方法，实现如下：
```
 @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                //取出消息
                PendingPost pendingPost = queue.poll();
                //如果没取到
                if (pendingPost == null) {
                //注意这里使用了同步快再使用了一次，为什么呢？
                //细心的同学可能发现在上面的enqueue方法中也用到了this这个锁来同步进入队列
                //那么考虑一种情况，锁被enqueue方法中拿了，然后进了队列，之后释放锁
                //而这里刚好到这一步实际又没取到，所以同步代码快只是为了让enqueue释放锁的时候这里检查一下是否队列有元素
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        //确认过这个期间没有任何入队操作，那么设置handlerActive为false并退出循环
                        //表示当前没在处理消息了，是空闲的了
                        //enqueue方法中当判断handlerActive不为true，则会触发一次消息发送处理到这里来处理事件
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                //不为空那就好办了，直接调用invokeSubscriber反射调用订阅方法了
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                //在事件处理之前和处理之后，记录了个时间差。如果超过了maxMillisInsideHandleMessage
                //则返回 重新发送消息
                //为什么要这么操作呢
                //原因是当前的Handler Looper那是主线程的消息循环啊
                //主线程不能在那里等待所有事件处理完，那样容易阻塞了主线程去处理其他事情了
                //所以每处理一个事件超过了maxMillisInsideHandleMessage，则重新发送个消息在下个消息处理，让主线程的其他任务得以执行
                //maxMillisInsideHandleMessages是多少呢？默认在MainThreadSupport的内部类AndroidHandlerMainThreadSupport中，创建Handler的时候传入的值是10
                //如果我们是自己定义的Poster，则就是自己传入的一个数额了
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
```

mainThreadPoster到这里就讲完了，对于默认的mainThreadPoster来说，可以理解为其实mainThreadPoster本身就是个持有主线程Looper的Handler
它把事件送入队列来，然后不断的发送消息 自己处理消息的时候对事件反射调用。达到了一个事件在主线程中发布的效果。之所以加了一层Handler，是因为主线程需要防止阻塞的，不能长时间在处理某个任务，导致其他的消息阻塞了

# EventBus#  backgroundPoster

追踪得好累，接下来继续讲backgroundPoster的模式。同样，我们先得看看backgroundPoster来自哪个类的对象：
```
backgroundPoster = new BackgroundPoster(this);
```
EventBus中是这么赋值的，它并不像主线程的Poster会留出接口让开发者自定义,
还有他并没有继承Handler，而是实现了Runnable。因为后台线程通常又不怕阻塞，生命周期短，没必要用Handler
所以源码中采用的是线程池的方式
我们直接看BackgroundPoster类的enqueue方法，因为事件是在这个方法传进来的：
```

    public void enqueue(Subscription subscription, Object event) {
    //这里同样用个PendingPost来封装下事件和订阅者
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
        //在同步代码快中入队
            queue.enqueue(pendingPost);
            //跟HandlerPoster的这里有点像，也采用了个executorRunning参数来判断当前是否在运行（活跃中）
            if (!executorRunning) {
                executorRunning = true;
                //如果当前没有在运行中，则出发一次线程池来执行
                // 追踪eventBus.getExecutorService()来自于默认的 Executors.newCachedThreadPool()
                //或者是自己用EventBusBuilder时传入的ExecutorService
                //既然交给了线程池，本身又是个Runnable 那么重点就在run方法了
                eventBus.getExecutorService().execute(this);
            }
        }
    }
```

我们看下run方法，源码如下：
```
 @Override
    public void run() {
        try {
            try {
                //循环取事件
                while (true) {
                    //这里的1000是个啥，点击进去追踪下发现是执行一个等待操作
                    //也就是说这个方法是阻塞1000ms的
                    //为什么要阻塞1000ms？这里我个人认为只是为了避免重复线程任务
                    //没有这个阻塞的话，队列完毕后则线程就结束了，下次得继续执行任务
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            //双层校验，避免这个时候有TMD刚好有入队操作
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    //拿到后立马反射调用
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            //这里也有必要说下，最终是设定标记位为不在运行中
            //考虑一种情况，假如上面的事件循环被打断了呢，也就是到了InterruptedException怎么办
            //没怎么办，打断就打算吧，反正要是打断的话我也设置当前不是在运行中
            //等线程池下次任务执行又可以继续事件循环了
            //由此另一方面可以看出，在一次处理队列中的消息并不一定都是由同个线程执行
            //EventBus保证的是事件按队列方式先进先出，串行的方式来执行
            executorRunning = false;
        }
    }
```


# EventBus#  asyncPoster

追踪得好痛苦 ，老套路，先看下asyncPoster的创建：
```
asyncPoster = new AsyncPoster(this);
```

那就追踪下AsyncPoster这个类，与BackgroundPoster类似，也是实现Runnable和Poster

enqueue和run方法如下：
```
   public void enqueue(Subscription subscription, Object event) {
        //看上去好简单，直接入队然后进去线程池
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }

//直接取队列后执行，并不管哪个Runnable会被先执行，反正先被线程池执行的Runnable就先去取队列然后反射调用订阅方法
    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }
```

这个简单了吧，与BackgroundPoster相比，相同的地方是同样使用的是非主线程的方式。不同在于AsyncPoster中一个Runnable只会执行一个事件
所以几个事件一起送入任务中，就不确保哪个先执行了，这个正是异步事件的模式了。





讲解完毕了，
我们再来滤清下方法流程，
首先从**post**方法，传入具体事件

post方法里调用了**postSingleEvent**方法，传入该事件、线程对应的事件处理状态
postSingleEvent方法用于查出事件的继承链，根据每个事件类型来发布


postSingleEvent方法里调用了**postSingleEventForEventType**方法，传入该事件、线程对应的事件处理状态、事件类型
postSingleEventForEventType方法用于找出时间类型对应的订阅者列表，对每个订阅者进行发布



postSingleEventForEventType方法里调用了**postToSubscription**方法，传入订阅者、事件、是否在主线程
postToSubscription方法用于区分在不同的线程模式下进行反射调用订阅方法



postToSubscription中调用了**invokeSubscriber**方法最终反射调用订阅方法
或者是进去不同的Poster的队列



值得思考的地方：
1、源码中大量用了关于同步代码块的处理，有兴趣的同学可以详细的研究每个同步块的初衷
2、异常处理
3、优化处理

关于EventBus的源码解析，有出错之处烦请留言指出。

或者您对源码有什么疑问，请在下方留言与我探讨吧。















