在Android中使用Java8的lambda表达式
===

![showcase][]
--

好久没写blog了，原因也懒得解释，无非是太懒了。不过最近开始坚持跑步了，跑了也快两个月了，坚持下来的感觉真的很好，不管跑得快与慢，长与短，总之，去跑就对了。所以，关键就是，去写就对了，至于内容，嗯哼，肯定是积极的啦！

写写关于我目前的工作上的东东吧，最近这段时间都在忙公司的Android端app，一个人也要是一个团队，仿佛自己好像已经习惯了这种节奏...写Android应用，开发语言自然是Java，我用Java也蛮久了，从最开始的Java SE到Java EE的一大堆开源框架到现在的Android应用，感觉这语言比起现在流行的一些语言诸如Go，JavaScript，Swift等，确实感觉编写的过程有时候挺慢的，真不如那些语言风骚啊。不过也还好了，语言工具嘛，用得顺手达到目的就好了。嗯哼，不过身边的同事貌似都是Java黑，哎，好吧。我承认，在语言简洁性上面Java确实啰嗦。不过今年春Oracle官方终于释出了最新的JDK8，引入了函数式编程，lambda表达式等新特性，虽然推出来有点晚，但是好歹也是有了嘛。

坑爹的在于，Android使用的Java版本不是Oracle她家的，而是一个已经废弃了的Apache Harmony项目，这个项目大概就是兼容了JDK6的一个Java API...Google和Oracle这两家还一直在闹个不停...总之，Android的Java可以理解为Java6。貌似Google已经改进可以兼容Java7的语法了（好像还是只能用在Kitkat上）...我们想要使用lambda还是不行...

happy coding, happy hacking. 开源社区可是很伟大的，Google一下，你就会发现有一个叫做[retro-lambda][]的项目，它可以在JDK6使用上lambda的语法，另外，还有一个基于Gradle的插件[gradle-retro-lambda][]，可以在Android Studio中编译通过lambda的语法。看看下面的代码对比，是不是感觉很棒呢~

```java
public class MainActivity extends ActionBarActivity {
  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    TextView text = new TextView(this);
    text.setText("Hello, lambda in Android!");
    setContentView(text);

    // add an on click listener

    // 1. the old way
    text.setOnClickListener(new View.OnClickListener() {
      @Override public void onClick(View v) {
        Toast.makeText(MainActivity.this, "salutation!", Toast.LENGTH_SHORT).show();
      }
    });
    // 2. the lambda way
    text.setOnClickListener(v -> Toast.makeText(this, "salutation!", Toast.LENGTH_SHORT).show());
    // usually in android we must separate worker and ui thread
    new Thread(() -> {
      try {
        // simulate a worker thread
        TimeUnit.SECONDS.sleep(3);
      } catch (InterruptedException ignore) {
      }
      // post to ui thread
      runOnUiThread(() -> text.setText("after hard work =.="));
    }).start();
  }
}
```

看看上面的代码，是不是感觉用这种方式写起来很优雅呢？在八月底，我已经将公司的app切换到这种模式来了，目前用着挺好的，还没有发现什么问题。但是有两件事得注意一下。第一是前天我刚升级到Android Studio 0.9编译项目的时候提示出了问题，``Cannot invoke method remove() on null object``，Google了一下，发现原来gradle-retro-lambda这插件的问题，然后升级了这插件就没有问题了。第二是如果你最后打包的apk需要混淆，貌似有些问题，之前我尝试过混淆，没有成功，由于项目紧，就没有继续尝试了，以后还会尝试。

如果你在使用的过程中有什么问题或者发现欢迎告诉我~

### EOF
```yaml
background: https://ww4.sinaimg.cn/large/aa61ae0egw1ely9jgwzenj21io0v4aht.jpg
hide: false
license: cc-40-by
location: Shenzhen
summary: ""
tags:
- Android
weather: ""
date: 2014-11-04T00:45:07+08:00
```

[showcase]: https://ww4.sinaimg.cn/large/aa61ae0egw1ely9jgwzenj21io0v4aht.jpg
[retro-lambda]: https://github.com/orfjackal/retrolambda
[gradle-retro-lambda]: https://github.com/evant/gradle-retrolambda

