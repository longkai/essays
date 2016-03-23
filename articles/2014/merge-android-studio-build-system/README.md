android项目两种构建方式的整合（Eclipse/idea和Android Studio）
============================================================

## android的两种构建方式
目前android主要有两种构建方式，一种基于ant（传统的），另一种是13年Google/IO上新推出基于Gralde的构建（Android Studio）。从sdk的samples的例程也可以看到api18后的例程的项目目录结构也变了。

简单的看一下两种构建方式的目录结构，我们以当前最新sdk19为例

1. 传统的ant构建（Eclipse和Idea默认的Android结构）

``$ANDROID_HOME/samples/android-19/legacy/ActionBarCompat``

```
res/
src/
AndroidManifest.xml
```

这是最简单的结构，可能还会有assets，libs等目录，也就是我们在ide新建一个android项目的骨架结构啦Orz

2. 基于Gradle的Android Studio构建目录

``$ANDROID_HOME/samples/android-19/ui/DoneBar``

```
DoneBarSample/
	src/
		main/
			java/
			res/
			AndroidManifest.xml
	tests/
		src/
			AndroidManifest.xml
	build.gradle  	# DoneBarSample子项目的gradle构建脚本
gradle/				# gradle临时文件夹，不用管
build.gradle		# 根项目（DoneBar）的gradle构建脚本
gradlew				# gradle-wrapper在windows平台运行脚本（有了这个在本地可以无需安装Gralde）
gradlew.bat			# gradle-wrapper在linux，mac平台下的运行脚本（效果同上）
settings.gradle		# gradle多项目的项目申明文件
README.txt
```

看起来，Gradle好像更搞得更复杂了，但是，**gradle的优势在于多项目的构建**。实际上，1方案只是一个我们项目核心的源代码而已，没有任何的依赖。通常情况下，**当我们写android应用时，会依赖第三方库**，如果是个jar还好办，但是不少情况下同时需要引入第三方的资源文件（比如说actionbar-compat，actionbar-sherlock等），这样就相当于把第三方库作为一个项目给引入到我们这个独立的项目中来了（并且这里面的项目之间的依赖还得自己去调控，比如说咱们的项目依赖于support-v4, 那好，我们把v4引入然后在项目中申明依赖关系，接下来咱们的项目依赖support-v7-appcompat，导入这个库，尼玛，这个库不仅有jar，还有资源文件，那好把这两个引入并声明好依赖关系，这时，你会发现项目依旧报错，因为v7那个jar依赖于v4那个jar...)

很讨厌是不是？还好啦，不过配多了你就觉得蛋疼了。所以，就是为什么Gradle比较有优势的地方了。对于以上的问题，我们只需要在项目中的``build.gradle``声明

```groovy
dependencies {
	compile "com.android.support:support-v4:$supportLibVersion"
	compile "com.android.support:support-v13:$supportLibVersion"
	// compile project(':your project')
	// compile ('libs/*.jar') // all your jar in the libs dir
}
```

这只是最简单的应用，gradle还提供了很多构建的特性，比如直接把第三方库作为一个子项目依赖进来，具体参阅其文档

## 我们是否应该从ant构建迁移到Androd Studio的Gralde构建？
很明显，Android Studio是google力推的开发工具，是趋势，而且，老实说Eclipse在开发android应用方面不如Android-Studio（idea）好使。但是，Android Studio目前还没有到正式版，还在开发阶段，出个bug你也伤不起Orz

另外，Gradle的构建目前还是很慢，相对与ant的构建，慢了好多，改了一处地方，run，要等不少时间，而我在idea或者eclipse上很快就构建好了

综上，没有解决以上两个问题还是不太推荐android-studio的，不过，倒是挺建议使用eclipse开发android的朋友有使用Intellij IDEA的开源社区版去开发android，事实上，Android Studio就是架在IDEA上的嘛

## 一种过渡的方式，同时支持两种构建方式
对于现有的项目或者新的项目，以后怎么转移到Android Studio呢，或者说，我就是想试试我这个项目在Android  Studio上开发爽不爽？出于Gradle的灵活性，我们完全可以做出一个同时支持两种构建方式的项目。

项目的结构和传统的一样，这样便可以作为android项目直接导入到eclipse或者idea中，核心在于gradle的``build.gradle``文件

不做过多介绍，直接上代码，看着改改就可以啦，还有更多的需求的话可以参考这里[](http://tools.android.com/tech-docs/new-build-system/user-guide)：

```groovy
/*
 * The MIT License (MIT)
 * Copyright (c) 2014 longkai
 * The software shall be used for good, not evil.
 */
buildscript {
	repositories {
		mavenCentral()
	}
	dependencies {
		classpath 'com.android.tools.build:gradle:0.6.+'
	}
}

// for our non-android-support libs, such as gson, etc.
repositories {
	// prefer and fall back
	jcenter()
	mavenCentral()
}

apply plugin: 'android'

ext {
	supportLibVersion = '19.0.1'
}

android {
	compileSdkVersion 19
	buildToolsVersion = '19.0.1'

	sourceSets {
		defaultConfig {
			testPackageName 'tingting.chen.tests'
		}

		main {
			assets.srcDirs = ['assets']
			res.srcDirs = ['res']
			aidl.srcDirs = ['src']
			resources.srcDirs = ['src']
			renderscript.srcDirs = ['src']
			java.srcDirs = ['src']
			manifest.srcFile 'AndroidManifest.xml'
		}

		instrumentTest {
			assets.srcDirs = ["tests/assets"]
			res.srcDirs = ["tests/res"]
			resources.srcDirs = ["tests/src"]
			java.srcDirs = ["tests/src"]
		}
	}
}

dependencies {
	// if you use Android Studio with a lib has its own res/ directory,
	// and that lib is not available in remote maven repo,
	// you need to use gradle' s multi-project build facility.
	// if you don' t know how it works, please refer Gradle' s docs or google.
	compile files("libs/*.jar")
	compile "com.android.support:support-v4:$supportLibVersion"
	compile "com.android.support:support-v13:$supportLibVersion"
	// please download google android-volley and compile it to a jar or multi-project build!
}

task wrapper(type: Wrapper) {
	// 1.9+ is not supported for now!
	gradleVersion = 1.8
}
```

---
本作品采用知识共享署名-相同方式共享 4.0 国际许可协议进行许可。


### EOF
```json
{
  "tags": ["Campus", "Android"],
  "reserved": false,
  "date": "2014-03-07T00:02:51+08:00",
  "weather": "",
  "summary": "",
  "location": "Nanning",
  "background": "/assets/images/xida.jpg"
}
```
