go lang 初探
============

## 安装
请参考官方文档或者google

## 设置GOPATH变量
### GOPATH变量是指自己的代码库的目录。go约定所有的代码都必须按照开源代码的目录放置，不管你是否开源还是不开源。位置是任意的，假设我们把代码库放到``~/src/go``下，那么就把``export GOPATH=~/src/go``即可，当然你也可以有多个src目录,需要的时候改变一下GOPATH就行啦。

### 接下来，看看go约定的目录结构
```
$GOPATH
	src/
	bin/
	pkg/
```
其中，``src``指的是源代码存放目录，``bin``表示go构建的可执行文件目，``pgk``就有点像第三方或者自己的依赖（像maven仓库）

### 建立自己的源码目录
go要求开源代码的目录放置，那么假设你有一个github帐号，那么``github.com/userid``就是你的源码根路径，加起来就是``$GOPATH/src/github.com/longkai/``

## 构建第一个go项目
### 建立项目目录
假设项目名叫hello，那么
```sh
# 进入你的源码目录，建立项目
$ cd $GOPATH/src/github.com/userid
$ mkdir hello
$ vim hello.go
# edit file
```
最终的hello.go内容
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

接下来，你可以在hello目录下键入``go install``或者任意目录``go install github.com/userid/hello``。如果shell没有任何输出，就表示执行成功了。接下来，在``$GOPATH/bin``目录下后生成hello的可执行文件，为了方便以后的工作，强烈建议把``$GOPATH/bin``假如到环境变量中！执行``export PATH=$PATH:$GOPATH/bin``即可！

运行hello程序，就可以看到熟悉的东西啦！

## 创建自己的package
package有点像java的package，是用来组织自己的源代码的。但是区别还是有的。go的package不要求和java那样必须和目录一致，也就是说像``/path/to/package``下的go源码文件，你的package name可以和package不一样(但是最好一样吧，比较好管理)，但是同一个目录下的所有go源码的package必须一样（不包含子目录）。另外，main是一个特殊的package name，他表示生成的是可执行文件。

那么接下来我们以``go-test``目录新建一个lib

```sh
$ mkdir -p $GOPATH/src/github.com/userid/go-test
$ vim cal.go
# edit file
```
最终的文件
```go
package cal

func Add(a, b int) int {
        return a + b
}

func Minus(a, b int) int {
        return a - b
}

func Times(a, b int) int {
        return a * b
}

func Divide(a, b int) int {
        return a / b
}
```
测试以下~
我们的测试文件，``cal_test.go``，名字也是约定必须以xx_test.go的字样！
```go
package cal

import "testing"

func TestAdd(t *testing.T) {
        const a, b, result = 1, 2, 3
        if value := Add(a, b); value != result {
                t.Errorf("error!")
        }
}

func TestMinus(t *testing.T) {
        const a, b, result = 1, 2, -1
        if value := Minus(a, b); value != result {
                t.Errorf("error!")
        }
}

func TestTimes(t *testing.T) {
        const a, b, result = 1, 2, 2
        if value := Times(a, b); value != result {
                t.Errorf("error!")
        }

func TestDivide(t *testing.T) {
        const a, b, result = 4, 2, 2
        if value := Divide(a, b); value != result {
                t.Errorf("error!")
        }
}
```

测试的func名也是约定的Test_FuncName()，接下来，与前面install类似，执行go test即可看到是否通过测试！

如果通过了测试，接下来再``go install``，没有输出任何信息的话就标识ok啦，在``$GOPATH/pkg/``下应该能找到和你的系统相关的lib了

## 调用刚才的lib
```sh
# vim $GOPATH/src/github.com/userid/hello/hello.go
# edit file
```
最终的内容，这里我们没有使用和目录名意义的package name，其实是不太好的= =
```go
package main

import (
	"fmt"
	"github.com/go-test"
)

func main() {
	fmt.Printf("3+2=%d\n", cal.Add(3, 2))
}
```

接下来，依照之前的构建方式``go install``，没有任何输出的话，运行hello就好了

**另外，package下面还可以有子package，也是和刚才的类似的

## 从远程获取lib
git的源代码目录非常的好处就是有利于整个go源代码的管理，你所有用到的代码都统一放到了``$GOPATH``下，比如你需要使用到``example.com/xxx``的源码，那么相应的目录就是``$GOPATH/src/example.com/xxx``，尤其适合git，hg这种版本控制系统！！！

用``go get ecamle.com/xxx/project``可以自动的下载远程版本库到你的$GOPATH中，很方便，不过前提是你得装有git，hg这些软件才行

## 抓取远程lib到本地进行构建
已刚才的hello项目为例，该项目已经push到了远程的github版本库。

首先，``go get github.com/longkai/go-test``，那么go-test这个源码会自动下载到你的本地，如果已经存在了的话就不用下载了，然后重新install刚才的hello项目即可！

## 说明
1. 这个笔记参考的是官方的文档http://golang.org/doc/code.html，强烈大家看看这个文档！这里不详细。
2. 欢迎讨论，我的邮箱是im.longkai@gmail.com

### EOF
```json
{
  "tags": ["Campus", "Golang"],
  "reserved": false,
  "date": "2013-12-05T18:18:07+08:00",
  "weather": "",
  "summary": "",
  "location": "Nanning",
  "background": "/assets/images/xida.jpg"
}
```
