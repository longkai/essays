The Java Plugin
===============

java plugin给项目添加了java的编译，测试和打包等功能。它同时也是很多其他plugin的依赖。

## 使用
要使用java plugin，只需要在你的`build.gradle`中添加如下一行
```groovy
apply plugin: 'java'
```
## Tasks
> 1. compileJava（使用javac编译src/main/java中的源代码）
> 2. processResources（将src/main/resources中的资源文件复制到主程序编译后的类路径下）
> 3. classes（compileJava+processResources）
> 4. compileTestJava（使用javac编译src/test/java总的源代码）
> 5. processTestResources （将src/test/resources中的资源文件复制到主程序编译后的类路径下）
> 6. testClasses（compileTestJava+processTestResources）
> 7. jar（将源代码打包成jar文件）
> 8. javadoc（使用javadoc生成项目的javadoc）
> 9. test（对项目进行单元测试，junit或者testng）
> 10. uploadArchives（上传项目生成的可执行文件到在configuration指定的目录）
> 11. clean（删除项目的build目录）
> 12. cleanTaskName（删除特定命令生产了中间文件以及最终文件，比如cleanTest将会删除掉test过程中生成的文件）

对于每一个添加到项目中的源代码目录，java plugin添加了如下的task
> 1. compileSourceSetJava（用javac编译给定的源代码目录）
> 2. processSourceSetResources（复制资源文件到生成的主类目录）
> 3. sourceSetClasses（前面两个相加）

此外，java plugin也添加了构建生命周期的task
> 1. assemble（编译整个项目）
> 2. check（检查项目）
> 3. build （assenble+check）
> 4. buildNeeded（对项目和此项目所有的依赖执行完成的构建）
> 5. buildDependents（对项目和对于本项目有依赖的项目执行完整的构建）
> 6. buildConfigurationName（使用特定的配置对项目进行构建）
> 7. uploadConfigurationName（使用特定的配置上传项目）

下面是想各种tasks间的关系
> ![taskrelationship](https://docs.gradle.org/current/userguide/img/javaPluginTasks.png)

## java plugin 的一些属性
> 1. sourceSets 包括了项目的源代码
> 2. sourceCompatibility 项目的兼容性
> 3. targetCompatibility 项目编译时使用的jdk
> 4. archivesBaseName 生成的项目名，默认是项目名.jar(.zip)等
> 5. manifest jar文件里包含的manifest文件

### EOF
```yaml
background: //docs.gradle.org/current/userguide/img/javaPluginTasks.png
date: 2013-08-08T16:09:38+08:00
hide: false
license: cc-40-by
location: Nanning
summary: ""
tags:
- Campus
- Gradle
weather: ""
```
