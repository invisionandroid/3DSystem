#android客户端通过系统服务注册回调
##上一篇文章讲到如何新增一个系统服务，但是当系统服务运行起来后，有一些事件是要传递给客户端的
## 这篇文章讲述如何实现Service和Activity之间的跨进程调度 

###接着上一篇文章 继续添加
###1.新增一个aidl 接口文件，定义客户端的回调接口
>frameworks\base\core\java\com\invision\invisionbtcontrol\IBTServiceBinder.aidl
```

package com.invision.invisionbtcontrol;


interface IBTServiceBinder {

    void onKeyEventChanged(in int keycode,in int keyevent,in int deviceID);

}

```

###2.在IBTService.aidl 增加注册、注销的方法
>frameworks\base\core\java\com\invision\invisionbtcontrol\IBTService.aidl
```
import com.invision.invisionbtcontrol.IBTServiceBinder;
interface IBTService {
   	void serviceRegisterCallback(IBTServiceBinder cb);
	void serviceUnregisterCallback(IBTServiceBinder cb);
}
```
###3.aidl添加到Android.mk里
>frameworks\base\Android.mk
```
core/java/com/invision/invisionbtcontrol/IBTServiceBinder.aidl \
```

###3.通过RemoteCallbackList类 在IvBtService里实现注册注销操作
>\frameworks\base\core\java\com\invision\invisionbtcontrol\service\IvBtService.java
```
import com.invision.invisionbtcontrol.IBTServiceBinder;
    //新增RemoteCallbackList类
    private RemoteCallbackList<IBTServiceBinder> mCallbackList = new RemoteCallbackList<IBTServiceBinder>();

    //实现注册和注销
    @Override
    public void serviceRegisterCallback(IBTServiceBinder cb) throws RemoteException {
        mCallbackList.register(cb);
    }

    @Override
    public void serviceUnregisterCallback(IBTServiceBinder cb) throws RemoteException {
        mCallbackList.unregister(cb);
    }
```

``` 
新增方法 给注册过的客户端回调onKeyEventChanged方法
    public void doBoardCastonKeyEventChanged(int keycode, int keyevent, IvDevice.Index index) {
        synchronized (mCallbackList) {
            int N = mCallbackList.beginBroadcast();
            try {
                for (int i = 0; i < N; i++) {
                    mCallbackList.getBroadcastItem(i).onKeyEventChanged(keycode, keyevent,index.mIndex);
                }
            } catch (RemoteException e) {
            }
            mCallbackList.finishBroadcast();
        }
    }
```

### 4. 定义一个client端 用于连接服务以及注册服务监听
>\frameworks\base\core\java\com\invision\invisionbtcontrol\manager\IvControlManager.java
public class IvControlManager {
private IBTServiceBinder.Stub mServiceBinderImpl = new IBTServiceBinderImpl();

```
初始化绑定服务
public IvControlManager(Context m){
        mContext = m;
                mService = IBTService.Stub.asInterface(ServiceManager.getService(Context.INVISION_BT_CONTROL_SERVICE));
        try {
			mService.serviceRegisterCallback(mServiceBinderImpl);
		} catch (RemoteException e) {
		}
        
    }
}

    public void registerServiceDataCallback(IvControlServiceCallback mCb){
        mServiceClientCallback = mCb;
            try {
                mService.serviceRegisterCallback(mServiceBinderImpl);
            } catch (RemoteException e) {
            }

    }

    public void unregisterServiceDataCallback(){
        try {
            mService.serviceUnregisterCallback(mServiceBinderImpl);
        } catch (RemoteException e) {
        }
        mServiceClientCallback = null;
    }

实现service的回调
public class IBTServiceBinderImpl extends IBTServiceBinder.Stub{
        @Override
        public void onKeyEventChanged(int keycode, int keyevent,int deviceId) throws RemoteException {
           //do something
        }
}
```

###5.APP端如何调用
### 一般会将第4点的IvControlManager封装成一个jar包，主要提供app registerServiceDataCallback、unregisterServiceDataCallback等方法
### 在Activity端，引用jar包后，初始化 IvControlManager 并且实现注册监听，就能收到service的回调信息了