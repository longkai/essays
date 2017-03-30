Android主线程和组件生命周期那些事儿
===

> 这篇翻译得到了原作者的允许，[原文链接](https://corner.squareup.com/2013/12/android-main-thread-2.html)

在[上一部分][1]中我们深入探讨了 looper 和 handler 以及他们和 Android 主线程之间的关系。

今天，我们会近距离的看看主线程是怎样和 Android 组件的生命周期交互的。

## Activities love orientation changes
Activity 喜欢拥抱变化（你没法预期甚至是限制变化，想想）。让我们从 Activity 的生命周期和处理奇妙的 configuration 变化背后开始。

### Why it matters
这篇文章灵感来源于一个发生在 [Square Register][2] 应用上的一个真实 crash。

一个简化后的代码如下：

```java
public class MyActivity extends Activity {
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Handler handler = new Handler(Looper.getMainLooper());
    handler.post(new Runnable() {
      public void run() {
        doSomething();
      }
    });
  }

  void doSomething() {
    // Uses the activity instance
  }
}
```

正如我们看到的，``doSomething()`` 能够在 Activity 的``onDestroy()`` 方法之后被调用，而 configuration 的 改变 则会导致 ``onDestroy()`` 被调用。在那个时刻，你不应该再使用 Activity 的实例（为什么，留给读者）。

### A refresher on orientation changes
设备的转向可以在任何时候改变。我们在 activity 被创建的时候调用``Activity#setRequestedOrientation(int)``来模拟一个屏幕转向。

你能够预测在竖屏的启动这个 activity 时的 log 输出吗？

```java
public class MyActivity extends Activity {
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Log.d("Square", "onCreate()");
    if (savedInstanceState == null) {
      Log.d("Square", "Requesting orientation change");
      setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
    }
  }

  protected void onResume() {
    super.onResume();
    Log.d("Square", "onResume()");
  }

  protected void onPause() {
    super.onPause();
    Log.d("Square", "onPause()");
  }

  protected void onDestroy() {
    super.onDestroy();
    Log.d("Square", "onDestroy()");
  }
}
```

如果你了解[Android lifecycle][3]，你也许会这样预测：

```
onCreate()
Requesting orientation change
onResume()
onPause()
onDestroy()
onCreate()
onResume()
```

Android 的生命周期正常的运转，activity created，resumed，然后屏幕转向变了导致 activity  paused，destroyed，接着一个新的 activity  created 和 resumed。

### Orientation changes and the main thread
这儿有一个重要的细节需要记住：一个屏幕转变将会导致 activity 简单地通过向主线程发送消息得到重建

让我们写个蜘蛛程序通过反射去监听 looper 中消息队列的内容瞧瞧这个细节：

```java
public class MainLooperSpy {
  private final Field messagesField;
  private final Field nextField;
  private final MessageQueue mainMessageQueue;

  public MainLooperSpy() {
    try {
      Field queueField = Looper.class.getDeclaredField("mQueue");
      queueField.setAccessible(true);
      messagesField = MessageQueue.class.getDeclaredField("mMessages");
      messagesField.setAccessible(true);
      nextField = Message.class.getDeclaredField("next");
      nextField.setAccessible(true);
      Looper mainLooper = Looper.getMainLooper();
      mainMessageQueue = (MessageQueue) queueField.get(mainLooper);
    } catch (Exception e) {
      throw new RuntimeException(e);
    }
  }

  public void dumpQueue() {
    try {
      Message nextMessage = (Message) messagesField.get(mainMessageQueue);
      Log.d("MainLooperSpy", "Begin dumping queue");
      dumpMessages(nextMessage);
      Log.d("MainLooperSpy", "End dumping queue");
    } catch (IllegalAccessException e) {
      throw new RuntimeException(e);
    }
  }

  public void dumpMessages(Message message) throws IllegalAccessException {
    if (message != null) {
      Log.d("MainLooperSpy", message.toString());
      Message next = (Message) nextField.get(message);
      dumpMessages(next);
    }
  }
}
```

如你所见，消息队列不过也就是一个链表，每个消息都有一个指向下一条消息的引用。

我们在屏幕转变的那个时刻 log 队列的内容：

```java
public class MyActivity extends Activity {
  private final MainLooperSpy mainLooperSpy = new MainLooperSpy();

  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Log.d("Square", "onCreate()");
    if (savedInstanceState == null) {
      Log.d("Square", "Requesting orientation change");
      setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
      mainLooperSpy.dumpQueue();
    }
  }
}
```

这是输出：

```
onCreate()
Requesting orientation change
Begin dumping queue
{ what=118 when=-94ms obj={1.0 208mcc15mnc en_US ldltr sw360dp w598dp h335dp 320dpi nrml land finger -keyb/v/h -nav/h s.44?spn} }
{ what=126 when=-32ms obj=ActivityRecord{41fd2b48 token=android.os.BinderProxy@41fcce50 no component name} }
End dumping queue
```

扫一眼[ActivityThread][4] 这个类，她告诉我们 118 和 126 是什么鬼：

```java
public final class ActivityThread {
  private class H extends Handler {
    public static final int CONFIGURATION_CHANGED   = 118;
    public static final int RELAUNCH_ACTIVITY       = 126;
  }
}
```

请求一个屏幕转向将会添加一个``CONFIGURATION_CHANGED``和一个``RELAUNCH_ACTIVITY``消息到主线程的 looper 的消息队列中。

让我们往回看想想到底发生了什么：

当 activity 第一次启动的时候，队列是空的。当前被执行的消息是``LAUNCH_ACTIVITY``，她会创建 activity 的势利，调用``onCreate()``接着调用``onResume()``，接着由主线程的 looper 处理她的队列里面的消息。

当设备的转向被检测到，一个``RELAUNCH_ACTIVITY``消息将发送到队列中。当这个消息被处理时，她：

1. 在老的 activity 实例依次调用``onSaveInstanceState()``, `` onPause()``, ``onDestroy()``
2. 创建一个新的 activity 实例
3. 调用新 activity 利的``onCreate()``和``onResume()``

### Tying it all together
如果在``onCreate()`` 转屏时使用 handler 发送消息，会发生啥？让我们看看这俩货，正好在转屏前后。

```java
public class MyActivity extends Activity {
  private final MainLooperSpy mainLooperSpy = new MainLooperSpy();

  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Log.d("Square", "onCreate()");
    if (savedInstanceState == null) {
      Handler handler = new Handler(Looper.getMainLooper());
      handler.post(new Runnable() {
        public void run() {
          Log.d("Square", "Posted before requesting orientation change");
        }
      });
      Log.d("Square", "Requesting orientation change");
      setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
      handler.post(new Runnable() {
        public void run() {
          Log.d("Square", "Posted after requesting orientation change");
        }
      });
      mainLooperSpy.dumpQueue();
    }
  }

  protected void onResume() {
    super.onResume();
    Log.d("Square", "onResume()");
  }

  protected void onPause() {
    super.onPause();
    Log.d("Square", "onPause()");
  }

  protected void onDestroy() {
    super.onDestroy();
    Log.d("Square", "onDestroy()");
  }
}
```

输出：

```
onCreate()
Requesting orientation change
Begin dumping queue
{ what=0 when=-129ms }
{ what=118 when=-96ms obj={1.0 208mcc15mnc en_US ldltr sw360dp w598dp h335dp 320dpi nrml land finger -keyb/v/h -nav/h s.46?spn} }
{ what=126 when=-69ms obj=ActivityRecord{41fd6b68 token=android.os.BinderProxy@41fd0ae0 no component name} }
{ what=0 when=-6ms }
End dumping queue
onResume()
Posted before requesting orientation change
onPause()
onDestroy()
onCreate()
onResume()
Posted after requesting orientation change
```

总结一下：在``onCreate()``的最后，消息队列包含了4条消息。第一条是在转屏前发送的，接下来的两条和转屏相关，最后一条发送在请求转屏之后。logs 表明了这些消息是按照顺序被执行的。

因此，任何在转屏前发送的消息都会在即将被销毁的 activity 的``onPause()``之前处理，然后任何在转屏之后发送的消息将会在即将被创建的 activity 的``onResume()``之后处理（再问自己，真的明白了？）。

这暗示我们，当发送一个消息时，我们**不能保证**那个 activity 的实例**在该消息被处理的时候仍然存活着**（即使你是在``onCreate()`` 或者``onResume()`` 中发送的）。如果你的消息持有一个 view 的引用或者一个 activity，那么那个 activity 将不会被垃圾处理器给回收，直到该消息被处理完毕（想一想，这样会造成什么问题？）。

### What could you do? 你该怎么做？

#### The real fix 真正把问题修复
不要调用``handler.post()`` 方法当你正好已经在主线程。在大多数实际情况下，``handler.post()``被用来快速修复有序的问题。调整你的架构而不是随意的调用``handle.post()``把事情搞得一团糟。

#### If you have a good reason to post 如果你有好的理由去发送消息
确保你的消息没有持有 activity 的引用，当你想要做一个后台操作时。

#### If you really need that activity reference 如果你真的需要引用 activity 对象
在 activity 的``onPause()``方法里调用``handler.removeCallbacks()``将消息从队列中移除。

#### If you want to get fired 如果想要直接直接看到效果
使用``handler.postAtFrontOfQueue()``去保证那条消息总是在``onPause()``之前被处理。如此一来，你的代码将会变得非常难以阅读和理解。严肃些，不要这样做。

#### A word on runOnUiThread() 对 runOnUiThread() 的简评
你有没有发现我们创建一个 handler 然后通过``handler.post()``发送消息而不是直接调用``Activity.runOnUiThread()``？

这就是为什么：

```java
public class Activity {
  public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
      mHandler.post(action);
    } else {
      action.run();
    }
  }
}
```

和``handler.post()``不同，如果当前就在主线程，``runOnUiThread()``并不会将 runnable 消息发送出去，而是同步地调用``run()``。

## Services
有个常见的错误观念需要摒弃： service **没有** 跑在后台线程上。

所有的 service 的生命周期方法（``onCreate()``, ``onStartCommand()``,  等等）都跑在主线程上（对，没错，就是那个跑各种动画的 activity 线程）。

不管你是在 service 还是 activity，耗时的任务必须在后台线程中跑。这个后台线程能够和你的 app 进程存活一样久（如果真要那么久），即使你的 acticity 早就 byebye 了。

然而，Android 系统可以在任何时候决定是否要 kill 掉你的 app 进程。一个 service 可以作为一个途径：请求系统如果可以的话让我们活下去，如果一定要跪，也要礼貌地让 service 知道自己快跪了，做好心理准备，而不是让咱突然死掉-.-

注意例外：当一个从``onBind()``返回的``IBinder``收到其他进程的调用，那么调用的方法将会在后台线程跑。

花些时间看看[Service documentation][5] -- 写得好棒。

### IntentService
[IntentService][6] 提供了一个简单的方式在后台线程里按顺序处理意图（a queue of intents）的队列。

```java
public class MyService extends IntentService {
  public MyService() {
    super("MyService");
  }

  protected void onHandleIntent(Intent intent) {
    // This is called on a background thread.
  }
}
```

在内部，她在一个``HandlerThread``里使用了一个``Looper``去处理所有的意图。当 service 被销毁的时候，looper 会让你结束处理当前的 intent，然后终止掉当前的后台线程。

## Conclusion
大多数 Android 的生命周期方法都在主线程被调用。可以简单的把这些回调方法当做发往主线程 looper 的消息队列中的消息。

如果忘了提醒，这篇文章不会完整：**不要阻塞主线程**。

有没有想过阻塞主线程到底意味着什么？这是下一篇的主题！

### EOF
```yaml
background: /assets/images/default.jpg
hide: false
license: cc-40-by
location: Shenzhen
summary: 翻译
tags:
- Android
- Translation
weather: ""
date: 2015-06-17T20:41:25+08:00
```

[1]: ../a-journey-on-the-android-main-thread-psvm/README.md
[2]: https://squareup.com/sell-in-store
[3]: http://developer.android.com/training/basics/activity-lifecycle/index.html
[4]: https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/android/app/ActivityThread.java
[5]: http://developer.android.com/reference/android/app/Service.html
[6]: http://developer.android.com/reference/android/app/IntentService.html

