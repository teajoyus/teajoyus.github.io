&emsp;&emsp;AIDL，全称名为：Android Interface Definition Language。它是安卓中一种跨进程通信的实现方式，使得不同进程不同应用之间可以保持通信。
&emsp;&emsp;本篇内容为基础使用篇，下面将写一个例子，来实现不同应用进程之间的通信。
&emsp;&emsp;首先我们先来写服务端。编写Phone.java，来代表一个我们要传输的实体，代码如下：
```
public class Phone implements Parcelable{
    private int id;
    private String name;
    public Phone() {
    }
    public Phone(int id, String name) {
        this.id = id;
        this.name = name;
    }
    protected Phone(Parcel in) {
        id = in.readInt();
        name = in.readString();
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
    }
    @Override
    public int describeContents() {
        return 0;
    }
    public static final Creator<Phone> CREATOR = new Creator<Phone>() {
        @Override
        public Phone createFromParcel(Parcel in) {
            return new Phone(in);
        }
        @Override
        public Phone[] newArray(int size) {
            return new Phone[size];
        }
    };
    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return "Phone{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
```
&emsp;&emsp;由于我们是要跨进程传输，所以对于自定义类来说，类对象必须要被序列化后才能够进行传输。这里我们实现了Parcelable接口，需要实现的方法，在Android Studio上已经帮我们自动实现好了(真是人性化)。

&emsp;&emsp;然后我们在Android Studio工程上右键->new -> AIDL ->AIDL File来创建一个aidl文件，这里我们来编写几个AIDL接口。
创建的aidl文件命名为：IPhoneManager.aidl，代码如下：

```
// IBookManager.aidl
package com.example.aidl;
import com.example.aidl.Phone;
// Declare any non-default types here with import statements

interface IPhoneManager {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    List<Phone> getAllPhone();
    void addPhone(in Phone phone);
    void deletePhone(in int id);
}
```
&emsp;&emsp;可以看到我们写了个IPhoneManager 接口，接口有添加、删除、返回列表三个方法。需要注意的地方有两个：
&emsp;&emsp;第一是在AIDL文件中，并不是所有的数据类型都是可以使用的，可以使用的数据类型有如下：
1、基本类数据类型（int、long、char、boolean、double等等）;
2、String和CharSequence;
3、List：只支持ArrayList，里面每个元素都必须能够被AIDL支持;
4、Map:只支持HashMap。里面的每个元素都必须被AIDL之词，包括key和value；
5、Parceable:所有实现了Parceable接口的对象；
6、AIDL：所有的AIDL接口本身也可以在AIDL文件中使用。

&emsp;&emsp;以上对于AIDL所支持的数据类型的描述摘自任玉刚的《Android开发艺术探索》里面的第72页。笔者也是通过看他的这本书来学习和总结。一些知识在项目里面涉及过，但是又没有进行系统的总结。所以才有必要重新整理一下。

&emsp;&emsp;由于我们用到自定义的实体类，实现了Parceable接口，属于上面提到的第5点。可以被AIDL所支持。但是值得注意的是必须显示的导入包名进来，尽管我们定义的phone是在同一个包名下。

&emsp;&emsp;第二是不知道读者有没有注意到void addPhone(in Phone phone);这个方法的声明，参数前面加了个in，在AIDL文件的定义中，接口参数除了基本数据类型之外，我们定义的实现Parceable的类必须加上方向：in、out或者inout，来指明采纳数是输入型参数、输出型参数、输入输出型参数。

&emsp;&emsp;由于我们在IPhoneManager.aidl中用到了Phone这个类，所以我们也必须新建一个Phone.aidl文件，代码如下：
```
// phone.aidl
package com.example.aidl;
parcelable Phone;
```
&emsp;&emsp;代码只需要声明包名并且声明一下这个Phone类为parcelable，注意parcelable 为小写开头。

&emsp;&emsp;声明好了服务端的aidl文件之后，我们就可以来写一个服务端应用的service了，用于为远程客户端提供方法调用。

&emsp;&emsp;我们新建一个PhoneService.java，代码如下：
```
public class PhoneService extends Service {
    List<Phone> list = new CopyOnWriteArrayList<>();
    private  IBinder mBinder  =new IPhoneManager.Stub(){
       @Override
        public List<Phone> getAllPhone() throws RemoteException {
            Log.i("AIDL-demo", "service getAllPhone :request list:"+list);
            return list;
        }
        @Override
        public void addPhone(Phone phone) throws RemoteException {
            Log.i("AIDL-demo", "service addPhone :revice Phone:"+phone);
            list.add(phone);
        }
        @Override
        public void deletePhone(int id) throws RemoteException {
            Log.i("AIDL-demo", "service deletePhone :revice id:"+id);
        for(Phone phone:list){
            if(phone.getId()==id)list.remove(phone);
        }
        }
    };
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
    @Override
    public void onCreate() {
        super.onCreate();
        Log.i("AIDL-demo", "service:i am created... ");
        new Thread(){
            @Override
            public void run() {
                super.run();
                while(true) {
                    Log.i("AIDL-demo", "service:  still running... ");
                    try {
                        sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();
    }
}
```
&emsp;&emsp;在上面的代码结构中，我们主要是创建一个IBinder，然后在onBind方法中返回它。

&emsp;&emsp;IBinder的对象来自于我们继承的IPhoneManager内部抽象类Stub类，从而实现我们在aidl文件中声明的那几个接口方法。关于此处暂时不追踪Stub的由来，这将在下一个探讨AIDL的章节再进行讲述，这里我们只需要了解IBinder对象继承了这个抽象类从而实现接口方法，然后再把IBinder对象在onBind中返回即可。

&emsp;&emsp;在oncreate方法中，我们new出了一个死循环线程，旨在用来打印出它被创建和存活的“证据”。

&emsp;&emsp;在PhoneService中声明了一个CopyOnWriteArrayList，为啥不用ArrayList呢，原因是CopyOnWriteArrayList是支持并发读/写的，当我们的service被多个应用同时调用时，使用CopyOnWriteArrayList便起了一个同步的作用。当然，有时候我们并不是很想让其他应用来调用我们编写的aidl接口，特别是如果我们的aidl接口涉及到比较隐秘的信息的时候，更不应该被暴漏出来，此处我们可以在继承Stub类时重写onTransact方法：
```
 @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            return super.onTransact(code, data, reply, flags);
        }
```
&emsp;&emsp;当次方法返回false时，客户端便无权限调用远程方法，从而起到了一个权限验证的作用，比如我们可以在此方法里面通过调用：
```
getPackageManager().getPackagesForUid(getCallingPid())；
```
&emsp;&emsp;来验证客户端的包名，从而判断是不是我们允许调用的客户端包名。此处不是我们的重点，这将在后续在阐明。

&emsp;&emsp;最后，记得在服务端的注册清单文件中，注册我们刚才写的PhoneService

```
      <service android:name=".PhoneService"
            android:exported="true"
            android:process=":remote"
            >
            <intent-filter>
                <category android:name="android.intent.category.DEFAULT"></category>
                <action android:name="com.example.aidl.phone"></action>
            </intent-filter>

        </service>
```
&emsp;&emsp;由于我们需要远程调用，所以必须指定category和action。

&emsp;&emsp;好啦，服务端的编写就已经好了。接下来我们来看一下客户端的。

&emsp;&emsp;在服务端创建的AIDL文件中，
![目录结构](http://img.blog.csdn.net/20170724160215293?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;把我们编写的AIDL整个包名复制，拷贝到客户端中。这是因为，在AIDL远程调用中，必须保证这些接口和类的包名路径必须一致，才能够进行调用，否则将会运行时抛出异常如下：

![异常日志](http://img.blog.csdn.net/20170724160438916?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;当我们把服务端的aidl包名文件全部复制到客户端来后，在客户端中同样要在该包名下声明Phone.java（不声明你客户端怎么使用phone呢），或者从服务端拷贝Phone.java文件过来。

&emsp;&emsp;复制粘贴完成之后，客户端要做的就是去启动远程服务，并且调用远程方法从而实现对phone进行增加、删除、查找。
我们创建一个AIDLActivity.java，代码如下：

```
public class AIDLActivity extends AppCompatActivity implements View.OnClickListener {
    private static final String TAG = "AIDLActivity";
    private IPhoneManager manager;
    private int id;
    Button add,delete,all,bind;
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            manager = IPhoneManager.Stub.asInterface(iBinder);
            Log.i(TAG, "onServiceConnected: manager = "+manager);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {

        }
    };
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.aidl);
         initView();

    }

    private void initView() {
        add = (Button) findViewById(R.id.add);
        add.setOnClickListener(this);
        bind = (Button) findViewById(R.id.bind);
        bind.setOnClickListener(this);
        delete = (Button) findViewById(R.id.delete);
        delete.setOnClickListener(this);
        all = (Button) findViewById(R.id.all);
        all.setOnClickListener(this);
    }

    @Override
    protected void onDestroy() {
        unbindService(connection);
        super.onDestroy();
    }

    @Override
    public void onClick(View view) {
        switch (view.getId()){
            case R.id.bind:
                //java.lang.IllegalArgumentException: Service Intent must be explicit: Intent { act=com.example.aidl.phone }
                Intent intent= new Intent();
                intent.setAction("com.example.aidl.phone");
                intent.setPackage("com.example.aidl");
                bindService(intent,connection, Service.BIND_AUTO_CREATE);
                break;
            case R.id.add:
            try {
                manager.addPhone(new Phone(id++,"phone "+id+" plus"));
            } catch (RemoteException e) {
                e.printStackTrace();
            }
             break;
            case R.id.delete:
                try {
                    manager.deletePhone(id);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
            case R.id.all:
                try {
                    Toast.makeText(this,manager.getAllPhone().toString(),Toast.LENGTH_LONG).show();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
        }
    }
}
```
代码结构比较简单，所以没有加注释。

&emsp;&emsp;可以看到加载出了几个按钮的布局，并且创建了IPhoneMnager接口，通过这个接口去调用远程方法。
其中：

```
    private IPhoneManager manager;
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            manager = IPhoneManager.Stub.asInterface(iBinder);
            Log.i(TAG, "onServiceConnected: manager = "+manager);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {

        }
    };
```
&emsp;&emsp;我们创建一个ServiceConnection，也就是连接服务，在连接上服务后会调用onServiceConnected这个方法，该方法的参数iBinder正是我们在服务端中的onBind方法返回的ibinder.我们可以通过它来获得接口服务端的接口实现。也就是manager = IPhoneManager.Stub.asInterface(iBinder);
之后我们在通过调用manager.addPhone（）就可以远程调用服务端的addPhone方法。

&emsp;&emsp;值得注意的是，在启动与绑定服务端service的时候，我一开始是这么写：

```
    Intent intent= new Intent("com.example.aidl.phone");
    bindService(intent,connection, Service.BIND_AUTO_CREATE);
```
&emsp;&emsp;结果抛出了一个异常：

![ibinder异常](http://img.blog.csdn.net/20170724161528107?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;这是因为，android在5.0之后就不能够隐式启动一个Service了，必须显示的加上应用包名和action来调用：

```
     Intent intent= new Intent();
     intent.setAction("com.example.aidl.phone");
     intent.setPackage("com.example.aidl");
     bindService(intent,connection, Service.BIND_AUTO_CREATE);
```
&emsp;&emsp;好了，我们来运行一下客户端。

&emsp;&emsp;客户端启动截图如下：

![启动图](http://img.blog.csdn.net/20170724162046696?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;然后我们点击绑定Service后，查看log如下：

![启动service](http://img.blog.csdn.net/20170724162349085?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;可以看到我们已经启动了远程service，并且它已经在运行了，

&emsp;&emsp;接下来我们按下增加phone按钮和按下所有phone按钮，来看看log还有模拟器，log显示如下：

![通信](http://img.blog.csdn.net/20170724162519727?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;可以看到已经打印出了服务端的log，说明我们调用服务端的方法成功了，
看看模拟器：
![显示](http://img.blog.csdn.net/20170724162609704?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;模拟器的Toast显示出了服务端返回的phone列表，说明客户端对服务端进行addPhone成功了，然后对服务端调用manager.getAllPhone()也成功了。

&emsp;&emsp;到此，关于AIDL的基础使用已经实现了，然而，关于学会和理解AIDL的调用过程并不是那么简单的事，我将在后面的博客中继续对AIDL的知识点进行更深一步的阐述。
