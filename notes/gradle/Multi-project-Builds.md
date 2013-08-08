多项目构建
==========

**具体的细节说明不再赘述，有兴趣的可以直接去参考官方文档。这里就直接上例子了。**

## 定义通用的行为
下面这个例子是根项目叫做water，子项目为bluewhale的多项目工程。它的目录结构是这样的。
``` groovy
water/
  build.gradle
  settings.gradle
  bluewhale/
```

`settings.gradle`
```
include 'bluewhale'
```

父项目（water）的配置`build.gradle`
```groovy
Closure cl = { task -> println "I'm $task.project.name" }
task hello << cl
project(':bluewhale') {
    task hello << cl
}
```
当你在water/目录下执行`gradle -q hello`时便会得到如下的结果
```
> gradle -q hello
I'm water
I'm bluewhale
```
我们可以看到，父项目和子项目的名称都输出了。Gradle允许你在一个多项目的工程里在任意的build.gradle中访问任意的项目。project api提供了prject()接口调用。传递一个项目的路径给他他便会将该项目的对象传递回来给你。这个在gradle中叫做**跨项目配置**

但是我们仍然不爽，如果要为每个项目都手动的添加这样的task那就太麻烦了！我们可以做的更好！那么接下来就在现有的基础上再添加一个叫做*krill*的子项目。

项目目录结构如下：
```groovy
water/
  build.gradle
  settings.gradle
  bluewhale/
  krill/
```
`settings.gradle`
```groovy
include 'bluewhale', 'krill'
```
接下来我们重写刚才的build.gradle，只有一行！
```groovy
allprojects {
  task hello << { task -> println "I'm $task.project.name" }
}
```
执行`gradle -q hello`得到如下的结果
```
> gradle -q hello
I'm water
I'm bluewhale
I'm krill
```
这很酷。project API提供了一个`allproject`的属性，它会返回当前的项目和所有他里面的项目的一个list。如果你调用了allprojests并传递一个闭包进去，那么该闭包就会关联到所有的项目（声明在settings.gradle中的）
