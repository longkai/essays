Basic Script
===

Every thing in Gradle sits on top of two basic concepts: **project** and **tasks**

1. each gradle **build** is made up of one or more **projects**
2. each gradle **project** is made up of noe or more **task**

## a simple hello world of gradle task

```groovy
task hello {
  doLast {
    println 'Hello, World!'
  }
}
```
the shortcut of the above sample, as you can see below, it' s nicer than above

```groovy
task << {
  println 'Hello, World!'
}
```

下面来简单的介绍一下gradle构建的基本语法。

build.gradle
```groovy
task << upper {
  String str = 'hello'
  println 'before: ' + str
  println 'after: ' + str.toUpperCase()
}
```
在shell中键入`gradle -q upper`就会得到下面的的结果：

```
before: hello
after: HELLO
```

## task的依赖
比如task b的运行必须依赖于task a的运行
```groovy
task a << {
  println 'a'
}

task b(dependsOn: a) << {
  println 'b'
}
```
运行`gradle -q a`便会得到下面的结果
```
a
b
```
当然，并不是意为着a 一定到先于a申明，下面的例子说明了这一点
```groovy
task a(dependsOn: 'b') << {
  println 'a'
}

task b << {
  println 'b'
}
```
这样也是可以的，但是如你所见，这样b就需要用引号括起来当作字符串看待，否则会报错的

## 默认的task
如果项目没有指定task，但是可以指定默认的task，比如在子项目中没有明确指出的话，可以继承父项目的task（如果父项目有的话）

```groovy
defaultTasks: 'clean', 'run'

task clean << {
  println 'clean...'
}

task run << {
  println 'run...'
}

task other << {
  ptinln 'othering...'
}
```

### EOF
```yaml
background: /assets/images/xida.jpg
date: 2013-08-07T04:09:20+08:00
hide: false
license: cc-40-by
location: Nanning
summary: ""
tags:
- Campus
- Gradle
weather: ""
```
