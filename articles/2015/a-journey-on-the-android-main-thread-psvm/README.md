挖挖Android 主线程的坟 - PSVM
===

> 这篇翻译得到了原作者的允许，原文链接：https://corner.squareup.com/2013/10/android-main-thread-1.html

有一篇关于编程恐惧的文章，谈到了为什么[我们要学会读源码][1]。Android天生就是开源的，这是她最棒的属性之一。

## PSVM
```java
public class BigBang {
  public static void main(String... args) {
    // The Java universe starts here.
  }
}
```
所有的java程序都是从调用``public static void main()``方法开始。这无论是对于java桌面程序，jee servlet服务端程序，还是Android应用都成立。

当Android系统启动时，系统会启动一个叫做**ZygoteInit**的Linux进程。这是在一个线程中载入了大量Android SDK通用类库，然后挂起的Dalvik VM（虚拟机）进程。

当启动一个新的Android应用时，Android系统会复制ZygoteInit进程，然后恢复之前挂起的那个线程，接着调用``ActivityThread.main()``。

![zygote](zygote.png)

维基百科上的解释，一个zygote就是一个受精的细胞

## Loopers
在我们继续深入之前，我们需要看看[Lopper][3]这个类。

使用Looper可以非常方便地让一个线程去按照顺序处理消息。每一个Looper都有一个``loop()``方法去处理消息队列里面的每一个消息，当消息队列为空时这个线程便会阻塞。

``Looper.loop()``方法的代码类似于下面这样：

```java
void loop() {
  while(true) {
    Message message = queue.next(); // blocks if empty.
    dispatchMessage(message);
    message.recycle();
  }
}
```

每个looper都会关联到一个线程。当要创建一个新的looper并且关联到当前的线程时，你必须调用``Looper.prepare()``。进程中所有的looper都被存放在``Looper``类的静态``ThreadLocal``成员变量中。你可以通过调用``Looper.myLooper()``拿到和当前线程相关联的Looper对象。

有个[HandlerThread][4]类帮你做了所有的事儿：

```java
HandlerThread thread = new HandlerThread("SquareHandlerThread");
thread.start(); // starts the thread.
Looper looper = thread.getLooper();
```

她的代码和下面类似：

```java
class HandlerThread extends Thread {
  Looper looper;
  public void run() {
    Looper.prepare(); // Create a Looper and store it in a ThreadLocal.
    looper = Looper.myLooper(); // Retrieve the looper instance from the ThreadLocal, for later use.
    Looper.loop(); // Loop forever.
  }
}
```

## Handlers
[handler][5]是looper的好基友，指腹为婚的~

使用Handler大抵有两个目的：

1. 在任意的线程发送消息到looper的消息队列中
2. 处理从looper中消息队列中出列的消息

```java
// Each handler is associated to one looper.
Handler handler = new Handler(looper) {
  public void handleMessage(Message message) {
    // Handle the message on the thread associated to the given looper.
    if (message.what == DO_SOMETHING) {
      // do something
    }
  }
};

// Create a new message associated to that handler.
Message message = handler.obtainMessage(DO_SOMETHING);

// Add the message to the looper queue.
// Can be called from any thread.
handler.sendMessage(message);
```

你可以把多个handler关联到一个looper上，looper会通过``message.target``把消息推送给目标。

一个简单常见的做法是通过handler把一个``Runnable``发送出去：

```java
// Create a message containing a reference to the runnable and add it to the looper queue
handler.post(new Runnable() {
  public void run() {
    // Runs on the thread associated to the looper associated to that handler.
  }
});
```

另外，一个handler可以不提供looper来去创建：

```java
// DON'T DO THIS
Handler handler = new Handler();
```

无参的构造函数会调用``Looper.myLooper()``去拿到和当前线程关联着的looper。这也许，也许不，是你的handler想要关联的那个looper。

大多数时候，你你需要创建一个能够在主线程处理消息的handler：

```java
Handler handler = new Handler(Looper.getMainLooper());
```

## Back to PSVM
让我们再瞧瞧``ActivityThread.main()``，这是她主要做的事情：
```java
public class ActivityThread {
  public static void main(String... args) {
    Looper.prepare();

    // You can now retrieve the main looper at any time by calling Looper.getMainLooper().
    Looper.setMainLooper(Looper.myLooper());

    // Post the first messages to the looper.
    // { ... }

    Looper.loop();
  }
}
```

现在你应该知道为毛这个线程叫主线程了:)

注意：和你预期的一样，主线程最开始做的事情之一就是创建[Application][6]对象然后调用``Application.onCreate()``方法。

[下一部分][7]，我们回去瞧瞧Android的生命周期和主线程的关系，以及为毛稍不注意，就会导致bug。

---
作者：[Pierre-Yves Ricau][8]

### EOF
```yaml
background: /assets/images/xida.jpg
date: 2015-06-10T23:24:22+08:00
hide: false
license: cc-40-by
location: Shenzhen
summary: 翻译
tags:
- Android
- Translation
weather: ""
```

[1]: http://www.codinghorror.com/blog/2012/04/learn-to-read-the-source-luke.html
[2]: https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/preloaded-classes
[3]: http://developer.android.com/reference/android/os/L
[4]: http://developer.android.com/reference/android/os/HandlerThread.html
[5]: http://developer.android.com/reference/android/os/Handler.html
[6]: http://developer.android.com/reference/android/app/Application.html
[7]: http://corner.squareup.com/2013/12/android-main-thread-2.html
[8]: http://piwai.info/
