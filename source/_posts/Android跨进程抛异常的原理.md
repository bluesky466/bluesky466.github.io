title: Android跨进程抛异常的原理
date: 2018-11-10 14:21:42
tags:
    - 技术相关
    - Android
---

今天接到了个需求,需要用到跨进程抛异常。

# 怎样将异常从服务端抛到客户端

也就是说在Service端抛出的异常需要可以在Client端接收。印象中binder是可以传异常的,所以aidl直接走起:

```
// aidl文件
interface ITestExceptionAidl {
    boolean testThrowException();
}

// service端实现
public class AidlService extends Service {
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return new ITestExceptionAidl.Stub() {

            @Override
            public boolean testThrowException() throws RemoteException {
                if (true) {
                    throw new RuntimeException("TestException");
                }
                return true;
            }
        };
    }
}

// client端实现
bindService(intent, new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        ITestExceptionAidl aidl = ITestExceptionAidl.Stub.asInterface(service);

        try {
            aidl.testThrowException();
        } catch (Exception e) {
            Log.e("testtest", "Exception", e);
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
}, Context.BIND_AUTO_CREATE);
```

但是这个程序实际上运行起来是这样的:

```
01-01 05:31:55.475  4868  4880 E JavaBinder: *** Uncaught remote exception!  (Exceptions are not yet supported across processes.)
01-01 05:31:55.475  4868  4880 E JavaBinder: java.lang.RuntimeException: TestException
01-01 05:31:55.475  4868  4880 E JavaBinder:    at me.linjw.demo.ipcdemo.AidlService$1.testThrowException(AidlService.java:22)
01-01 05:31:55.475  4868  4880 E JavaBinder:    at me.linjw.demo.ipcdemo.ITestExceptionAidl$Stub.onTransact(ITestExceptionAidl.java:48)
01-01 05:31:55.475  4868  4880 E JavaBinder:    at android.os.Binder.execTransact(Binder.java:565)
```

看日志里面的ITestExceptionAidl$Stub.onTransact,也就是说在service端就已经被异常打断了,并没有传给client端,而且第一个大大的"Exceptions are not yet supported across processes."是说异常不允许跨进程吗?但是我明明记得AIDL生成的代码里面就有向Parcel写入异常啊:

```
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
    switch (code) {
        case INTERFACE_TRANSACTION: {
            reply.writeString(DESCRIPTOR);
            return true;
        }
        case TRANSACTION_testThrowException: {
            data.enforceInterface(DESCRIPTOR);
            boolean _result = this.testThrowException();
            reply.writeNoException(); // 这里写入的是没有抛出异常
            reply.writeInt(((_result) ? (1) : (0)));
            return true;
        }
    }
    return super.onTransact(code, data, reply, flags);
}
```

查找Parcel的源码,其实是有writeException方法的:


```
public final void writeException(Exception e) {
    int code = 0;
    if (e instanceof Parcelable
            && (e.getClass().getClassLoader() == Parcelable.class.getClassLoader())) {
        // We only send Parcelable exceptions that are in the
        // BootClassLoader to ensure that the receiver can unpack them
        code = EX_PARCELABLE;
    } else if (e instanceof SecurityException) {
        code = EX_SECURITY;
    } else if (e instanceof BadParcelableException) {
        code = EX_BAD_PARCELABLE;
    } else if (e instanceof IllegalArgumentException) {
        code = EX_ILLEGAL_ARGUMENT;
    } else if (e instanceof NullPointerException) {
        code = EX_NULL_POINTER;
    } else if (e instanceof IllegalStateException) {
        code = EX_ILLEGAL_STATE;
    } else if (e instanceof NetworkOnMainThreadException) {
        code = EX_NETWORK_MAIN_THREAD;
    } else if (e instanceof UnsupportedOperationException) {
        code = EX_UNSUPPORTED_OPERATION;
    } else if (e instanceof ServiceSpecificException) {
        code = EX_SERVICE_SPECIFIC;
    }
    writeInt(code);
    StrictMode.clearGatheredViolations();
    if (code == 0) {
        if (e instanceof RuntimeException) {
            throw (RuntimeException) e;
        }
        throw new RuntimeException(e);
    }
    writeString(e.getMessage());
    ...
}
```

可以看到其实Parcel是支持写入异常的,但是只支持Parcelable的异常或者下面这几种异常:

- SecurityException
- BadParcelableException
- IllegalArgumentException
- NullPointerException
- IllegalStateException
- NetworkOnMainThreadException
- UnsupportedOperationException
- ServiceSpecificException

如果是普通的RuntimeException,这打断写入,继续抛出。

于是我们将RuntimeException改成它支持的UnsupportedOperationException试试:

```
// service端改成抛出UnsupportedOperationException
ppublic class AidlService extends Service {
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return new ITestExceptionAidl.Stub() {

            @Override
            public boolean testThrowException() throws RemoteException {
                if (true) {
                    throw new UnsupportedOperationException("TestException");
                }
                return true;
            }
        };
    }
}

// client端实现还是一样,不变
bindService(intent, new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        ITestExceptionAidl aidl = ITestExceptionAidl.Stub.asInterface(service);

        try {
            aidl.testThrowException();
        } catch (Exception e) {
            Log.e("testtest", "Exception", e);
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
}, Context.BIND_AUTO_CREATE);
```

这样运行的话客户端就能捕获到异常:

```
01-01 05:49:46.770 19937 19937 E testtest: RemoteException
01-01 05:49:46.770 19937 19937 E testtest: java.lang.UnsupportedOperationException: TestException
01-01 05:49:46.770 19937 19937 E testtest:      at android.os.Parcel.readException(Parcel.java:1728)
01-01 05:49:46.770 19937 19937 E testtest:      at android.os.Parcel.readException(Parcel.java:1669)
01-01 05:49:46.770 19937 19937 E testtest:      at me.linjw.demo.ipcdemo.ITestExceptionAidl$Stub$Proxy.testThrowException(ITestExceptionAidl.java:77)
01-01 05:49:46.770 19937 19937 E testtest:      at me.linjw.demo.ipcdemo.MainActivity$3.onServiceConnected(MainActivity.java:132)
01-01 05:49:46.770 19937 19937 E testtest:      at android.app.LoadedApk$ServiceDispatcher.doConnected(LoadedApk.java:1465)
01-01 05:49:46.770 19937 19937 E testtest:      at android.app.LoadedApk$ServiceDispatcher$RunConnection.run(LoadedApk.java:1482)
01-01 05:49:46.770 19937 19937 E testtest:      at android.os.Handler.handleCallback(Handler.java:751)
01-01 05:49:46.770 19937 19937 E testtest:      at android.os.Handler.dispatchMessage(Handler.java:95)
01-01 05:49:46.770 19937 19937 E testtest:      at android.os.Looper.loop(Looper.java:154)
01-01 05:49:46.770 19937 19937 E testtest:      at android.app.ActivityThread.main(ActivityThread.java:6097)
01-01 05:49:46.770 19937 19937 E testtest:      at java.lang.reflect.Method.invoke(Native Method)
01-01 05:49:46.770 19937 19937 E testtest:      at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:1052)
01-01 05:49:46.770 19937 19937 E testtest:      at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:942)
```

# 跨进程传递异常的原理

好,知道了如何去跨进程传递异常之后,然后我们来看看异常到底是如何传递过去的。

让我们再来看看异常写入的代码:

```
// 有异常的情况
public final void writeException(Exception e) {
    int code = 0;
    if (e instanceof Parcelable
            && (e.getClass().getClassLoader() == Parcelable.class.getClassLoader())) {
        // We only send Parcelable exceptions that are in the
        // BootClassLoader to ensure that the receiver can unpack them
        code = EX_PARCELABLE;
    } else if (e instanceof SecurityException) {
        code = EX_SECURITY;
    } else if (e instanceof BadParcelableException) {
        code = EX_BAD_PARCELABLE;
    } else if (e instanceof IllegalArgumentException) {
        code = EX_ILLEGAL_ARGUMENT;
    } else if (e instanceof NullPointerException) {
        code = EX_NULL_POINTER;
    } else if (e instanceof IllegalStateException) {
        code = EX_ILLEGAL_STATE;
    } else if (e instanceof NetworkOnMainThreadException) {
        code = EX_NETWORK_MAIN_THREAD;
    } else if (e instanceof UnsupportedOperationException) {
        code = EX_UNSUPPORTED_OPERATION;
    } else if (e instanceof ServiceSpecificException) {
        code = EX_SERVICE_SPECIFIC;
    }
    writeInt(code);
    StrictMode.clearGatheredViolations();
    if (code == 0) {
        if (e instanceof RuntimeException) {
            throw (RuntimeException) e;
        }
        throw new RuntimeException(e);
    }
    writeString(e.getMessage());
    
    // 之后还有一些写入堆栈的操作,比较多,这里可以不看
}

public final void writeNoException() {
    if (StrictMode.hasGatheredViolations()) {
    	
		// 如果StrictMode收集到了写违规行为会走这里,我们可以不关注它
        writeInt(EX_HAS_REPLY_HEADER);
        ...
    } else {
    	// 一般情况下会走这里
        writeInt(0);
    }
}
```

这里给每种支持的异常都编了个号码,它会往Parcel写入。而0代表的是没有发生异常。然后再看看读取异常的代码:

```
public boolean testThrowException() throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    boolean _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        mRemote.transact(Stub.TRANSACTION_testThrowException, _data, _reply, 0);
        _reply.readException();
        _result = (0 != _reply.readInt());
    } finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}


// android.os.Parcel.readException
public final void readException() {
    int code = readExceptionCode();
    if (code != 0) {
        String msg = readString();
        
        //在这个方法里面创建异常并且抛出
        readException(code, msg);
    }
}
```

然后这里有个需要注意的点就是异常必须是写在Parcel的头部的,也就是说如果没有异常,我们先要将0写到头部,然后再将返回值继续往后面写入。如果有异常,我们要先将异常编码写入头部,然后就不需要再写入返回值了。

这样,在客户端读取的时候读取的头部就能知道到底有没有异常,没有异常就继续读取返回值,有异常就将异常读取出来并且抛出。


```
// service端代码
boolean _result = this.testThrowException();
reply.writeNoException(); // 先写入异常
reply.writeInt(((_result) ? (1) : (0))); // 再写入返回值


// client端代码
mRemote.transact(Stub.TRANSACTION_testThrowException, _data, _reply, 0);
_reply.readException(); // 先读取异常,有异常的话readException方法里面会直接抛出
_result = (0 != _reply.readInt()); // 再读取返回值
```

也就是Parcel的头部是一个标志位,标志了有异常或者无异常:


{% img /Android跨进程抛异常的原理/1.png %}

但是我们看到AIDL生成的代码都是写入的无异常,那我们抛出的异常是怎么传过去的呢?还记得这个打印吗?

```
01-01 05:31:55.475  4868  4880 E JavaBinder: *** Uncaught remote exception!  (Exceptions are not yet supported across processes.)
01-01 05:31:55.475  4868  4880 E JavaBinder: java.lang.RuntimeException: TestException
01-01 05:31:55.475  4868  4880 E JavaBinder:    at me.linjw.demo.ipcdemo.AidlService$1.testThrowException(AidlService.java:22)
01-01 05:31:55.475  4868  4880 E JavaBinder:    at me.linjw.demo.ipcdemo.ITestExceptionAidl$Stub.onTransact(ITestExceptionAidl.java:48)
01-01 05:31:55.475  4868  4880 E JavaBinder:    at android.os.Binder.execTransact(Binder.java:565)
```

我们去android.os.Binder.execTransact这里找找看, onTransact方法实际就是在这里被调用的

```
private boolean execTransact(int code, long dataObj, long replyObj, int flags) {
	Parcel data = Parcel.obtain(dataObj);
	Parcel reply = Parcel.obtain(replyObj);
	boolean res;
	
	try {
	    res = onTransact(code, data, reply, flags);
	} catch (RemoteException|RuntimeException e) {
	    ...
	    reply.setDataPosition(0);
	    reply.writeException(e);
	    res = true;
	} catch (OutOfMemoryError e) {
	    RuntimeException re = new RuntimeException("Out of memory", e);
	    reply.setDataPosition(0);
	    reply.writeException(re);
	    res = true;
	}
	checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
	reply.recycle();
	data.recycle();
	
	return res;
}
```

看,这里如果catch到了方法,也就是说我们服务端有抛出异常,就会在catch代码块里面先就Parcel的游标重置回0,然后往Parcel头部写入异常。

好,到了这里其实整个流程就差不多了,但是我发现我没有看到那个"Exceptions are not yet supported across processes."字符串,这个不支持的提示又是哪里来的呢?

让我们再回忆下代码,在遇到不支持的异常类型的时候, writeException也会抛出异常:

```
public final void writeException(Exception e) {
    int code = 0;
    if (e instanceof Parcelable
            && (e.getClass().getClassLoader() == Parcelable.class.getClassLoader())) {
        // We only send Parcelable exceptions that are in the
        // BootClassLoader to ensure that the receiver can unpack them
        code = EX_PARCELABLE;
    } else if (e instanceof SecurityException) {
        code = EX_SECURITY;
    } else if (e instanceof BadParcelableException) {
        code = EX_BAD_PARCELABLE;
    } else if (e instanceof IllegalArgumentException) {
        code = EX_ILLEGAL_ARGUMENT;
    } else if (e instanceof NullPointerException) {
        code = EX_NULL_POINTER;
    } else if (e instanceof IllegalStateException) {
        code = EX_ILLEGAL_STATE;
    } else if (e instanceof NetworkOnMainThreadException) {
        code = EX_NETWORK_MAIN_THREAD;
    } else if (e instanceof UnsupportedOperationException) {
        code = EX_UNSUPPORTED_OPERATION;
    } else if (e instanceof ServiceSpecificException) {
        code = EX_SERVICE_SPECIFIC;
    }
    writeInt(code);
    StrictMode.clearGatheredViolations();
    
    // code为0,代表不支持这种异常,继续把异常抛出或者创建RuntimeException抛出
    if (code == 0) {
        if (e instanceof RuntimeException) {
            throw (RuntimeException) e;
        }
        throw new RuntimeException(e);
    }
    ...
}
```

由于这个writeException,已经是在catch代码块里面运行的了,没有人再去catch它,于是就会打断这个流程,直接跳出。形成了一个Uncaught remote exception。

最后我们找到/frameworks/base/core/jni/android\_util\_Binder.cpp的onTransact方法,这里通过jni调到Java的execTransact方法,调用完之后进行ExceptionCheck,如果发现有异常的话就report_exception:

```
virtual status_t onTransact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) {
    JNIEnv* env = javavm_to_jnienv(mVM);

    IPCThreadState* thread_state = IPCThreadState::self();
    const int32_t strict_policy_before = thread_state->getStrictModePolicy();
    
    jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
        code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);

    if (env->ExceptionCheck()) {
        jthrowable excep = env->ExceptionOccurred();

        // 就是这里啦
        report_exception(env, excep,
            "*** Uncaught remote exception!  "
            "(Exceptions are not yet supported across processes.)");
        res = JNI_FALSE;

        env->DeleteLocalRef(excep);
    }
    ...
}
```

