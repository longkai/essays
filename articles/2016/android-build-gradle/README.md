Android build.gradle
===

> 温故而知新，可以为师矣。  -- 孔子

从今天开始，分享一些自己这两年多来的 Android 开发心得；有些东西你可能已经似曾相识在某些地方看过了或者已经掌握，这些都没有关系，因为：

- 这里没有转载
- 内容不是烂大街，尽可能地保持新鲜与实用性
- 不一定都是原创，但都包含自己的思考与实践

好了，就从最开始的 build.gralde 构建开始吧；这篇文章想要阐述的中心是**让配置和构建程序化**。

## Android Studio build.gradle
相信很多项目都已经开始使用 Android Studio 了，在构建上相比 Eclipse 最大的不同便是使用 [gradle][1] 来取代了 [ant][2]（实际上 gradle 底层也是使用 ant 来编译）。直观的说就是多出了 ``build.gralde`` 构建文件，在这文件里我们可能声明项目用到的库，插件，manifest 的包信息，多项目构建，多渠道 apk 等等。相信大家都清楚这文件可以干什么，那就直接进入下一节我想要说的吧。

## Top level 根目录的配置
不知道大家有没有注意，新建项目默认会在根目录生成一个这样的 ``build.gradle``

```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
  repositories {
    jcenter()
  }
  dependencies {
    classpath 'com.android.tools.build:gradle:2.0.0-alpha1'

    // NOTE: Do not place your application dependencies here; they belong
    // in the individual module build.gradle files
  }
}

allprojects {
  repositories {
    jcenter()
    mavenCentral()
  }
}

task clean(type: Delete) {
  delete rootProject.buildDir
}
```

**根目录的配置会影响所有的子项目**，在以上配置的情况下，所有的子项目的 classpath 都会加入 Android 环境（也就认为那个项目会是 android app/library 项目）；大多数情况下这样没问题，但是当项目存在**纯java**项目就会有问题了（直观的说，跑不了``public static void main(...)``）。

推荐的做法是**在子项目的 build.gradle 里声明**``buildscript { ... } ``部分；按需（根据项目类型，java，android lib，android app 等）配置环境，而不是一刀切。

## dependencies, plugins, ..., version 不统一？
build.gradle 最爽的，可能就是引入库是如此简单，比如项目需要``appcompat-v7``库，只需要一句

```groovy
compile 'com.android.support:appcompat-v7:23.2.0'
```

并且，appcompat 库自身的依赖也会给你找齐，非常棒；猜测有些项目的``dependencies { ... }``下应该声明了不少依赖。那么问题来了，如果有几个子项目依赖同样的库，如何保证他们的版本一致？同理，每个子项目都需要指定``compileSdkVersion``，``targetSdkVersion`` 等配置，整个项目版本应该保持一致才对（你肯定不想看到各个子项目依赖各种不同的 API level......）。

简单的办法是一个一个手动改，但这太容易出错了。这就像和写程序一样，这些都是变量，应该**统一声明到一个地方再引用**就行了，build.gradle 构建文件里的语句其实是 [groovy][3]，这也是一门程序语言。那就好办了，把所有的变量都定义到一个地方，这个地方显然是上一节提到的 top level 的``build.gradle``，举例来说：

```groovy
// top level build.gradle
ext {
  libraries = [
      android: 'com.android.tools.build:gradle:1.5.0'
      supportV4: 'com.android.support:support-v4:23.2.0'
  ]

  plugins = [
      android: 'com.android.application'
  ]
}
```

```groovy
// sub-project build.gradle
buildscript {
  dependencies {
    classpath rootProject.ext.libraries.android
  }
}

apply plugin: rootProject.ext.plugins.android
// ...

dependencies {
  compile rootProject.ext.libraries.supportV4
  // ...
}
```

## 更进一步
可以做得更彻底一些，把版本号，依赖，插件，甚至是环境变量等分别定义。参考如下，完整的栗子可以[看看这里][4]。

```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.

subprojects {
  buildscript {
    repositories {
      jcenter()
    }
  }

  repositories {
    jcenter()
  }

  configurations.all {
    resolutionStrategy {
      force rootProject.ext.dependencies.annotation
    }
  }
}

ext {
  versions = [
      // plugins
      android       : '1.5.0',
      fabric        : '1.+',
      apt           : '1.8',
      retrolambda   : '3.2.3',
      googleServices: '1.5.0',
      // build config
      compileSdk    : 23,
      targetSdk     : 23,
      buildTools    : '23.0.1',
      // testing
      mockito       : '1.+',
      dexmaker      : '1.2',
      rules         : '0.4.1',
      runner        : '0.4.1',
      espresso      : '2.2.1',
      // debug
      leakCanary    : '1.3.1',
      // dependencies
      colors        : '1.0',
      supportLibrary: '23.1.1',
      rxandroid     : '1.0.1',
      rxjava        : '1.0.16',
      butterKnife   : '7.0.1',
      timber        : '3.1.0',
      picasso       : '2.5.2',
      retrofit      : '1.9.0',
      okhttp        : '2.5.0',
      dagger        : '2.0.2',
      javax         : '10.0-b28',
      crashlytics   : '2.5.2',
      analytics     : '8.3.0',
  ]

  dependencies = [
      // plugins
      android        : "com.android.tools.build:gradle:$versions.android",
      fabric         : "io.fabric.tools:gradle:$versions.fabric",
      apt            : "com.neenbedankt.gradle.plugins:android-apt:$versions.apt",
      retrolambda    : "me.tatarka:gradle-retrolambda:$versions.retrolambda",
      googleServices : "com.google.gms:google-services:$versions.googleServices",
      // testing
      mockito        : "org.mockito:mockito-core:$versions.mockito",
      dexmaker       : "com.google.dexmaker:dexmaker:$versions.dexmaker",
      dexmakerMockito: "com.google.dexmaker:dexmaker-mockito:$versions.dexmaker",
      rules          : "com.android.support.test:runner:$versions.rules",
      runner         : "com.android.support.test:rules:$versions.runner",
      espresso       : "com.android.support.test.espresso:espresso-core:$versions.espresso",
      espressoContrib: "com.android.support.test.espresso:espresso-contrib:$versions.espresso",
      espressoIntents: "com.android.support.test.espresso:espresso-intents:$versions.espresso",
      // debug
      leakCanary     : "com.squareup.leakcanary:leakcanary-android:$versions.leakCanary",
      leakCcanaryNoop: "com.squareup.leakcanary:leakcanary-android-no-op:$versions.leakCanary",
      // dependencies
      colors         : "material:palette:$versions.colors",
      annotation     : "com.android.support:support-annotations:$versions.supportLibrary",
      v4             : "com.android.support:cardview-v7:$versions.supportLibrary",
      rxandroid      : "io.reactivex:rxandroid:$versions.rxandroid",
      rxjava         : "io.reactivex:rxjava:$versions.rxjava",
      butterKnife    : "com.jakewharton:butterknife:$versions.butterKnife",
      appcompat      : "com.android.support:appcompat-v7:$versions.supportLibrary",
      design         : "com.android.support:design:$versions.supportLibrary",
      cardview       : "com.android.support:cardview-v7:$versions.supportLibrary",
      recyclerview   : "com.android.support:recyclerview-v7:$versions.supportLibrary",
      preference     : "com.android.support:preference-v7:$versions.supportLibrary",
      gridlayout     : "com.android.support:gridlayout-v7:$versions.supportLibrary",
      palette        : "com.android.support:palette-v7:$versions.supportLibrary",
      timber         : "com.jakewharton.timber:timber:$versions.timber",
      picasso        : "com.squareup.picasso:picasso:$versions.picasso",
      retrofit       : "com.squareup.retrofit:retrofit:$versions.retrofit",
      okhttp         : "com.squareup.okhttp:okhttp:$versions.okhttp",
      dagger         : "com.google.dagger:dagger:$versions.dagger",
      daggerCompiler : "com.google.dagger:dagger-compiler:$versions.dagger",
      javax          : "org.glassfish:javax.annotation:$versions.javax",
      crashlytics    : "com.crashlytics.sdk.android:crashlytics:${versions.crashlytics}@aar",
      analytics      : "com.google.android.gms:play-services-analytics:$versions.analytics",
  ]

  plugins = [
      android       : 'com.android.application',
      androidTest   : 'com.android.test',
      fabric        : 'io.fabric',
      apt           : 'com.neenbedankt.android-apt',
      retrolambda   : 'me.tatarka.retrolambda',
      googleServices: 'com.google.gms.google-services',
  ]

  repositories = [
      fabric   : 'https://maven.fabric.io/public',
      snapshots: 'https://oss.sonatype.org/content/repositories/snapshots/',
  ]
}
```

### EOF
```yaml
background: /assets/images/default.jpg
hide: false
license: cc-40-by
location: Shenzhen
summary: 让配置和构建程序化
tags:
- Android
weather: rainy
date: 2016-03-30T22:05:24+08:00
```

[1]: http://gradle.org/
[2]: http://ant.apache.org/
[3]: http://www.groovy-lang.org/
[4]: https://github.com/longkai/Musseta
