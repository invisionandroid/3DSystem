# 实现SurfaceFlinger的主动刷新

## Revision

Date       | Author               | Notes
---------- | -------------------  | -------------------------
2019-3-3   | xieweikol@gmail.com  | Initialize the document



## Abstract

SurfaceFlinger在收到来自硬件层的Vsync事件之后，不是每次都会调用`SurfaceFlinger::handleMessageInvalidate()`的。而SurfaceFlinger中处理Vsync事件涉及到的类有：`DispSyncThread`, `DispSync`, `SurfaceFlinger::DispSyncSource`, `EventThread`, `EventThread::Connection`, `MessageQueue`.

本文首先研究这些类之间的关系，以及自Vysnc事件发生后这些类的处理流程。最终，通过修改EventThread::Connection中的成员count实现了每个Vsync事件都会调用SurfaceFlinger::handleMessageInvalidate()的功能。

## Process

### 从SurfaceFlinger.cpp的init()开始

在SurfaceFlinger.cpp的`init()`方法中，通过如下的代码初始化了DispSyncSource和EventThread，并将mSFEventThread关联到成员变量`MessageQueue mEventQueue`中:

```c++{.line-numbers}
sp<VSyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync,
      vsyncPhaseOffsetNs, true, "app");
mEventThread = new EventThread(vsyncSrc, *this, false);
sp<VSyncSource> sfVsyncSrc = new DispSyncSource(&mPrimaryDispSync,
      sfVsyncPhaseOffsetNs, true, "sf");
mSFEventThread = new EventThread(sfVsyncSrc, *this, true);
mEventQueue.setEventThread(mSFEventThread);
```

在EventThread.cpp中可以看到当EventThread的实例创建时，在其onFirstRef()中会开启线程:

```c++{.line-numbers}
void EventThread::onFirstRef() {
    run("EventThread", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);
}
```

而在`MessageQueue::setEventThread()`中，会创建`EventThread::Connection`:

```c++{.line-numbers}
void MessageQueue::setEventThread(const sp<EventThread>& eventThread)
{
   if (mEventThread == eventThread) {
      return;
   }

   if (mEventTube.getFd() >= 0) {
      mLooper->removeFd(mEventTube.getFd());
   }

   mEventThread = eventThread;
   mEvents = eventThread->createEventConnection();
   mEvents->stealReceiveChannel(&mEventTube);
   mLooper->addFd(mEventTube.getFd(), 0, Looper::EVENT_INPUT,
         MessageQueue::cb_eventReceiver, this);
}
```

```c++{.line-numbers}
sp<EventThread::Connection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}

EventThread::Connection::Connection(
      const sp<EventThread>& eventThread)
   : count(-1), mEventThread(eventThread), mChannel(gui::BitTube::DefaultSize) {}
```

注意 __count的初始值是-1__。
在`EventThread::Connection`的`onFirstConnection`中，会将`Connection`的实例传给`EventThread`，并加到其成员变量`mDisplayEventConnections`:

```c++{.line-numbers}
void EventThread::Connection::onFirstRef() {
   mEventThread->registerDisplayEventConnection(this);
}

status_t EventThread::registerDisplayEventConnection(
      const sp<EventThread::Connection>& connection) {
   Mutex::Autolock _l(mLock);
   mDisplayEventConnections.add(connection);

   if (!mFirstConnectionInited) {
      // additional external displays can't be enumerated by default,
      // need to report additional hotplug to listeners
      for (int32_t type = DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES;
            type < (int32_t)mHotplugEvent.size(); type++) {
         if (mHotplugEvent[type].hotplug.connected) {
               connection->postEvent(mHotplugEvent[type]);
         }
      }
      mFirstConnectionInited = true;
   }

   mCondition.broadcast();
   return NO_ERROR;
}
```

### 第一次执行EventThread::waitForEvent()

当`EventThread::threadLoop()`运行起来后，首先会执行`EventThread::waitForEvent()`方法：

```c++{.line-numbers}
Vector< sp<EventThread::Connection> > EventThread::waitForEvent(
      DisplayEventReceiver::Event* event)
{
   Mutex::Autolock _l(mLock);
   Vector< sp<EventThread::Connection> > signalConnections;

   do {
      bool eventPending = false;
      bool waitForVSync = false;

      size_t vsyncCount = 0;
      nsecs_t timestamp = 0;
      for (int32_t i=0 ; i<(int32_t)mVSyncEvent.size(); i++) {
         timestamp = mVSyncEvent[i].header.timestamp;
         if (timestamp) {
               // we have a vsync event to dispatch
               if (mInterceptVSyncs) {
                  mFlinger.mInterceptor.saveVSyncEvent(timestamp);
               }
               *event = mVSyncEvent[i];
               mVSyncEvent[i].header.timestamp = 0;
               vsyncCount = mVSyncEvent[i].vsync.count;
               break;
         }
      }

      if (!timestamp) {
         // no vsync event, see if there are some other event
         eventPending = !mPendingEvents.isEmpty();
         if (eventPending) {
               // we have some other event to dispatch
               *event = mPendingEvents[0];
               mPendingEvents.removeAt(0);
         }
      }

      // find out connections waiting for events
      size_t count = mDisplayEventConnections.size();
      for (size_t i=0 ; i<count ; i++) {
         sp<Connection> connection(mDisplayEventConnections[i].promote());
         if (connection != NULL) {
               bool added = false;
               if (connection->count >= 0) {
                  // we need vsync events because at least
                  // one connection is waiting for it
                  waitForVSync = true;
                  if (timestamp) {
                     // we consume the event only if it's time
                     // (ie: we received a vsync event)
                     if (connection->count == 0) {
                           // fired this time around
                           connection->count = -1;
                           signalConnections.add(connection);
                           added = true;
                     } else if (connection->count == 1 ||
                              (vsyncCount % connection->count) == 0) {
                           // continuous event, and time to report it
                           signalConnections.add(connection);
                           added = true;
                     }
                  }
               }

               if (eventPending && !timestamp && !added) {
                  // we don't have a vsync event to process
                  // (timestamp==0), but we have some pending
                  // messages.
                  signalConnections.add(connection);
               }
         } else {
               // we couldn't promote this reference, the connection has
               // died, so clean-up!
               mDisplayEventConnections.removeAt(i);
               --i; --count;
         }
      }

      // Here we figure out if we need to enable or disable vsyncs
      if (timestamp && !waitForVSync) {
         // we received a VSYNC but we have no clients
         // don't report it, and disable VSYNC events
         disableVSyncLocked();
      } else if (!timestamp && waitForVSync) {
         // we have at least one client, so we want vsync enabled
         // (TODO: this function is called right after we finish
         // notifying clients of a vsync, so this call will be made
         // at the vsync rate, e.g. 60fps.  If we can accurately
         // track the current state we could avoid making this call
         // so often.)
         enableVSyncLocked();
      }

      // note: !timestamp implies signalConnections.isEmpty(), because we
      // don't populate signalConnections if there's no vsync pending
      if (!timestamp && !eventPending) {
         // wait for something to happen
         if (waitForVSync) {
               // This is where we spend most of our time, waiting
               // for vsync events and new client registrations.
               //
               // If the screen is off, we can't use h/w vsync, so we
               // use a 16ms timeout instead.  It doesn't need to be
               // precise, we just need to keep feeding our clients.
               //
               // We don't want to stall if there's a driver bug, so we
               // use a (long) timeout when waiting for h/w vsync, and
               // generate fake events when necessary.
               bool softwareSync = mUseSoftwareVSync;
               nsecs_t timeout = softwareSync ? ms2ns(16) : ms2ns(1000);
               if (mCondition.waitRelative(mLock, timeout) == TIMED_OUT) {
                  if (!softwareSync) {
                     ALOGW("Timed out waiting for hw vsync; faking it");
                  }
                  // FIXME: how do we decide which display id the fake
                  // vsync came from ?
                  mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
                  mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
                  mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
                  mVSyncEvent[0].vsync.count++;
               }
         } else {
               // Nobody is interested in vsync, so we just want to sleep.
               // h/w vsync should be disabled, so this will wait until we
               // get a new connection, or an existing connection becomes
               // interested in receiving vsync again.
               mCondition.wait(mLock);
         }
      }
   } while (signalConnections.isEmpty());

   // here we're guaranteed to have a timestamp and some connections to signal
   // (The connections might have dropped out of mDisplayEventConnections
   // while we were asleep, but we'll still have strong references to them.)
   return signalConnections;
}
```

初次运行`waitForEvent()`时，由于`EventThread::onVSyncEvent()`和`EventThread::onHotplugReceived()`方法都没有被调用。因此经过循环:

```c++{.line-numbers}
for (int32_t i=0 ; i<(int32_t)mVSyncEvent.size(); i++) {
```

之后，__timestamp的值还是0__。同样地，`mPendingEvents`为空，因此`eventPending=false`.
前面提到，在`Connection`的`onFirstRef()`方法中，会将自己加到`mDisplayEventConnections`中。因此在这里，`mDisplayEventConnections.size()=1`。而EventThread::Connection的构造方法中，其成员count的初始值为-1。因此代码块：

```c++{.line-numbers}
if (connection->count >= 0) {
   ...
}
```

不会执行到，进而`waitForVSync=false`.
于是执行代码:

```c++{.line-numbers}
if (!timestamp && !eventPending) {
   if (waitForVSync) {
   } else {
      mCondition.wait(mLock);
   }
```

EventThread线程会一直等待下去(这里没有Timeout).

### IDisplayEventConnection的接口

ventThread::Connection类是BnDisplayEventConnection的服务端，其客户端IDisplayEventConnection开放出来的接口只有三个:

```c++{.line-numbers}
class IDisplayEventConnection : public IInterface {
public:
    DECLARE_META_INTERFACE(DisplayEventConnection)

    /*
     * stealReceiveChannel() returns a BitTube to receive events from. Only the receive file
     * descriptor of outChannel will be initialized, and this effectively "steals" the receive
     * channel from the remote end (such that the remote end can only use its send channel).
     */
    virtual status_t stealReceiveChannel(gui::BitTube* outChannel) = 0;

    /*
     * setVsyncRate() sets the vsync event delivery rate. A value of 1 returns every vsync event.
     * A value of 2 returns every other event, etc. A value of 0 returns no event unless
     * requestNextVsync() has been called.
     */
    virtual status_t setVsyncRate(uint32_t count) = 0;

    /*
     * requestNextVsync() schedules the next vsync event. It has no effect if the vsync rate is > 0.
     */
    virtual void requestNextVsync() = 0; // Asynchronous
};
```

其中用于控制EventThread的接口是`setVsyncRate()`和`requestNextVsync()`. 需要注意这里的注释:

1. 传入setVsyncRate()的参数 __count的意义是vsync rate__. 它指定了每N个VSYNC事件进行一次传递。当传入的值是1时，就是每次VSYNC事件都进行传递；当传入值是2时，就是每两次VSYNC事件才进行一次传递；以此类推.
2. requestNextVsync()用于VSYNC事件单次的传递，而它也 __只是在vsync rate小于0时起作用__.

再来看EventThread::Connection中这两个接口的实现:

```c++{.line-numbers}
void EventThread::setVsyncRate(uint32_t count,
      const sp<EventThread::Connection>& connection) {
   if (int32_t(count) >= 0) { // server must protect against bad params
      Mutex::Autolock _l(mLock);
      const int32_t new_count = (count == 0) ? -1 : count;
      if (connection->count != new_count) {
         connection->count = new_count;
         mCondition.broadcast();
      }
   }
}

void EventThread::requestNextVsync(
      const sp<EventThread::Connection>& connection) {
   Mutex::Autolock _l(mLock);

   mFlinger.resyncWithRateLimit();

   if (connection->count < 0) {
      connection->count = 0;
      mCondition.broadcast();
   }
}
```

这两个方法都会设置Connection类中的count成员变量（即vsync rate）. 接下来先研究requestNextVsync()被调用后waitForEvent()的情况.

### EventThread::requestNextVsync()之后执行waitForEvent()

`EventThread::requestNextVsync()`会把`EventThread::Connection`的`count`的值赋为0并唤醒`EventThread`线程。由于`waitForEvent()`方法的主体是一个`do...while (signalConnections.isEmpty())`循环，而`signalConnections`依然为空，所以当线程被唤醒之后，还是从`waitForEvent()`继续.
此时timestamp依然为0，eventPending也依然为false。但由于connection->count的值变成了0，因此会执行如下的代码块

```c++{.line-numbers}
if (connection->count >= 0) {
   // we need vsync events because at least
   // one connection is waiting for it
   waitForVSync = true;
   if (timestamp) {
      // we consume the event only if it's time
      // (ie: we received a vsync event)
      if (connection->count == 0) {
            // fired this time around
            connection->count = -1;
            signalConnections.add(connection);
            added = true;
      } else if (connection->count == 1 ||
               (vsyncCount % connection->count) == 0) {
            // continuous event, and time to report it
            signalConnections.add(connection);
            added = true;
      }
   }
}
```

可以看到此时`waitForVSync=true`. 由于`timestamp=0`而`waitForVSync=true`，因此满足条件

```c++{.line-numbers}
else if (!timestamp && waitForVSync) {
```

从而会执行`EventThread::enableVSyncLocked()`. 且满足如下条件:

```c++{.line-numbers}
if (!timestamp && !eventPending) {
   if (waitForVSync) {
      bool softwareSync = mUseSoftwareVSync;
      nsecs_t timeout = softwareSync ? ms2ns(16) : ms2ns(1000);
      if (mCondition.waitRelative(mLock, timeout) == TIMED_OUT) {
         mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
         mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
         mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
         mVSyncEvent[0].vsync.count++;
      }
   }
}
```

由于`mUseSoftwareVSync`的值为false，因此这里设置timeout为1秒钟并执行:

```c++{.line-numbers}
mCondition.waitRelative(mLock, timeout)
```

这里有三种情况:

1. 在超时之前又有`EventThread::requestNextVsync()`被调用，从`EventThread::requestNextVsync()`的实现可以看到，由于此时`EventThread::Connection`的`count`成员的值依然为0，那么该方法是不能唤醒EventThread线程的.
2. 在超时之前发生了VSYNC事件.
3. 发生了超时. 此时会生成一个模拟的VSYNC信号。这样当来到新的do...while()循环中时，局部变量timestamp就会有值，因此会把count变量重置为-1，并向signalConnections中添加Connection的实例. 进而跳出 `do...while`循环. 然后回到EventThread::threadLoop()，执行Connection::postEvent()方法，向Connection的成员变量发送消息:

```c++{.line-numbers}
status_t EventThread::Connection::postEvent(
      const DisplayEventReceiver::Event& event) {
   ...
   ssize_t size = DisplayEventReceiver::sendEvents(mChannel, &event, 1);
   return size < 0 ? status_t(size) : status_t(NO_ERROR);
}
```

由于EventThread继承自system/core中的Thread类，因此当postEvent()方法执行完，然后threadLoop()方法返回true之后，会继续执行threadLoop()方法。此时`waitForEvent()`方法中，timestamp的值为0，count的值为-1，进而waitForVSync的值为false，那么会执行到

```c++{.line-numbers}
mCondition.wait(mLock);
```

EventThread会一直等待下去。直到下一个requestNextVsync()发生.

### Vsync发生后

当`setVSyncEnable()`方法执行后，Vsync事件就可以从`DispSyncThread`起，顺序执行:

1. DispSyncThread的fireCallbackInvocations()
2. DispSyncSource的onDispSyncEvent()
3. EventThread的onVSyncEvent()

在`EventThread::onVSyncEvent()`中，设置了`mVSyncEvent`并唤醒了`EventThread`线程。此时再执行`waitForEvent()`时，timestamp有值，而Connection的count值不确定:

1. count的值为-1。说明在onVSyncEvent()被调用之前已经发生了TIMEOUT，EventThread处于一直等待的状态。此时满足条件`timestamp && !waitForVSync` 因此会执行disableVSyncLocked():

   ```c++{.line-numbers}
   void EventThread::disableVSyncLocked() {
      if (mVsyncEnabled) {
         mVsyncEnabled = false;
         mVSyncSource->setVSyncEnabled(false);
         mDebugVsyncEnabled = false;
      }
   }
   ```

   该方法调用了DispSyncSource的`setVSyncEnabled(false)`方法，最终会执行DispSyncThread的`removeEventListener()`方法，从mEventListeners中移除了DispSyncSource这个Callback，那么`DispSyncThread::fireCallbackInvocations()`方法是不会传给EventThread了。
   继续执行`waitForEvent()`，由于`signalConnections`内容还是为空，因此还是在do...while()的循环里面。再次开始循环时，timestamp的值已经是0。这是因为上一个循环中，当给timestamp赋值之后，会设置`mVSyncEvent[i].header.timestamp`的值为0.
   此时waitForEvent()最终会走到：

   ```c++{.line-numbers}
   mCondition.wait(mLock);
   ```

   从而EventThread线程一直等待下去.

2. count的值为0。说明waitForEvent()此时的状态是停留在:

   ```c++{.line-numbers}
   mCondition.waitRelative(mLock, timeout)
   ```

   并且还没有超时。当本线程被唤醒之后，会重新执行do...while()循环。此时timestamp的值非0，而count的值为0，此时会把waitForVSync赋为true，count的值赋为-1，并把当前Connection的实例添加到signalConnections中。这样signalConnections非空，走出do...while()循环。然后EventThread::threadLoop()调用postEvent()方法。接下来再次执行waitForEvent()，最终会执行:

   ```c++{.line-numbers}
   mCondition.wait(mLock);
   ```

   EventThread会一直等待下去。直到下一个requestNextVsync()发生.

### 调用setVsyncRate(1)之后

第一次执行waitForEvent()时，会执行enableVSyncLocked()。而无须等待requestNextVsync()的发生。然后waitForEvent()会执行到mCondition.waitRelative()。接下来有3种情况:

1. requestNextVsync()发生了：无变化.
2. `onVSyncEvent()`比超时先发生：`waitForEvent()`会从do...while()中跳出来，并执行EventThread::postEvent()方法。再次执行`waitForEvent()`时，会执行到`mCondition.waitRelative()`.
3. `onVSyncEvent()`比超时晚发生：当超时发生时，`waitForEvent()`会从`do...while()`中跳出来，并执行`EventThread::postEvent()`方法。再次执行`waitForEvent()`时，会执行到`mCondition.waitRelative()`，而非`mCondition.wait()`。接下来`onVSyncEvent()`发生时，情况与2同。

可以看到，当count的值为1时，__产生的效果是消除了对requestNextVsync()方法的依赖__.
于是可以看到每次VSYNC事件到来之后，都能最终调用到`SurfaceFlinger::handleMessageInvalidate()`

## 小结

1. 从SurfaceFlinger的init()发生到第一次执行EventThread的waitForEvent()，EventThread线程会执行到mCondition.wait()从而一直等待。由于此时尚未enableVSyncLocked()，那么EventThread是不会执行onVSyncEvent()方法的.
2. requestNextVsync()发生后，会执行enableVSyncLocked()，waitForEvent()会执行到mCondition.waitRelative()，接下来有三种情况:
2.1 requestNextVsync()比onVSyncEvent()先发生：此时无变化
2.2 __onVSyncEvent()比超时先发生__：waitForEvent()会从do...while()中跳出来，并执行EventThread::postEvent()方法。再次执行waitForEvent()时，该线程会执行到mCondition.wait()一直等待下去.
2.3 __onVSyncEvent()比超时晚发生__：当超时发生时，waitForEvent()会从do...while()中跳出来，并执行EventThread::postEvent()方法。再次执行waitForEvent()时，该线程会执行到mCondition.wait一直等待下去。接下来onVSyncEvent()发生时，会执行disableVSyncLocked()，从此阻止onVSyncEvent()。然后waitForEvent()会执行到mCondition.wait()一直等待下去.
然后ventThread线程等待下一个requestNextVsync()的发生.
3. 调用`EventThread.setVsyncRate(1)` 可以使EventThread不再依赖`requestNextVsync()`. 进而每次VSYNC事件到来之后，都能最终调用到`SurfaceFlinger::handleMessageInvalidate()`.