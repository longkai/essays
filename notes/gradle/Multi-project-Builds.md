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
