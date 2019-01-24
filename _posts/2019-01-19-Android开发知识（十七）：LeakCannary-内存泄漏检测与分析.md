---
tags: Android 
read: 1056   
---

@[toc]


LeakCannary介绍
-------------
LeakCannary来自Square开发的一个可视化内存泄漏分析工具，github链接：https://github.com/square/leakcanary
通过让LeakCannary框架注入到你的程序中，在检测到可疑的内存泄漏现象时会发出通知（Nofitycation）给你，并且桌面会增加一个图标，点击可以看到内存泄漏的引用链，由此可以帮助我们来排查产生泄漏的根源



LeakCannary使用演示
---------------
添加依赖包：
```
    implementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
    // 如果你想检测fragment中的泄漏，可以添加这个依赖
    implementation 'com.squareup.leakcanary:leakcanary-support-fragment:1.6.3'
```

然后我们创建个继承Application的类，并且在Manifests文件中添加进来：
```
public class LeakApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        //泄漏分析自身也会有个进程，这里判断如果是那个泄漏分析进程的话则不需要进行分析
        if(!LeakCanary.isInAnalyzerProcess(this)){
             LeakCanary.install(this);
        }
    }
}
```

上面的代码，只需要一句 LeakCanary.install(this);就可以注入进来，非常简单。
注意：通过LeakCanary.install(this)来检测Activity是否泄漏是**Android4.0**之后才可以的，因为源码内部的原理是通过Application的registerActivityLifecycleCallbacks()方法来监听Activity的生成周期，
而因为源码内部的原理是通过Application的registerActivityLifecycleCallbacks这个方法是在API 14之后才加起来的
如果需要在API 14之前的版本添加的话，则需要自己在Activity的onDestory方法中调用RefWatcher.watch(this) 来实现由于内存分析会有另外一个进程，所以判断下LeakCanary.isInAnalyzerProcess(this)

在内存泄漏案例中，通常造成Activity泄漏的案例特别常见，由于我们很多地方需要用到Context，导致Context有可能被其他生命周期更长的对象多持有，导致Activity在销毁后也无法回收
下面简单演示这个案例，我们编写个静态对象持有Context的例子：
```
public class UtilsManager {
    public  Context context;
    public static UtilsManager manager;

    public UtilsManager(Context context) {
        this.context = context;
    }

    public static UtilsManager getInstance(Context context){
        if (manager == null) {
            synchronized (UtilsManager.class) {
                if(manager==null){
                    manager = new UtilsManager(context);
                }
            }
        }
        return manager;
    }
}
```
例子很简单，我们往往会写这么一个单例对象，而这个单例的静态对象持有Context的引用

创建一个SecondActivity，并且在onCreate里面传入Conext给这个单例对象：
```
  @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //context被静态的UtilsManager所持有而造成泄漏
        UtilsManager.getInstance(this);
        TextView tv = findViewById(R.id.tv);
        tv.setText("我是SecondActivity");

    }
```

然后我们在MainActivity中添加个点击事件跳转到SecondActivity，并且退出来。
正常来说，退出来后SecondActivity会执行onDestory方法之后被回收。

运行一下，效果如下：

![泄漏运行图](https://img-blog.csdnimg.cn/20190118111318585.gif)

可以发现，当我们点击进去SecondActivity后，退出来。会有个自动分析的通知，之后也通知我们有可疑现象，点击进去之后列出来了可能泄漏的引用链


![泄漏运行图](https://img-blog.csdnimg.cn/20190118111551274.png)


由图可以看出，处于引用链最底端的SecondActivity还在内存里，因为被UtilsManager对象的成员context所引用，而这个UtilsManager对象又被UtilsManager类的静态成员manager对象所持有
而造成无法回收




LeakCannary在Fragmen中使用
-----------------------
在上面的分析中我们只注入了一句   LeakCanary.install(this);，它并不能检测出哪个类比如Fragment是否泄漏
所以检测Fragment的方法是这样的，

这里需要用到一个RefWatcher类，通过他来观察对象泄漏的。其实LeakCanary.install(this);返回的就是这个对象
我们改写下LeakApplication的代码，把返回对象拿出来供外部调用
```
public class LeakApplication extends Application {
    private RefWatcher watcher;
    @Override
    public void onCreate() {
        super.onCreate();
        //分析后自身会有个进程，这里分析如果是那个泄漏分析进程的话则返回
        if(LeakCanary.isInAnalyzerProcess(this)){
           return;
        }
        watcher =  LeakCanary.install(this);
    }

    public RefWatcher getWatcher() {
        return watcher;
    }
}
```
我们编写个LeakFragment，在它的onDestory方法中把引用加入到RefWatcher中：

```
public class LeakFragment extends Fragment {

  @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        FragmentManager.getInstance(this);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if(getActivity().getApplication() instanceof LeakApplication){
            RefWatcher watcher = ((LeakApplication) getActivity().getApplication()).getWatcher();
            watcher.watch(this);
        }
    }
}
```

复制一份UtilsManger代码修改为FragmentManager：
```
public class FragmentManager {
    public Fragment fragment;
    public static FragmentManager manager;

    public FragmentManager(Fragment fragment) {
        this.fragment = fragment;
    }

    public static FragmentManager getInstance(Fragment fragment){
        if (manager == null) {
            synchronized (FragmentManager.class) {
                if(manager==null){
                    manager = new FragmentManager(fragment);
                }
            }
        }
        return manager;
    }
}
```

编写个LeakFragmentActivity：
```
public class LeakFragmentActivity  extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_fragment);
        getFragmentManager().beginTransaction().add(R.id.fl,new LeakFragment()).commit();
    }
}
```

看下分析效果：

![Fragment泄漏](https://img-blog.csdnimg.cn/20190118114936593.png)

可以看到指出了引用链最终指向LeakFragment导致LeakFragment无法被回收


LeakCannary检测Object泄漏
---------------------
其他的类也是同理，我们可以确认对象没有再被引用了后，再来一句watcher.watch(object);检测下这个对象是否真的被回收了


我们编写一个LeakObjectActivity 检测Object内存泄漏的：
```
public class LeakObjectActivity extends AppCompatActivity {

    public static class LeakObject{
        //2M
        byte[] buff = new byte[1024*1024*2];
    }
    public static class LeakObjectHolder{
        public static LeakObject object;

    }
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        LeakObject object = new LeakObject();
        //这里只是模拟不小心object被其他生命周期更长的对象所持有
        LeakObjectHolder.object = object;
        //检测下这个对象
        if(getApplication() instanceof LeakApplication){
            RefWatcher watcher = ((LeakApplication)getApplication()).getWatcher();
            watcher.watch(object);
        }
        //我们以为置空了，对象可以释放
        object = null;
    }

}
```


![在Object泄漏](https://img-blog.csdnimg.cn/20190118135252421.png)

如上，我们以为object=null了对象是释放了，但是不知道在其他地方又被持有了（当然这里LeakObjectHolder.object = object;这里写的这么明显，只是举个例子，方便测试下），导致对象不为空儿导致泄漏

LeakCannary的 release 版本
-----------------------
有的同学说，我们在代码里面写了那么多watcher.watch(this);，而程序在运行的时候分析这个也是要消耗性能的，有没有办法只需要在debug的时候才来，正式发布的时候就不去分析这些了，总不能删除掉关于LeakCannary的代码把。
我们引入区分调试和发布版的依赖包：
```
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
    //Release版并不会进行分析
    releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
```

哪些对象容易造成泄漏
----------

在上面的例子是举例了由于对象被另外一个静态对象所强引用而造成该对象无法释放。
而在实际开发时，有以下场景容易造成内存泄漏的

1、内部类：内部类本身持有外部类的引用，若内部类的生命周期比外部类长，则造成外部类无法释放。举个例子，假如在Activity中有一个内部Thread类在跑，由于Thread的run方法还没结束，而Thread对象本身持有Activity的引用，造成Activity无法释放

2、Handler：Activity中的Handler对象，跟内部类是同样的道理，如果没有声明称static，那么默认也持有Activity的引用，而往往Handler不一定在Activity销毁之前处理完消息

3、Timer：总容易被用来做一些循环的Task，如果没有对它进行stop的话，那么Task则一直持有

4、资源对象未没关闭 资源性对象比如（Cursor，File文件等）。


分析hprof 文件
-----------------------
我们项目比较复杂，并不一定单靠LeakCannary能看得那么清楚，这时候我们可以导出一份heap dump文件，通过内存泄漏分析工具来分析。

直接导出hprof文件：

![hprof文件](https://img-blog.csdnimg.cn/20190118140251933.png?)

导出来之后利用Android Studio自带的ProFiler工具查看堆情况，可以看到



![堆情况](https://img-blog.csdnimg.cn/20190118144607585.png)

可以发现堆中还存着一个LeakObject的实例，这个实例被一个所对象持有。想要让这个LeakObject回收掉，就必须干掉那个对象的持有


也可以使用MemoryAnalyzer工具，这是Eclipse中的一个插件，当然也可单独运行。要用这个工具分析hprof文件，前提是要先把android的hprof文件转为java的
输入命令如下：
```
hprof-conv android.hprof java.hprof
```

之后在MemoryAnalyzer打开后：

![MemoryAnalyzer](https://img-blog.csdnimg.cn/20190119143034638.png)


进入Leak Suspects中，为我们展示了一个饼图，指出目前内存使用情况。重要的是下面的Problem Suspect 指出了是LeakObjectActivity中的一个内部类LeakObject拥有一个实例不能回收，该实例占了2097184 bytes，该内存是由于一个类型为byte[]的对象所分配的

点击Deatils可以看到引用链：

![Leak Detail](https://img-blog.csdnimg.cn/20190119170938884.png)

可以发现是由于被LeakObjectHolder这个类对象锁所持有而无法释放
