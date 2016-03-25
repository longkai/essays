是时候使用Android Studio写Android应用了
===
香皂用完了，来sz也俩月了，Hello, sz ^_^

最近工作挺忙的，一个人负责公司的Android端，每天都会有新的发现或者想法，并且将其运用在项目中，感觉还蛮不错的。我想我现在不记下来和大家分享一下，以后也很难了0.0.

于是，决定每天下班前都和大家分享今天的发现，不光是技术哦，还有你懂的~但是，我很懒的，不保证都能更新，也不保证有意思，技术这东西，唉。。。

那么开始吧。

---

上周末，趁着刚release完一个阶段的版本，将项目重构了一下。具体主要是将项目IDE从Intellj IDEA切换到了Android Studio，支持使用Java8的lambda表达式写Android了哦，并且提供了多渠道自动打包的功能，想打多少渠道包就打多少，不费事儿，之前，按照公司要求，打了140多个渠道包，写的自动打包脚本够疼的（当时使用的是IDEA），各种问题，不过搞定了之后还真有意思的！

如果你现在还是用Eclipse写Android那么你真的可以尝试切换到Android Studio了，Eclipse支持的构建真的落后了，你能想象在Eclipse加入一个带有res目录的lib是有多蛋疼吗？btw, Google也已经将中心放在了Android Studio上。如果你现在用IDEA开发，那么切换到Android Studio毫无压力（Android Studio就是基于IDEA，除了构建方式和一些new features，并无二致），只会更easy。

一年前的这个时候，那时我还在用Eclipse写Java EE，当时虽然Eclipse已经被我玩得「一溜一溜」的，但是总是觉得还是缺乏灵活，当时看到Google推出的基于IDEA的Android Studio，认识了IDEA（其实早有听闻，只是没有尝试），后来花了三天熟悉，就再也不想用Eclipse了，后来陆续使用Gradle取代Maven开发项目了...

再后来写Android，尝试用了Studio，慢的简直无法忍受，而且各种奇奇怪怪的问题，IDEA则是速度秒杀的节奏，唯一不爽的IDEA的项目构建虽然引入了Module化，但是各种项目间的依赖得你自己去调，对Android了解不深的同学估计过不了这关，就很难再继续了。调几次还好（并且可以加深项目的熟悉程度），但是调多次就烦了。Android Studio当时就是在项目构建上优秀些。

前几天，看到Android Studio到了0.8.x的beta版了，试了试，发现快了很多耶，虽然还是慢于IDEA，但是已经可以接受了。而且一些feture很赞~并且当时项目的构建已趋复杂，并且多渠道打包那一个叫麻烦，于是趁着周末，切换了。用来开发一周了，感觉很不错，一些很nice的feature接下来会介绍~

如果是旧项目切换过来，那么，项目结构和Android Studio的默认项目结构基本上是不一致的，如果使用了git这样的vcs，肯定是不能任性改项目目录的。好在Android Studio（实质是Gradle）有着很灵活的构建方式。附一段兼容项目目录结构代码

```groovy
android {
 sourceSets {
    main {
      assets.srcDirs = ['assets']
      res.srcDirs = ['res']
      aidl.srcDirs = ['src']
      resources.srcDirs = ['src']
      renderscript.srcDirs = ['src']
      java.srcDirs = ['src']
      jniLibs.srcDirs = ['jni'] // native lib
      manifest.srcFile 'AndroidManifest.xml'
    }
  }
}
```

---
by longkai on 2014-08-30 in Sz.

### EOF
```json
{
  "tags": ["Android"],
  "render_option": 0,
  "date": "2014-08-31T01:09:52+08:00",
  "weather": "",
  "summary": "",
  "location": "Shenzhen",
  "background": ""
}
```
