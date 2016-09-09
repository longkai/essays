The Build Lifecycle
==================

## 构建的三个阶段
> 1. 初始化（Gradle支持多项目的构建，在这个阶段gradle会决定哪些项目将会被纳入构建，并且为每个构建实例化Instance对象）
> 2. 配置
> 3. 执行（执行配置阶段中的task）

## settings文件
文件名必须是settings.gradle，它会在初始化阶段被执行。一个多项目的构建必须有一个settings文件。 

## muilti-project需要记住的五点
> 1. allprojects
> 2. subprojects
> 3. evaluationDependsOn
> 4. evaluationDependsOnChildren
> 5. project lib dependcency

### EOF
```yaml
background: /assets/images/xida.jpg
date: 2013-08-07T22:12:46+08:00
hide: false
license: cc-40-by
location: Nanning
summary: ""
tags:
- Campus
- Gradle
weather: ""
```
