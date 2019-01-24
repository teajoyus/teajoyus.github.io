---
tags: Android 
read: 1056   
---
&emsp;&emsp;在这之前，我们已经讲解了关于AIDL的基础使用，若不了解AIDL基础知识的读者请先点击阅读[《Android开发知识（三）Android进程间Binder通信机制的源码分析（上）》](http://blog.csdn.net/lc_miao/article/details/76022479)之后再回来阅读本文。


&emsp;&emsp;虽然Android系统是基于Linux内核，但是它的进程间通信方式并没有完全跟Linux一样，它拥有自己独特的通信方式--Binder。通过Binder我们可以进行不同应用与进程之间的相互通信以及远程方法调用。

&emsp;&emsp;不得不说，Binder是在Android中较为深入的东西，它的底层实现过程非常复杂，本文中提及的只是分析Binder的上层使用原理。

&emsp;&emsp;在上文中，我们提到了AIDL文件的编写，如下：
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
&emsp;&emsp;为什么我们编写完这个文件后就可以使用它跟java代码交互呢，实际上，在编写完成之后，Android Studio会帮我们自动生成对应的一个Binder类的java文件。具体查看该文件的方法如下：
&emsp;&emsp;把工程的目录切换成Projcet Files，然后依次点开app->build->generated/source->aidl/debug就可以看到编译环境给我们生成了一个ADIL包名下的java文件。(使用Eclipse的话，生成文件是放在了gen目录下面)
![查看Binder](http://img.blog.csdn.net/20170726175058024?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;这个生成文件的代码如下：

```
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: C:\\Users\\linmh\\Desktop\\AS\\Study\\app\\src\\main\\aidl\\com\\example\\aidl\\IPhoneManager.aidl
 */
package com.example.aidl;
// Declare any non-default types here with import statements
import android.os.RemoteCallbackList;
public interface IPhoneManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.aidl.IPhoneManager {
        private static final java.lang.String DESCRIPTOR = "com.example.aidl.IPhoneManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.aidl.IPhoneManager interface,
         * generating a proxy if needed.
         */
        public static com.example.aidl.IPhoneManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.aidl.IPhoneManager))) {
                return ((com.example.aidl.IPhoneManager) iin);
            }
            return new com.example.aidl.IPhoneManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getAllPhone: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.example.aidl.Phone> _result = this.getAllPhone();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addPhone: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.aidl.Phone _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.aidl.Phone.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addPhone(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_deletePhone: {
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    this.deletePhone(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.example.aidl.IPhoneManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public java.util.List<com.example.aidl.Phone> getAllPhone() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.aidl.Phone> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getAllPhone, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.aidl.Phone.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addPhone(com.example.aidl.Phone phone) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((phone != null)) {
                        _data.writeInt(1);
                        phone.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addPhone, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public void deletePhone(int id) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(id);
                    mRemote.transact(Stub.TRANSACTION_deletePhone, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_getAllPhone = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addPhone = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_deletePhone = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
    }

    public java.util.List<com.example.aidl.Phone> getAllPhone() throws android.os.RemoteException;

    public void addPhone(com.example.aidl.Phone phone) throws android.os.RemoteException;

    public void deletePhone(int id) throws android.os.RemoteException;
}

```
&emsp;&emsp;源码结构看起来貌似非常复杂，实际上有它一定的生成规则，并不难理解。接下来，我们来分析一下关于这个生成文件的结构：

首先，它生成的IPhoneManager是一个接口，并且实现了IInterface接口：

```
public interface IInterface
{
    /**
     * Retrieve the Binder object associated with this interface.
     * You must use this instead of a plain cast, so that proxy objects
     * can return the correct result.
     */
    public IBinder asBinder();
}
```
&emsp;&emsp;android.os.IInterface接口只有一个asBinder()方法，用于返回一个binder对象。事实上，**所有的AIDL接口都需要继承IInterface接口**。
**重点内容**
&emsp;&emsp;其次，我们在AIDL文件中声明的接口方法都在这个文件中生成了，这个就不必多说了：

```
    public java.util.List<com.example.aidl.Phone> getAllPhone() throws android.os.RemoteException;

    public void addPhone(com.example.aidl.Phone phone) throws android.os.RemoteException;

    public void deletePhone(int id) throws android.os.RemoteException;
```
&emsp;&emsp;而我们看的，其中看起来非常复杂的，是接口里面定义的一个抽象类Stub，它继承了android.os.Binder ，并且实现了IPhoneManager接口中的方法。
我们来一步一步分析这个抽象类：

 - **DESCRIPTOR** 
 
```
private static final java.lang.String DESCRIPTOR = "com.example.aidl.IPhoneManager";
```
&emsp;&emsp;DESCRIPTOR 代表了我们AIDL接口的完整类名，它被作为Binder的唯一标识。当然，我们也可以自定义其他标识，只是一般用当前的AIDL完整类名来表示，可以启到一个唯一标识的作用。
 
 - **Stub()**
 
 该构造方法调用了attachInterface方法，该方法的源码如下：
 
```
     
    /**
     * Convenience method for associating a specific interface with the Binder.
     * After calling, queryLocalInterface() will be implemented for you
     * to return the given owner IInterface when the corresponding
     * descriptor is requested.
     */
    public void attachInterface(IInterface owner, String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }
```
&emsp;&emsp;可以发现只是把当前对象和标识符做了赋值。

 - **asBinder()**
 
	返回当前的Binder对象

 - **asInterface()**
  
```
/**
         * Cast an IBinder object into an com.example.aidl.IPhoneManager interface,
         * generating a proxy if needed.
         */
        public static com.example.aidl.IPhoneManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.aidl.IPhoneManager))) {
                return ((com.example.aidl.IPhoneManager) iin);
            }
            return new com.example.aidl.IPhoneManager.Stub.Proxy(obj);
        }
```
&emsp;&emsp;从代码得知，该方法是把一个Binder对象转换成对应的AIDL接口。其中先执行android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR); 也就是根据我们前面定义的标识来查找AIDL接口对象，这种方式是在同个应用进程下的时候，则返回服务端的AIDL对象本身，我们可以查看queryLocalInterface的源码：
```
  /**
     * Use information supplied to attachInterface() to return the
     * associated IInterface if it matches the requested
     * descriptor.
     */
    public IInterface queryLocalInterface(String descriptor) {
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }
```
&emsp;&emsp;源码很简单，就是判断一下标识符是不是等同于mDescriptor，是的话则返回mOwner，到这里可能有读者会问了，前面不是在构造方法里面已经赋值了mDescriptor和mOwner吗，那这里不是会等同于么？
&emsp;&emsp;其实未必，这也是接下来要说的跨进程通信和同进程通信的差异，我们知道，不同进程之间，内存会共享失败，也就是说，不同进程之间各自维护自己的对象，并不能直接交互。（这里感兴趣的读者可以自行尝试新建一个工程，然后新建两个Activity，通过配置android:process来让两个Activity运行在不同的进程上，编写一个静态变量，在第一个Activity上修改该变量，然后再启动第二个Activity并打印出该变量，会发现打印出的静态变量并不是修改后的值。）

&emsp;&emsp;所以说，如果是不同进程，那么Binder便不像在同个进程中能使用queryLocalInterface来得到AIDL接口对象，那么怎么做到呢，我们看一下asInterface方法中接下来的源码：
```
return new com.example.aidl.IPhoneManager.Stub.Proxy(obj);
```
&emsp;&emsp;发现它创建并返回了一个Stub类中的内部代理类Proxy，其实，正因为不同进程不能共享binder，所以此处使用了一个代理类来给我们把远程的binder对象转换成AIDL接口，这里暂先不解释Proxy，放到后面再说。

 - **onTransact()**
 
 该方法运行在服务端中的Binder线程池中，当客户端绑定Service并进行远程方法调用时系统会调用此方法，其中的几个参数作用如下：
 
 code：服务端通过code来确定客户端请求调用的是那个方法；
 data：存放目标方法所需要的参数；
 reply：调用完目标方法后向reply写入返回值

&emsp;&emsp;该方法的代码流程是根据这个code来确定调用的方法，如果有输入型参数的话就往data里面取出输入参数，然后再执行对应的方法，如果有返回值的话再写入到reply中。

&emsp;&emsp;值得注意的是，该方法有个boolean返回值，如返回false则客户端会请求失败，因此我们可以利用这个特点来进行客户端的权限验证，在这里我们先不做讨论，放到后面再说

&emsp;&emsp;到此，我们差不多就分析完了Stub类，正因为它继承了Binder又实现了AIDL接口，所以才能够建立Binder与AIDL接口的通信，从而实现进程间通信。

&emsp;&emsp;刚才我们提到，当不同进程时，asInterface返回的是一个Proxy代理类对象，那么这个类的实现是怎么样的呢，接下来我们就来分析下这个Stub的内部代理类的源码结构。

```
private static class Proxy implements com.example.aidl.IPhoneManager 
```
看上去这个代理类只是实现了我们的AIDL接口，并且他是私有的。

```
     private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }
```
类中只有一个对象，就是IBinder对象，也就是一个被代理的对象（此处设计模式中的一个代理模式，不了解的读者也可自行查阅相关资料）
而Proxy类的构造方法是传入了IBinder对象，这就有区别于前面我们提到的内部抽象类Stub的构造方法，它的是调用：
```
this.attachInterface(this, DESCRIPTOR);
```
这个方法，才使得queryLocalInterface方法返回了自身对象：
```
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
```
&emsp;&emsp;而代理类对传入的这个远程的Ibinder对象并没有调用这个方法，实际上通过打印出这个远程的Ibinder对象的可以看出来他是一个android.os.BinderProxy类，这是客户端的binder代理类，对服务端的binder对象进行代理，由jni实现。在Android的C代码层中是通过android_util_Binder.cpp其中的static jboolean android_os_BinderProxy_transact函数，来实现连接到服务端的Binder对象。
&emsp;&emsp;这里太深入我们便不再做分析执行。只需要知道在执行queryLocalInterface方法后返回了null，然后才走了代理类，为我们做了远程binder对象转换为AIDL接口的代理。

```
   @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }
```
&emsp;&emsp;接下来的代理类这两个方法就不用多说了把，asBinder是我们实现AIDL接口必须实现的接口方法，它用于把AIDL接口转换成Binder对象（与asInterface方法反过来）。

&emsp;&emsp;最下面，代理类实现了我们定义的几个AIDL方法，实际上他跟Stub类对于这几个方法的过程并没有太大区别，它的方法过程为：

```
  android.os.Parcel _data = android.os.Parcel.obtain();
  android.os.Parcel _reply = android.os.Parcel.obtain();
```
&emsp;&emsp;先拿到一个我们基于序列化的传输载体，_data是写入了方法的输入参数，而 _reply是写入了方法的返回值。

&emsp;&emsp;如果该方法有输入参数，比如我们的addPhone方法如下：
```
 _data.writeInterfaceToken(DESCRIPTOR);
   if ((phone != null)) {
       _data.writeInt(1);
       phone.writeToParcel(_data, 0);
   } else {
       _data.writeInt(0);
   }
```
&emsp;&emsp;需要传入一个Phone对象，那么就会先把输入参数写入到_data中，并且写入个1作为有参数的标记，否则写入0代表没参数，它调用的phone.writeToParcel(_data, 0);正是因为我们在定义Phone类的时候实现了Pracelable序列号接口。

&emsp;&emsp;如果是无输入参数的方法，比如getAllPhone方法，那么就没有这部分代码环节，
然后调用：

```
mRemote.transact(Stub.TRANSACTION_addPhone, _data, _reply, 0);
```
&emsp;&emsp;进行远程方法调用，实际上我们也看出来只有服务端和客户端是不同进程的时候，才用到了代理类并且走了这个transact过程，而同个进程并不会走transact过程。我们查看transact的源码：

```
   /**
     * Default implementation rewinds the parcels and calls onTransact.  On
     * the remote side, transact calls into the binder to do the IPC.
     */
    public final boolean transact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (false) Log.v("Binder", "Transact: " + code + " to " + this);
        if (data != null) {
            data.setDataPosition(0);
        }
        boolean r = onTransact(code, data, reply, flags);
        if (reply != null) {
            reply.setDataPosition(0);
        }
        return r;
    }
```
&emsp;&emsp;发现实际上走的是boolean r = onTransact(code, data, reply, flags);
可以看到最终也是走了onTransact方法。

&emsp;&emsp;到此，我们关于AIDL的原理分析就已经差不多了，讲的是有一点绕。
我们再来屡一下，总结一下这个过程的原理为如下几个步骤：
1、首先由客户端通过绑定服务端；
2、服务端被创建启动后，生成一个继承Stub类的对象，也就是binder，并且在onBind中返回；
3、客户端监听到连接成功，调用了Stub类的asInterface方法来通过返回的Binder对象生成一个AIDL接口对象；
	如果是同个进程，那么在连接成功后返回的是服务端的Binder对象;
	如果是不同进程，那么返回的是BinderProxy对象，并且由Stub类的内部代理类来代理BinderProxy转换成AIDL接口。
	所以，如果是同个进程，那么对于客户端来说它的AIDL接口对象就是直接来源于服务端对象(相当于客户端的接口由服务端实现)。
	如果是不同进程，那么对于客户端来说，它的AIDL接口对象就是Stub类的内部Proxy类对象，这个Proxy类对象代理了BinderProxy对象，而BinderProxy对象代理了服务端的Binder对象。
4、客户端发起远程调用请求，服务端会执行onTransact方法，也是通过这个方法来确定客户端要调用的目标方法，onTransact方法返回false时则客户端会请求失败。

&emsp;&emsp;最后，不得不说一点，关于我们编写的aidl文件，其实并非必要。只是因为定义这个AIDL接口过程是有规律性的，又比较麻烦。所以编译环境为我们做出了这一步，它把我们编写的aidl文件转换成了java文件，实际上运行的时候aidl文件并不起作用，它只是在编译期自动生成文件，如果aidl文件格式写错了，那么编译器将会在生成java文件的时候报错。

&emsp;&emsp;所以，我们完全可以自己写一个AIDL接口，而不写aidl文件。事实上，倘若我们需要在AIDL接口上加一些打印或者做出一些修改，也可以通过aidl生成文件后放到源码对应的包目录，然后再进行修改。

以上，我们就已经分析完成了AIDL的基本原理过程，相信读者对于AIDL的原理已经有了一定的了解，后面的博客将再续写**关于AIDL的进阶使用**，希望本系列博客在总结知识的同时，也能帮读者解除在项目中碰到的疑惑。
