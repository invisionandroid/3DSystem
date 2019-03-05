# android系统如何新建一个系统服务

##在android开发过程中，我们经常需要一个service给其他应用分发消息，实现跨进程调度。 所以我们需要开发一个系统层的service。以下以新增一个服务为例做个总结。

###1.新增一个服务的名字

>core/java/android/content/Context.java

```
	public static final String INVISION_BT_CONTROL_SERVICE = "invision_btcontrol_service";
```

###2.系统开机启动服务
>frameworks\base\services\java\com\android\server\SystemServer.java
```
import com.invision.invisionbtcontrol.service.IvBtService;
...

private void startOtherServices() {

IvBtService mBtControlService = null;
...

mBtControlService = new IvBtService(context);
ServiceManager.addService(Context.INVISION_BT_CONTROL_SERVICE, mBtControlService);


}
```
###3.新增aidl 以及 service实现类
>frameworks\base\core\java\com\invision\invisionbtcontrol\IBTService.aidl
```
package com.invision.invisionbtcontrol;

interface IBTService {
    void startService();
    void stopService();
}
```
>\frameworks\base\core\java\com\invision\invisionbtcontrol\service\IvBtService.java

```
public final class IvBtService extends IBTService.Stub {
    @Override
    public void startService() {

    }

    @Override
    public void stopService() {

    }

}    
```
###4.aidl添加到Android.mk里
>frameworks\base\Android.mk
```
core/java/com/invision/invisionbtcontrol/IBTService.aidl \
```

###5.make framework 
