---
tags: Android 
read: 1056   
---
@[toc]
# EventBus前言

本文主要讲解EventBus的源码解析，如果您未听过/使用过EventBus的话请自行百度一下，几分钟即可GET到这门技能。EventBus的好处是实现类与类之间通讯的解耦：内部采用观察者模式（发布-订阅模式），该模式可以把发布者和订阅者解耦出来，两者并不需要做直接交互。
然而有利有弊，EventBus使用不当则容易造成代码难以解读，很多时候你并不知道订阅者会在什么时候接收消息（收到来自哪些发布者的消息），对于发布者来说，也不清楚发布消息后会被哪些订阅者接收（是想要通知哪个订阅者）。所以具体还是要衡量在某些情景下是否使用EventBus。
本节剖析的是EventBus的register和unregister流程，下篇再讲事件订阅分发过程（下篇的传送门： [框架源码解读系列之《EventBus3.1.1源码解析（下篇）》](https://blog.csdn.net/lc_miao/article/details/86480393).）

下面的源码解析是基于EventBus3.1.1的源码：
```
 implementation 'org.greenrobot:eventbus:3.1.1'
```
# EventBus#对象构建过程

通常来说，我们大多数时候是在Activity或Fragment中来注册订阅，也就是的调用EventBus#register方法，只需：

```
  EventBus.getDefault().register(this);
```
首先，我们先看一下EventBus.getDefault()：
```
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
 ```

由此得知，getDefault方法内部使用了双重校验锁来构建一个默认的EventBus实例，作为我们使用的默认EventBus对象。
而通常我们会封装一个EventBus，去定义一些特性，便使用了它的builder方法来构建一个新的实例：
```
    //使用builder设计模式来设置一些属性
   EventBus bus = EventBus.builder()
   //直接采用反射查找注解方法
   .ignoreGeneratedIndex(true)
   //检查注解方法正确性
   .strictMethodVerification(true)
   //构建一个新的实例
   .build();
   //之后再 调用
   bus.register(this);
 ```
 我们看看EventBus#builder方法：
 ```
   public static EventBusBuilder builder() {
         return new EventBusBuilder();
     }
```
该方法只是返回了一个新的EventBusBuilder实例，我们知道有一种设计模式叫做Builder设计模式，由于需要构造的参数比较多也不确定个数，如果写太多构造器的话代码可读性也很差，所以引用了Builder设计模式，将参数的构建转移到另外一个类，之后再直接把Builder对象传进来解析出各个对象，避免了重载多个构造器。

这里也一样，创建了个EventBusBuilder之后我们就可以一一设置我们要的属性.
我们看一下这个类的属性,我把各个参数简要的说明备注在下面的代码里：
```
public class EventBusBuilder {
    private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();
    //事件处理异常时是否打印异常信息，一般来说我们都需要这个打印，所以都是true
    boolean logSubscriberExceptions = true;
    //当没有订阅者订阅该事件时是否打印日志（也就是Post出去后没人接收），一般也是true
    boolean logNoSubscriberMessages = true;
    //当事件处理方法异常时是否发送 SubscriberExceptionEvent 事件，若此开关打开，订阅者可通过onEvent来处理
    boolean sendSubscriberExceptionEvent = true;
    //当消息事件发送出去后没有对应的处理方法时，是否发送 NoSubscriberEvent 事件，若此开关打开，订阅者可通过onEvent来处理
    boolean sendNoSubscriberEvent = true;
    //当调用事件处理方法异常时是否抛出异常
    boolean throwSubscriberException;
    /*是否支持事件继承，比如事件A 继承了 B，那么发出事件A后，订阅了B的订阅者也会接收到
    */
    boolean eventInheritance = true;
    /**
     * 是否忽略索引，也就是在获取注解方法的时候，是否直接通过记录的特定注解方法列表，还是用反射遍历所有的方法
     * 涉及到反射的效率问题，而且3.0之后引入了annotation preprocessor，我们可以生成注解方法列表，这样EventBus就不用一个个去遍历类里面的方法
     */
    boolean ignoreGeneratedIndex;
    /**
    *注解的方法是否严格限制正确性，我们知道Subscribe注解描述的方法（也就是事件处理方法）要求:
    * 1.访问级别是public;
    * 2.参数列表有且仅有一个;
    * 3.非抽象，非静态。
    如果此处设置为true，那么没按照这个规定的注解方法在注册时会抛出异常。
    */
    boolean strictMethodVerification;
    //发送事件的线程池
    ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;
    //对类忽略方法校验，忽略检验方法正确性的类名单
    List<Class<?>> skipMethodVerificationForClasses;
    //通过annotation preprocessor生成的注解方法列表，避免了反射获取注解方法，注意ignoreGeneratedIndex=false才有用（默认已经是false）
    List<SubscriberInfoIndex> subscriberInfoIndexes;
    //自定义日志的输出
    Logger logger;
    //事件处理的消息循环
    MainThreadSupport mainThreadSupport;

    ...
  /** Builds an EventBus based on the current configuration. */
         public EventBus build() {
             return new EventBus(this);
         }
 }
```
关于EventBus构建时的参数设置已经做了声明，我们只需要调用对应的build方法即可对设置参数值，最后调用build方法，build方法返回的是new EventBus(this);
也就是把EventBusBuilder对象作为参数传入了EventBus构造器中，我们看下EventBus的构造器：
```
   EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        /*用HashMap记录每个事件对应的订阅列表，
         *subscriptionsByEventType对象的定义是：Map<Class<?>, CopyOnWriteArrayList<Subscription>>
         */
        subscriptionsByEventType = new HashMap<>();
        //记录所有注册订阅的类以及对应的订阅事件类型（记录所有的订阅者）
        typesBySubscriber = new HashMap<>();
        /**
         *记录所有粘性事件类型，粘性事件：在消息发布之后再注册时也能接受到消息，例如粘性广播
         *多线程环境下可能同时发生遍历和添加，所以用并发的HashMap，不了解ConcurrentHashMap的同学建议百度了解一下
         */
        stickyEvents = new ConcurrentHashMap<>();
        //见下文代码解读getMainThreadSupport
        mainThreadSupport = builder.getMainThreadSupport();
        //获取主线程的消息发送队列，实际上为一个Handler
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        //创建一个后台线程的消息发送队列，当threadMode为BACKGROUND时才用这个队列，后面讲解消息订阅时再细讲
        backgroundPoster = new BackgroundPoster(this);
        //创建一个异步消息发送队列，当threadMode为ASYNC时才用这个队列，后面讲解消息订阅时再细讲
        asyncPoster = new AsyncPoster(this);
        //以下则是我们熟悉的builder设计模式了，上面我们将吧参数封装在EventBusBuilder里，现在把EventBusBuilder对象传进来一个一个取参数值
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```
代码解读getMainThreadSupport：

关于EventBusBuilder的getMainThreadSupport源码解读如下：
```
MainThreadSupport getMainThreadSupport() {
        //如果我们自己传入了MainThreadSupport则直接返回
        if (mainThreadSupport != null) {
            return mainThreadSupport;
         //这里isAndroidLogAvailable方法判断Android SDK里包含android.util.Log类没有
        } else if (Logger.AndroidLogger.isAndroidLogAvailable()) {
        //通过getAndroidMainLooperOrNull方法取得一个主线程的消息循环对象，该方法只是调用了 Looper.getMainLooper()
            Object looperOrNull = getAndroidMainLooperOrNull();
            //不为空则利用这个消息循环对象构建出AndroidHandlerMainThreadSupport对象，也就是MainThreadSupport的子类
            return looperOrNull == null ? null :
                    new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
        } else {
            return null;
        }
    }
```


到这里我们就已经讲完了构建EventBus时的这些配置参数。






# EventBus#register(Object subscriber)

理解了EventBus的构建过程，接下来就是解析register的时候了，我们直接从register方法下手:
```
    public void register(Object subscriber) {
        //获取类
        Class<?> subscriberClass = subscriber.getClass();
        //获取注解方法列表
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        //在同步块里把注解的方法列表添加到EventBus中来
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
当然，要是想像上面我这样注释来理解代码，那就等于没说了。重点就是SubscriberMethodFinder#findSubscriberMethods方法和EventBus#subscribe方法

先来看SubscriberMethodFinder这个类，从名称看上去字面理解就是一个用于查找注解方法的类。
我们直接看它的这个findSubscriberMethods方法：
```
 List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //从缓存中获取注解方法列表。一个类可能前前后后被注册多次，那么缓存起来下次就不会再去查找注解的方法了
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        //查找到缓存就直接返回
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        //这个参数不知道您还记得不，上面我们讲Build时说过这个参数
        //如果直接说是要不要忽略方法索引的意思，可能不好理解
        //换个角度可以把它理解成要不要使用反射来获取注解方法列表
        if (ignoreGeneratedIndex) {
            //通过反射的方式来获取方法列表
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            //通过生成的注解方法列表来读取
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        //如果没找到 @Subscribe的注解方法，则抛出异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            //好不容易查找了一次，当然要放进缓存下次直接拿了
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```
上面就注解方法查找的大纲流程，里面最需要了解的就是findUsingReflection方法和findUsingInfo方法了，两个不同方式的获取注解方法。
讲到这里的时候，下面会出现了一个新的类：FindState  ， 我们先暂时把这两个方法的流程走完再来详细讲解FindState这个类。
先看反射获取注解的，findUsingReflection方法：
```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        //FindState是一个方法查找状态的类，通过prepareFindState方法来池中取一个，或者是new出来一个
        FindState findState = prepareFindState();
        //把类传进来初始化查找
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            //最重要的就是这个方法了，该方法的注释看下面
            findUsingReflectionInSingleClass(findState);
            //移动到它的上一层父类
            findState.moveToSuperclass();
        }
        //释放findState，并添加到状态池中
        return getMethodsAndRelease(findState);
    }
//重点就是这个方法：
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        //获取一个类的方法数组，getDeclaredMethods是获取类里声明的公共方法（包括接口实现方法，但不包括继承父类的方法）
        //getMethods则获取的是这个类所拥有的所有公共方法（包括接口、继承）
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //遍历数组列表判断是否属于注解方法
        for (Method method : methods) {
            //获取方法的修饰
            int modifiers = method.getModifiers();
            //如果方法是Modifier.PUBLIC即公共方法，
            //MODIFIERS_IGNORE点进去看，是MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC
            //也就是方法不是抽象的也不是静态的也不是BRIDGE的（泛型后编译器生成对应的类型）也不是SYNTHETIC的（编译器生成的）
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                //获取该方法的参数列表
                Class<?>[] parameterTypes = method.getParameterTypes();
                //如果参数列表是1个（这里之所以限定1个，主要是因为后面消息发送出来好对应类型。如果有需要接收多参数，那么也可以在参数类里面组合进来）
                if (parameterTypes.length == 1) {
                    //判断该方法是否有用Subscribe注解
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    //如果有
                    if (subscribeAnnotation != null) {
                        //拿到该参数列表
                        Class<?> eventType = parameterTypes[0];
                        //我们还没细讲findState，这个checkAdd方法先留着，它就是检查方法是否已经被添加进来过
                        if (findState.checkAdd(method, eventType)) {
                            //指定的注解线程模式，我们在方法注解的时候可以指定线程模式（主线程 后台 异步等）
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //添加到subscriberMethods列表
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                 //如果参数列表不是1个，而又有Subscribe注解，那么当trictMethodVerification为true时会抛出异常
                 //我们前面讲Builder的时候说过这个strictMethodVerification参数了，就是获取注解方法错误时是否需要抛出异常
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
             /如果方法不是public，或者是抽象、桥接、生成的，而又有Subscribe注解，那么当trictMethodVerification为true时会抛出异常
             //我们前面讲Builder的时候说过这个strictMethodVerification参数了，就是获取注解方法错误时是否需要抛出异常
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

讲完了反射获取注解方法列表的方法，脑子里已经有个大体的理解，也就是去获取注解的方法列表后添加进来
我们还有另外一个获取注解方法的方法还没讲，findUsingInfo方法，代码如下：

```
 private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //同上
        FindState findState = prepareFindState();
        //同上
        findState.initForSubscriber(subscriberClass);
        //findState.clazz记录的是这个类的当前类或者是父类
        while (findState.clazz != null) {
            //获取注解信息，Eventbus3.0后引入的annotation preprocessor，编译器自动生成的方法注解列表
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                //直接取得注解方法列表，有区别于上面讲解的反射获取过程，这个方法看起来省去了反射，貌似效率高一点。事实是如此
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                //把注解方法列表添加进来
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
            //假如没获取到注解方法列表，那么无奈只能用反射了
                findUsingReflectionInSingleClass(findState);
            }
            //把findState.clazz移动到上一级父类
            findState.moveToSuperclass();
        }
        //释放findState
        return getMethodsAndRelease(findState);
    }
```

两个查找注解方法的方法已经讲完，区别最大的就是一个是通过反射来获取注解，一个是通过前期编译器自动生成的注解列表，在运行期自动读取。
反射成本是很大的，显然后者效率要高很多。

到这里把获取注解方法添加到subscriberMethods列表中来，也就完成了注册的过程了。

剩下的就是那个貌似也挺核心的FindState类，它是一个SubscriberMethodFinder的静态内部类。用于记录注解方法的查找状态
```
 static class FindState {
    final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
    final Map<Class, Object> anyMethodByEventType = new HashMap<>();
    final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
    final StringBuilder methodKeyBuilder = new StringBuilder(128);
    //被描述的类
    Class<?> subscriberClass;
    //是否跳过父类
    boolean skipSuperClasses;
    //自动生成的注解信息
    SubscriberInfo subscriberInfo;
    //初始化传入被描述的类
    void initForSubscriber(Class<?> subscriberClass) {
                this.subscriberClass = clazz = subscriberClass;
                //默认是不跳过
                skipSuperClasses = false;
                subscriberInfo = null;
            }
     //回收，此处之所以有回收的操作，是因为有FindState池在引用，避免重复创建多余的FindState
     //还记得上面的源码中的一句getMethodsAndRelease么，在那个方法里面调用了recycle方法后，又将放回去FindState池中供下一次使用
  void recycle() {
         subscriberMethods.clear();
         anyMethodByEventType.clear();
         subscriberClassByMethodKey.clear();
         methodKeyBuilder.setLength(0);
         subscriberClass = null;
         clazz = null;
         skipSuperClasses = false;
         subscriberInfo = null;
      }
    //这个方法我们上面轻描淡写的说了它作用就是检查这个方法是否已经被记录过了，
    //返回true则说明需要被添加，返回false则说明无需再添加
    //那么它是怎么检查的呢？见下面注释
    boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
            //anyMethodByEventType是一个HashMap，把方法存进去后拿返回值，如果不为空，则说明之前已经有过这个key-value
            Object existing = anyMethodByEventType.put(eventType, method);
            //如果没存过的话则返回true
            if (existing == null) {
                return true;
            } else {
                //校验一下确保返回的这个value类型是Method，要不万一是其它类型的value那就尴尬了
                if (existing instanceof Method) {
                    //第二层方法校验，进行方法签名检查
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);
                }
                 //第二层方法校验，进行方法签名检查
                return checkAddWithMethodSignature(method, eventType);
            }
        }
   //方法签名校验，主要是确保不要重复添加方法，另外一个是子类和父类之间，加入子类实现了父类的注解方法，那么就会代替父类的注解方法。
 private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
            //拼接：采用方法名和一个> 符号 再加上事件类型
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());
            //吧拼接的这个作为key，也就是作为签名
            String methodKey = methodKeyBuilder.toString();
            //获取方法对应的类
            Class<?> methodClass = method.getDeclaringClass();
            //subscriberClassByMethodKey是记录已经经过签名校验后而添加过的类
            //通过subscriberClassByMethodKey获取签名对应的类
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
            //如果没被添加过，或者添加过，但属于目前要添加进来的methodClass这个类自身或者子类，那么就需要被添加
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
                // Only add if not already found in a sub class
                return true;
            } else {
                // Revert the put, old class is further down the class hierarchy
                //否则记录已经添加 并返回false表示不需要添加了
                subscriberClassByMethodKey.put(methodKey, methodClassOld);
                return false;
            }
        }
   //在查找注解方法时，用到了这个移动到上一级父类的操作。实际上一个含有注解的类，可能是它的父类所注解的，所以需要遍历它的所有父类方法
  void moveToSuperclass() {
            if (skipSuperClasses) {
                clazz = null;
            } else {
                clazz = clazz.getSuperclass();
                String clazzName = clazz.getName();
                //注意这里，这几个开头的类就不用在遍历，因为都是performance的，我们当然不可能去注解它
                /** Skip system classes, this just degrades performance. */
                if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
                    clazz = null;
                }
            }
        }
 }
```

到这里我们只是讲完了把注解的方法添加到EventBus中来，那么添加进来之后也得放入一个监听状态吧，不然怎么接收呢。
花了这么大的篇幅，就是讲上面我们在register中遇到的SubscriberMethodFinder#findSubscriberMethods方法
而register方法还有个重点的方法，那就是subscribe了，这里先大概说一下它的作用：就是把添加进来的注解方法送入观察者队列，其中也判断了这个事件类型是否属于粘性事件，如果是的话则要进行接收操作。

```
//需要注意的是，源码也进行了下面的注释，说明该方法需要在同步块使用，引用方法里面对集合进行了读和写操作
// Must be called in synchronized block
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
            //获取订阅方法对一个的事件类型
        Class<?> eventType = subscriberMethod.eventType;
        //构建一个订阅类，包含了订阅者、注解的订阅方法
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        //取出该事件类型中的订阅列表
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        //如果该类型还没被订阅过，则创建一个列表放入到<事件，订阅列表>的map中来
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
             //如果已经有过订阅列表，检查一下该订阅是否已经添加过，避免重复添加
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        //根据优先级送入订阅列表，默认都不设置优先级则插入到末尾
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        //获取该订阅类的所有事件
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        //把事件也插入到<类-事件>的map中来
        subscribedEvents.add(eventType);
        //检查该注解方法是否注解为粘性事件，如果是则需要做接收事件的处理
        if (subscriberMethod.sticky) {
        //默认是支持事件继承的，比如方法接收的事件类型是A，而A继承了B，那么有A事件时也会有onEvent(B)
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                //取出所有的事件类型
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                //遍历事件
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    //取出事件类型
                    Class<?> candidateEventType = entry.getKey();
                    //如果取出的事件类型是属于此次我们注解的方法事件类型的子类的话，则也需要把事件post出来
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        //内部判空后发送事件，关于事件发布的下篇再讲
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                  //内部判空后发送事件，关于事件发布的下篇再讲
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```


讲到这里，我们就大致清楚了整个注册的流程，在滤清楚之后，我们需要对里面两个集合容器的作用有个清晰的认知：

//记录了所有的事件以及事件对应的订阅列表，通过这个集合就可以查看到具体都有哪些事件，每个事件具体都被哪些类所订阅
 private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType subscriptionsByEventType;

 //记录所有注册订阅的类以及对应的订阅事件类型（记录所有的订阅者）
 //通过这个集合，我们就可以知道目前具体都有哪些订阅者，每个订阅者都订阅了哪些事件类型
 private final Map<Object, List<Class<?>>> typesBySubscriber;


 经过上面的分析，Register的流程就走完了。对EventBus来说，它就已经知道了当前一共有哪些订阅者，一共有哪些事件，每个订阅者订阅了哪些类型，每个事件被那些订阅者所订阅。
 当清楚了这个之后，对于下面对事件类型去定向发送，就很容易了。篇幅问题，我们把事件订阅过程留在下篇讲。



# EventBus#unregister(Object subscriber)

不急着结束，我们讲完了register后，unRregister还没讲呢。实际上，我们滤清了subscriptionsByEventType和typesBySubscriber之后，那么unregister的流程变得特别简单了。

我们直接从unregister方法入手：
```
    public synchronized void unregister(Object subscriber) {
      //取出该订阅者的订阅消息类型列表
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        //如果有订阅过
        if (subscribedTypes != null) {
            //遍历取消掉这个订阅者订阅过的每个消息类型
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            //并且移除出订阅者列表
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```
看下unsubscribeByEventType方法：
```

    /** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
      //取出事件类型对应事件订阅列表
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            //遍历该事件的订阅列表
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                //把对应了该订阅者的事件，移除出事件订阅列表
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```


完事了，下篇我再讲EventBus的事件发布流程。

下篇文章链接入口： [
框架源码解读系列之《EventBus3.1.1源码解析（下篇）》](https://blog.csdn.net/lc_miao/article/details/86480393).













