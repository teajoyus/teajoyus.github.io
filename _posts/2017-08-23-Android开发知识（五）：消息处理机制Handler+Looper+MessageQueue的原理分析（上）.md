---
tags: Android 
read: 2354   
---
&emsp;&emsp;提到Android的消息处理机制，相信大家并不陌生。因为我们在开发中基本会运用到消息处理，比如在子线程我们做了耗时的网络访问操作，然后通过Handler对象的一个sendMessage()方法就可以在主线程上回调handlerMessage()方法来让我们完成UI的更新。

&emsp;&emsp;那么，读者是否考虑过这个问题：似乎在这个过程，只涉及到Handler对象，加上Message对象作为消息载体。那这两个线程是怎么关联上的呢？为何子线程调用Handler对象的方法，能回调在主线程上的方法？为何在子线程调用Handler对象的post方法，而传入的Runable对象的run方法是在主线程上执行的？

&emsp;&emsp;带着这些问题，我们展开对消息处理机制的原理分析。事实上，Android的消息处理机制远不止Handler、Message。其内部的实现原理比较重要的类还有Looper类、MessageQueue类、ThreadLocal类。

&emsp;&emsp;在分析原理之前，我们有必要先来介绍一下以上各个类，只有先了解了这几个类的作用以及原理，我们才能以此为基础来理解消息处理机制。

----------


 1. Handler
----------


 &emsp;&emsp;关于Handler也应该不用多说了吧，Handler往往被我们作为UI更新的工具，实际上从它诞生的那一刻起，它就不是为了UI更新而生，而是为了处理线程之间的消息通信。更新UI只是消息处理其中一个典型的例子。Handler使用起来也特别简单，通过一系列send方法：
 
![send](http://img.blog.csdn.net/20170823115657688?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

比如用如下代码：

```
 public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";
    Handler handler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            Log.i(TAG, "handleMessage: "+msg.what);
            Log.i(TAG, "handleMessage thread: "+Thread.currentThread());
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread("demo-thread-1"){
            @Override
            public void run() {
                super.run();
               handler.sendEmptyMessage(1);

            }
        }.start();
    }
}
```
&emsp;&emsp;可以看到在子线程调用handler.sendEmptyMessage(1)，即可在主线程中打印出消息。

&emsp;&emsp;当然，这不意味着这一系列send方法一定要是在子线程中执行，同时handlerMessage方法也不一定是在主线程上执行。这将放到后面再详细说明，在这里我们只需要了解Handler可以用来发送消息和接收消息实现线程通信就可以了。

&emsp;&emsp;我们也可以在子线程通过使用Handler的post方法来进行UI更新：
```
new Thread("demo-thread-1"){
            @Override
            public void run() {
                super.run();
              handler.post(new Runnable() {
                  @Override
                  public void run() {
                      //TODO
                      //main  thread
                      Log.i(TAG, "run: "+Thread.currentThread());
                  }
              });

            }
        }.start();
```
&emsp;&emsp;通过post方法，我们可以在run方法里面写各种UI的更新，因为在run方法里运行的是UI线程。
&emsp;&emsp;总的来说，Handler使用起来非常的简单也方便。

&emsp;&emsp;但是这里有一处地方值得注意说明，就是我们这样子new出Handler对象，编译器会给我们一个警告：
![警告](http://img.blog.csdn.net/20170823150430061?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
&emsp;&emsp;也就是提示我们这个Handler对象类要是静态类，否则的话会可能发生泄漏。为什么呢？我们知道，内部类会持有外部类的引用，如果内部类生命周期比较长，就会导致外部类在内部类回收之前无法回收，而静态内部类则不存在这个问题。那么，为什么Handler类生命周期会那么长呢？由于此处涉及到源码上的原理分析，所以我们放到后面再说，这里我们重点先来解决一下这个泄漏问题。

&emsp;&emsp;根据它的提示，我们只需要修改为静态的就好，那么问题又来了，handlerMessage方法里面可能需要activity类的动态方法，而又要我的Handler类是静态的，这不是啃爹吗？
你看看：
![static](http://img.blog.csdn.net/20170823151131674?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;这的确有点尴尬，但也不是没有办法解决，我们只是为了解决内部类持有外部类的引用而导致的内存泄漏问题，这普遍都可以采用弱引用的方式来防止泄漏，代码如下：

```
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";
      Handler handler = new MyHandler(this);
    class MyHandler extends Handler{
        WeakReference<MainActivity> mActivity;
        MyHandler(MainActivity activity) {
            mActivity = new WeakReference<>(activity);
        }
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if(mActivity.get()==null){
                return;
            }
            mActivity.get().handlerXX();
            Log.i(TAG, "handleMessage: "+msg);
            Log.i(TAG, "handleMessage thread: "+Thread.currentThread());
        }
    };
    private void handlerXX(){

    }
}
```
&emsp;&emsp;我们把Handler类写法换一下，把匿名内部类的形式改成内部类，然后使用一个WeakReference<MainActivity> mActivity;来存入对MainActivity的弱引用。然后通过mActivity.get()就可以返回MainActivity的引用，从而使用MainActivity的非静态方法。

&emsp;&emsp;需要说明的是在handlerMessage方法里面对Activity.get()进行空判断，这部分代码是非常必要的，因为如果MainActivity被销毁回收了，那么这个引用也就不存在了，不判断为空的话则会引发NullPointerException，如果引用不存在了，那么就直接返回。事实上，MainActivity都已经不存在了，也就没必要再处理接下来的消息。


2. Message
----------
&emsp;&emsp;Message相信也不用怎么多说，毕竟它确实比较好理解，也很容易使用。Message作为消息运输的载体，它为我们提供了what、arg1、arg2、obj、bundle等参数，使得我们可以利用what参数来指定消息代码，以让handler可以根据消息代码处理不同的业务逻辑。用arg1、arg2、obj来传输一些数据，也可以运用强大的Bundle对象来装载数据。

&emsp;&emsp;比如我们上面的代码中发送了一个空消息，我们可以写成：

```
 new Thread("demo-thread-1"){
            @Override
            public void run() {
                super.run();
               Message msg = handler.obtainMessage();
                msg.what = 1 ;
                msg.obj = new Object();
                Bundle bundle = new Bundle();
                msg.setData(bundle);
                handler.sendMessage(msg);

            }
        }.start();
```
&emsp;&emsp;根据我们业务需要来放入任意数据到Message中，需要注意的是放入Bundle中的自定义类对象必须先实现序列化。如果我们只需要传输个整数型或者一个对象，则可以使用arg1、arg2、obj，这样会比使用bundle效率更高。如果有比较多的数据种类的话，才用Bundle，然后将Bundle对象放入setData中。

3. MessageQueue
----------

 &emsp;&emsp;如果说Message是消息运输的载体，那么MessageQueue可以说是消息的仓库。从名字上看可以看出来是一个消息队列的意思，这没错。
 
 &emsp;&emsp;在我们发送消息的时候是存入这个消息队列里，然后处理消息的时候再取出来。实际上，它虽然叫做消息队列，但不意味着他的内部实现是一个队列。从源码角度来看，MessageQueue是基于单链表的数据结构来存储消息的，在Message类里面会有next参数，其代表的也正是指向下一个Message的引用。这和我们在C/C++里面学习的结构体链表，用next指向下一个结构体指针的原理是相似的。在MessageQueue类里面有两个重要的方法我们必须了解一下，就是enqueueMessage()方法和next()方法，分别对应了队列的“入队”和“出队”操作。相对源码来说next()方法的代码逻辑更为复杂，但是笔者认为这没必要去追究代码具体的含义，实际上这两个方法对应的就是链表的插入还有删除的操作。
 
 &emsp;&emsp;遗憾的是，这个入队出队的方法的访问权限是包访问权限，我们的应用中无法使用这个类来入队和出队（当然除了反射），实际上这个类也是消息处理的特定类，对开发者来说消息的入队出队过程完全是透明的，我们也不需要关心。
 
 4. ThreadLocal
----------

&emsp;&emsp;这东西可能很多人没听说过，因为它在我们开发中的确用得很少。但是不得不说它确实是个好东西。实际上ThreadLocal是一个线程内部的数据存储类。它最大的作用也是特点就是里面存储的数据只能在线程内部可见，每个线程都独立拥有自己的一份数据，线程之间数据互不影响，也互不可获取到数据。

&emsp;&emsp;为了更为直观的认清ThreadLocal的作用，我们来写一个例子。

&emsp;&emsp;我们创建了一个ThreadLocal对象，然后在不同的线程对ThreadLocal对象进行赋值，并用Log查看赋值前后的ThreadLocal对象的值：

```
public class ThreadLocalActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";
    ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread("Thread-1"){
            @Override
            public void run() {
                    Log.i(TAG, "run before: " + Thread.currentThread().getName() + " Integer = " + threadLocal.get());
                    threadLocal.set(10);
                    Log.i(TAG, "run after: " + Thread.currentThread().getName() + " Integer = " + threadLocal.get());
                    try {
                        sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
            }
        }.start();
        new Thread("Thread-2"){
            @Override
            public void run() {
                    Log.i(TAG, "run before: " + Thread.currentThread().getName() + " Integer = " + threadLocal.get());
                    threadLocal.set(20);
                    Log.i(TAG, "run after: " + Thread.currentThread().getName() + " Integer = " + threadLocal.get());
                    try {
                        sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
            }
        }.start();


    }
}
```
运行打印log截图：
![threadlocal log](http://img.blog.csdn.net/20170823144633555?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
&emsp;&emsp;从Log中我们看到，当线程1开始执行的时候，一开始ThreadLocal没有值所以打印出null，然后赋值为了10，就打印出了10.

&emsp;&emsp;而轮到线程2执行的时候，先打印出了ThreadLocal的值为null，说明ThreadLocal对象并没有因为线程1赋值为了10而这里就为10。因为在线程2里面拥有自己独立的一个Integer，并不与线程1的Integer有任何的联系。而Integer还没赋值过所以是null。

&emsp;&emsp;所以这里相信读者也就明白了ThreadLocal本质上的作用：相同的ThreadLocal对象在各个线程中独立拥有自己的一份存储数据。

&emsp;&emsp;不得不说，ThreadLocal最重要的两个方法则是set方法和get方法，通过set方法可以保存该线程的数据，通过get方法可以取得该线程持有的数据。

 5. Looper
----------

&emsp;&emsp; Looper类相信对于一些不太熟悉Handler的读者来说，算是比较陌生的，毕竟在子线程操作完成后发送消息到主线程更新UI这个过程，我们的代码并不涉及到Looper。

&emsp;&emsp;MessageQueue是一个消息存储单元，而不处理消息，Looper则扮演了消息循环的角色，用来处理MessageQueue中的消息。而实际上，每个线程都可以有自己的消息循环，也就是说每个线程拥有自己独立的一个Looper，说到这里是否有点熟悉感？没错，这和我们上面提到的ThreadLocal类对应起来了，正是因为了ThreadLocal，才使得每个线程可以拥有自己独立的一份消息循环。大家都各自处理自家的消息，而不打扰他家的事情（消息）。

&emsp;&emsp;在这篇关于消息处理机制的基础篇中，更多的只是认识关于消息处理我们必须了解的类，让读者有一个基础认知。在下一篇博文中我们再来详细的剖析这些类的源码，从而分析消息处理的原理。
