Android 构建技巧1（多运行时环境自动化构建）
===

**本文假设Android的开发环境为gradle编译。**

## 场景1
在Android的开发和测试中少不了构建的环节。通常我们会有一个测试环境（内部环境）和一个生产环境（最终用户使用的环境）。比如，在开发中我们使用后端接口的BASE_ENDPOINT可能是``http(s)://dev.example.com``，最终用户使用的BASE_ENDPOINT可能是``http(s)://api.example.com``。这样做的好处是是显然的，没有人会如此自信到用生产环境来开发。

在开发，打包和测试的过程中，少不了切换环境。这些**配置信息应该是写在自动化构建脚本中**，而不是打包前自己手动修改，这样改很乏味和易错。试想，开发调试到很晚都累成狗了，终于灭掉了list上的bug(s)，最后打包了，手抖把生产环境的配置写成了测试环境的，悲剧了...

## 技巧1
在build(debug或者release，或者其他的flavor)的时候自动切换配置。

在每一个android相关的项目中都会有一个``BuildConfig.java``文件，这个文件是由Android-Gradle插件自动生成的，我们不能手动修改它。正如它的名字一样，构建配置，这里面全是一些``public static final``常量，比如版本号，版本信息，是否debug环境等，这些常量均可以在程序中引用。试想，如果能够给这个文件添加一些构建信息，那不就解决**场景1**的问题了吗？

```groovy
buildTypes {
  debug {
    buildConfigField "String", "END_POINT", "\"http://dev.example.com\""
    buildConfigField "int", "FOO", "1"
    // others
  }

  release {
    buildConfigField "String", "END_POINT", "\"http://api.example.com\""
    signingConfig signingConfigs.release
    // others
  }
}
```

如上，那么对应到不同的环境（切换build variants或者打包（debug or release）），``BuildConfig.java``会追加相应的常量如下，你可以任意添加你想要的。

> debug env

```java
public static final String END_POINT = "http://dev.example.com";
```

> release env

```java
public static final String END_POINT = "http://api.example.com";
public static final int    FOO       = 1;
```

## 场景2
测试环境和生产环境的数据基本是不一样的。有些时候，测试完测试环境后，还得测一下生产环境（量变引起质变，不少时候一些测试环境没问题的到了生产环境就出问题了）。一般来说，这个时候，你得**打两个release包，一个是测试环境，一个是生产环境**。

## 技巧2
**技巧1**是理想的情况。只有打debug包的时候才会使用测试环境的配置。此时，我们需要一个release包也使用测试环境的数据。那么，这个时候就需要``productFlavors``来帮忙了，一次搞定俩包~

```groovy
productFlavors {
  // 测试环境下的release包
  internal {
    buildConfigField "String", "END_POINT", "\"http://dev.example.com\""
    // others
  }

  // 生产环境下的release包
  production {
    buildConfigField "String", "END_POINT", "\"http://api.example.com\""
    // others
  }
}
```

这里要注意，``BuildConfig.java``生成常量的顺序是先``buildType {}``，再``productFlavors``，最后到``defaultConfig {}``，**如果有重复的字段，那么只会生成最先发现的那个字段**，而不是后面的把前面的给覆盖掉，这应该是一个bug。

以上，happy coding~

===
2015-07-22

