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

## 子项目相关的配置
Project API同时也提供了只能够访问到子项目的属性。

### 定义通用的行为
`build.gradle`
```groovy
allprojects {
    task hello << {task -> println "I'm $task.project.name" }
}
subprojects {
    hello << {println "- I depend on water"}
}
```
`gradle -q hello`将会得到如下的输出
```
> gradle -q hello
I'm water
I'm bluewhale
- I depend on water
I'm krill
- I depend on water
```

为某个指定的项目定义特定的行为
```groovy
allprojects {
    task hello << {task -> println "I'm $task.project.name" }
}
subprojects {
    hello << {println "- I depend on water"}
}
project(':bluewhale').hello << {
    println "- I'm the largest animal that has ever lived on this planet."
}
```

```
I'm water
I'm bluewhale
- I depend on water
- I'm the largest animal that has ever lived on this planet.
I'm krill
- I depend on water
```
但是，就像我们之前所说的，我们通常更喜欢把某个项目相关的行为放到那个项目的`build.gradle`中去。下面让我们来重构一下我们项目的构建文件。

下面是项目的目录结构。
```
water/
  build.gradle
  settings.gradle
  bluewhale/
    build.gradle
  krill/
    build.gradle
```

`settings.gradle`
```groovy
include 'bluewhale', 'krill'
```

`bluewhale/build.gradle`
```groovy
hello.doLast { println "- I'm the largest animal that has ever lived on this planet." }
```

`krill/build.gradle`
```groovy
hello.doLast {
    println "- The weight of my species in summer is twice as heavy as all human beings."
}
```

`build.gradle`
```groovy
allprojects {
    task hello << {task -> println "I'm $task.project.name" }
}
subprojects {
    hello << {println "- I depend on water"}
}
```

`gradle -q hello`的输出
```
> gradle -q hello
I'm water
I'm bluewhale
- I depend on water
- I'm the largest animal that has ever lived on this planet.
I'm krill
- I depend on water
- The weight of my species in summer is twice as heavy as all human beings.
```

## 项目的过滤
未来来展示更多关于配置注入的强大，让我们在上一小节已有的基础上再添加一个叫做 *tropicalFish* 的项目

### 首先是通过名字来过滤
项目目录结构如下：
```
water/
  build.gradle
  settings.gradle
  bluewhale/
    build.gradle
  krill/
    build.gradle
  tropicalFish/
```  

`settings.gradle`
```groovy
include 'bluewhale', 'krill', 'tropicalFish'
```

`build.gradle`
```groovy
allprojects {
    task hello << {task -> println "I'm $task.project.name" }
}
subprojects {
    hello << {println "- I depend on water"}
}
configure(subprojects.findAll {it.name != 'tropicalFish'}) {
    hello << {println '- I love to spend time in the arctic waters.'}
}
```

输出
```
> gradle -q hello
I'm water
I'm bluewhale
- I depend on water
- I love to spend time in the arctic waters.
- I'm the largest animal that has ever lived on this planet.
I'm krill
- I depend on water
- I love to spend time in the arctic waters.
- The weight of my species in summer is twice as heavy as all human beings.
I'm tropicalFish
- I depend on water
```

configure()方法需要一个list的参数然后将配置应用与这个list里的所有item

### 通过属性来过滤项目
项目的目录结构
```
water/
  build.gradle
  settings.gradle
  bluewhale/
    build.gradle
  krill/
    build.gradle
  tropicalFish/
    build.gradle
```

`settings.gradle`
```groovy
include 'bluewhale', 'krill', 'tropicalFish'
```

`bluewhale/build.gradle`
```groovy
ext.arctic = true
hello.doLast { println "- I'm the largest animal that has ever lived on this planet." }
```

`krill/build.gradle`
```groovy
ext.arctic = true
hello.doLast {
    println "- The weight of my species in summer is twice as heavy as all human beings."
}
```

`tropicalFish/build.gradle`
```groovy
ext.arctic = false
```

`build.gradle`
```groovy
allprojects {
    task hello << {task -> println "I'm $task.project.name" }
}
subprojects {
    hello {
        doLast {println "- I depend on water"}
        afterEvaluate { Project project ->
            if (project.arctic) { doLast {
                println '- I love to spend time in the arctic waters.' }
            }
        }
    }
}
```

输出
```
> gradle -q hello
I'm water
I'm bluewhale
- I depend on water
- I'm the largest animal that has ever lived on this planet.
- I love to spend time in the arctic waters.
I'm krill
- I depend on water
- The weight of my species in summer is twice as heavy as all human beings.
- I love to spend time in the arctic waters.
I'm tropicalFish
- I depend on water
```

在构建文件中我们看到了 `afterEvaluate` 这个记号，它意为者我们传递的这个闭包只有在子项目得到执行后才会得到执行。我们也必须得再子项目的`arctic`属性得到赋值以后才能得到预期的结果。

## 多项目的执行规则
当我们在根项目里执行hello这个task时我们得到了比较直观的结果。在不同项目中的所有的hello任务都得到了执行。接下来让我们去bluewhale这个子项目里去执行gradle命令看看会得到什么结果。

```
> gradle -q hello
I'm bluewhale
- I depend on water
- I'm the largest animal that has ever lived on this planet.
- I love to spend time in the arctic waters.
```

> Gradle最基本的行为很简单。Gradle会从当前的目录开始向下查看整个项目的层次关系，去查找和task名字一样的任务然后去执行他们。 **有一件事情必须牢记：Gradle总是会对多项目里的每个子项目进行解析然后建立所有存在的task的对象。然后，更具输入的task名和当前的目录来过滤那些不需要的task。** 由于Gradle是一个跨项目的配置，所以只有在每一个个项目得到了解析之后给定的task才能得到执行。

接下来让我们给原有的项目添加task

`bluewhale/build.gradle`
```groovy
ext.arctic = true
hello << { println "- I'm the largest animal that has ever lived on this planet." }

task distanceToIceberg << {
    println '20 nautical miles'
}
```

`krill/build.gradle`
```groovy
ext.arctic = true
hello << { println "- The weight of my species in summer is twice as heavy as all human beings." }

task distanceToIceberg << {
    println '5 nautical miles'
}
```

结果
```
> gradle -q distanceToIceberg
20 nautical miles
5 nautical miles
```

下面是没有`-q`参数的结果
```
> gradle distanceToIceberg
:bluewhale:distanceToIceberg
20 nautical miles
:krill:distanceToIceberg
5 nautical miles

BUILD SUCCESSFUL

Total time: 1 secs
```

我们执行的目录是在water这个根目录，不管是water还是tropicalFish有没有distanceToIceberg这个task，Gradle并不在乎。最简单的规则已经在上面提到了： **在整个项目的task中找到给定了task，只有在所有的项目都没有这个task时Gradle才会报错！**

## 通过绝对路径执行task
像前面我们看到的，你可以在多项目里，进入某个子项目，然后在那个项目里执行命令。然后从那个项目开始，所有匹配到的task都会得到执行。Gradle同时也提供了通过绝对路径执行task。

```
> gradle -q :hello :krill:hello hello
I'm water
I'm krill
- I depend on water
- The weight of my species in summer is twice as heavy as all human beings.
- I love to spend time in the arctic waters.
I'm tropicalFish
- I depend on water
```
以上，是在tropicalFish这个子项目里执行的。它分别表示执行根项目，krill项目，和本项目里的hello这个task。

## 依赖
### 依赖和依赖执行的顺序
下面是项目的目录结构
```groovy
messages/
  settings.gradle
  consumer/
    build.gradle
  producer/
    build.gradle
```    

`settings.gradle`
```groovy
task action << {
    println("Consuming message: ${rootProject.producerMessage}")
}
```

`producer/build.gradle`
```groovy
task action << {
    println "Producing message:"
    rootProject.producerMessage = 'Watch the order of execution.'
}
```

执行`radle -q action`的结果
```
> gradle -q action
Consuming message: null
Producing message:
```

我们可以看到，没有执行成功。如果我们什么都没有定义的话，那么Gradle就会按照字典（a-z）来执行。所以:consumer:action 会在 :producer:action 前得到执行。下面用一个技巧去解决这个问题->通过重命名。

重新定义的项目结构
```
messages/
  settings.gradle
  aProducer/
    build.gradle
  consumer/
    build.gradle
```

`settings.gradle`
```groovy
include 'consumer', 'aProducer'
```

`aProducer/build.gradle`
```groovy
task action << {
    println "Producing message:"
    rootProject.producerMessage = 'Watch the order of execution.'
}
```

`consumer/build.gradle`
```groovy
task action << {
    println("Consuming message: ${rootProject.producerMessage}")
}
```

执行`gradle -q action`
```
> gradle -q action
Producing message:
Consuming message: Watch the order of execution.
```

现在我们先不理会这个技巧，渠道consumer这个目录执行命令
```
> gradle -q action
Consuming message: null
```
对于Gradle来说，两个task是没有什么关联的。如果你从根目录执行，他们两个都执行时因为他们有相同的名字并且在同一层次关系下。在上一个例子中只有一个task在层次关系中被找到，所以只有一个task得到了执行。我们需要一个比这个更好的技巧。

## 声明依赖
项目结构
```
messages/
  settings.gradle
  consumer/
    build.gradle
  producer/
    build.gradle
```

`settings.gradle`
```groovy
include 'consumer', 'producer'
```

`consumer/build.gradle`
```groovy
task action(dependsOn: ":producer:action") << {
    println("Consuming message: ${rootProject.producerMessage}")
}
```

`producer/build.gradle`
```groovy
task action << {
    println "Producing message:"
    rootProject.producerMessage = 'Watch the order of execution.'
}
```

执行`gradle -q action`的结果
```
task action << {
    println "Producing message:"
    rootProject.producerMessage = 'Watch the order of execution.'
}
```

在根项目中执行也可以得到同样的结果

## 跨项目task依赖最自然的用法
当然了，跨项目间，task的依赖并没有限制在名字必须相同上（像上一个例子一样，他们的task名师一样的）。让我们来改变task的名字看看。
`consumer/build.gradle`
```groovy
task consume(dependsOn: ':producer:produce') << {
    println("Consuming message: ${rootProject.producerMessage}")
}
```

`producer/build.gradle`
```groovy
task produce << {
    println "Producing message:"
    rootProject.producerMessage = 'Watch the order of execution.'
}
```

`gradle -q consume`执行的结果
```
> gradle -q consume
Producing message:
Consuming message: Watch the order of execution.
```

## 配置时依赖
让我们给producer添加一个属性，然后创建一个 **从consume到producer** 的配置时的依赖。
`consumer/build.gradle`
```groovy
message = rootProject.producerMessage

task consume << {
    println("Consuming message: " + message)
}
```

`producer/build.gradle`
```groovy
rootProject.producerMessage = 'Watch the order of evaluation.'
```

执行`gradle -q consume`结果
```
> gradle -q consume
Consuming message: null
```

默认的检查时间是基于字典的。所以consumer会在producer之前得到检查，但是此时producer还没有被检查，所以得不到我们想要的属性。Gradle给我吗提供了一下的解决方案：

`consumer/build.gradle`
```groovy
evaluationDependsOn(':producer')

message = rootProject.producerMessage

task consume << {
    println("Consuming message: " + message)
}
```
`gradle -q consume`的执行结果
```
> gradle -q consume
Consuming message: Watch the order of evaluation.
```

*evaluationDependsOn* 将会导致producer在consumer之前被检查
