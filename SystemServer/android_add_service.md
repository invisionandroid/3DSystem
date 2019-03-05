# android系统如何新建一个系统服务

##在android开发过程中，我们经常需要一个service给其他应用分发消息，实现跨进程调度。 所以我们需要开发一个系统层的service。以下以新增一个服务为例做个总结。

###1.新增一个服务的名字

>core/java/android/content/Context.java

```
	public static final String INVISION_GESTURE_SERVICE = "invision_gesture_service";
```

###2.系统开机启动服务
>frameworks\base\services\java\com\android\server\SystemServer.java
```
import com.invision.invisiongesture.InVisionGestureService;
...

private void startOtherServices() {

InVisionGestureService mGestureService = null;
...

mGestureService = new InVisionGestureService(context);
mGestureService.startService();
ServiceManager.addService(Context.INVISION_GESTURE_SERVICE, mGestureService);


}
```
###3.新增service实现类
>frameworks\base\core\java\com\invision\IInVisionGestureService.aidl
```
package com.invision.invisiongesture;
import com.invision.invisiongesture.IServiceBinder;
interface IInVisionGestureService {
    void startService();
    void stopService();
}
```
>frameworks\base\core\java\com\invision\InVisionGestureService.java

```
public final class InVisionGestureService extends IInVisionGestureService.Stub{
    @Override
    public void startService() {

    }

    @Override
    public void stopService() {

    }

}    
```
###4.aidl添加到Android.mk里
>core/java/com/invision/invisiongesture/IInVisionGestureService.aidl \
```
core/java/com/invision/invisiongesture/IInVisionGestureService.aidl \
```

###5.make framework 