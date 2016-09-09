Gradle 依赖管理
===============

## Gradle依赖管理的最佳实践

> 1. 指明依赖的版本号，这样方便管理的同时也减少了潜在的错误。
> 2. 管理依赖的转移（不要任由工具处理依赖，应该适当的自己在第一级别的时候处理依赖，或者综合几种做法，总之，不要任由Gradle帮你处理）。
> 3. 解决同一依赖不同版本号带来的冲突
> 4. 使用动态版本号（比如4.+）和改变的模块（比如maven的snapshot），区别在于前者会得到一个实际的静态的版本号，而后者会是你得到的是你请求的版本号，但是底层的lib可能是变化的）

## Gradle依赖的类型
> 1. 外部模块的依赖（在某个外部仓库的模块）
> 2. 项目的依赖（在同一构建里的另一个项目中的依赖）
> 3. 文件依赖（在本地的文件系统里的一些文件）
> 4. 客户端模块的依赖（这个依赖在外部的某个仓库，但是在你本地使用时你想指定你的一些依赖属性，相当于重载了他）
> 5. gradle api 依赖（用在自定义gradle的插件）
> 6. local groovy 依赖（用于groovy）

### 下面来介绍一下各种依赖的用法
> 外部依赖（这是最常用的）

**build.gradle**
```groovy
depencies {
  runtime group: 'org.springframework', name: 'spring-core', version: '3.2.2.RELEASE'
  runtime 'org.springframework:spring-tx:3.2.2.RELEASE', 'org.hibernate:hibernate-core:4.2.0.Final'
  runtime(
    [group: 'org.springframework', name: 'spring-web', version: '3.2.2.RELEASE'],
    [group: 'org.springframework', name: 'spring-webmvc', version: '3.2.2.RELEASE']
  )
  runtime('org.hibernate:hibernate:3.0.5') {
    transitive = true
  }
  runtime group: 'org.springframework', name: 'spring-core', version: '3.2.2.RELEASE', transitive: true
  runtime(group: 'org.springframework', name: 'spring-core', version: '3.2.2.RELEASE') {
    transitive = true
  }
}
```
可以指定下载下来的依赖的类型（默认是jar），使用 **@** 符号
```groovy
dependencies {
  runtime 'org.groovy:groovy:2.0.5@jar'
  runtime gourp: 'org.groovy', name: 'groovy', version: '2.0.5', ext: 'jar'
}
```

> 客户端模块依赖（它可以使你声明依赖的转移，会取代外部仓库对该模块的描述，相当于你在本地重载了依赖关系）

```groovy
dependencies {
  runtime module('org.groovy:groovy-all:2.0.5) {
    dependency('commons-cli:commons:cli:1.0') {
      transitive = false
    }
    module(group: 'org.apache.ant', name: 'ant', version: '1.8.4') {
      dependencies 'org.apache.ant:ant-launcher:1.8.4@jar', 'org.apache.ant:ant-junit:1.8.4'
    }
  }
}
```
### 项目的依赖（多项目构建中）
```groovy
dependencies {
  compile (':shared')
}
```
### 文件的依赖（当某个lib无法从仓库获取或者你无法使用仓库时）
```groovy
dependencies {
  runtime files('libs/a.jar', 'libs/b.jar')
  runtime fileTree(dir: 'libs', include: '*.jar')
}
```

### 移除转移的依赖
```groovy
configurations {
  compile.exclude module: 'commons'
  all*.exclude group: 'org.gradle.test.excludes', module: 'reports'
  compile.exclude group: 'commons-logging', name: 'commons-logging'
}

dependencies {
  compile('org.gradle.test.excludes:api:1.0') {
    exclude module: 'shared'
  }
}
```

### 可选的属性
```groovy
List spring = [
                'org.springframework:spring-core:3.2.2.RELEASE@jar',
                'org.springframework:spring-aop:3.2.2.RELEASE@jar',
                'org.springframework:spring-beans:3.2.2.RELEASE@jar'
              ]
dependencies {
  runtime spring
}
```

### 依赖的配置（必须使用键值对来声明）
```groovy
dependencies {
  runtime group: 'org.springframework', name: 'spring-core', version: '3.2.2.RELEASE', ext: 'jar', configuration: 'someconfiguration'
  compile project(path: ':api', configuration: 'someconfiguration')
}
```

### 替换某个依赖的版本
```groovy
configurations.all {
  resolutinoStratege.eachDependency { DependencyResolveDetails details ->
  // 这里可以不用完全指明，只要能够辨识出来就好了
    if (details.requested.group == 'org.springframework' && details.requested.name == 'spring-core' && details.requested.version == '3.2.2.RELEASE') {
      details.useVersion '3.2.1.RELEASE'
    }
  }
}
```

### 在版本号处使用占位符
```groovy
configurations.all {
  resolutionStratege.eachDependency { DependencyResolveDetail details ->
    if (details.requested.version == 'default') {
      def version = findVersion(details.requested.group, details.requested.name)
      details.useVersion version
    }
  }
}

def findVersion(String group, String name) {
  // do some logic
  '3.2.3.RELEASE'
}

dependencies {
  compile 'org.springframework:spring-core:default'
}
```

### 替换某个依赖为一个兼容的依赖

```groovy
configurations.all {
  resolutionStrategy.eachDependency { DependencyResolveDetails details ->
    if (details.requested.name == 'commons-logging') {
      details.useTarget 'org.slf4j:jcl-over-slf4j:1.7.5'
    }
  }
}
```

### EOF
```yaml
background: /assets/images/xida.jpg
date: 2013-08-07T16:19:24+08:00
hide: false
license: cc-40-by
location: Nanning
summary: ""
tags:
- Campus
- Gradle
weather: ""
```
