sh的一些很有用但是易忽略的方面
================================

#	说明
这里说的易忽略，是由于我经常忘记或者一直依赖都不知道还有这茬=.=

感觉还是能够提示效率的，故在此记一记:-)

##	通配符
通配符不能匹配开头的``.``和文件夹分隔符``/``

1. `*`		零个或者多个连续的字符
2. `?`		单个字符
3. `[set]`	出现在set中的任何单个字符
4. `[^set]`	任何在set中**未**出现的单个字符
5. `[!set]`	同^

```sh
# 查看fib.c
$ ls -l f[aeiou]b.c
```

##	花括号
含花括号的命令可以扩展为多个参数，以逗号分隔

```sh
# 打印 hello.c hello.cpp hello.java
$ echo hello.{c,cpp,java}
```

##	输入/输出重定向
sh可以将标准输入stdin，标准输出stdout，标准错误输出stderr重定向为文件。

也就是说，任何命令都可以使用``<``将输入数据来源从stdin重定向到文件。

```sh
# 将cmd的标准输入重定向到file文件
$ cmd < file

# 将echo的输出从标准输出重定向到文件
$ echo 'string' > file
# 追加到文件末尾
$ echo 'append' >> file

# 将标准错误输出stderr重定向到文件并且标准输出打印在屏幕上
$ cmd 2> file

# 将stderr与stdout同时重定向到文件
$ cmd > out 2> errr
# 简化
$ cmd >& out_and_error
```

##	一些快捷键
``^ -> ctrl``	``% -> alt``

1. ^a	光标到行首
2. ^e	光标到行为
3. ^u	删除光标到行首的内容
4. ^k	删除光标到行尾的内容
5. ^w	删除光标前的一个单词
6. %d	删除光标后的一个单词
7. ^d	删除光标后的一个字符
8. ^h	删除光标前的一个字符
9. ^r	提供几个关键字，搜索历史命令并执行（重要）
10. %.	上一条命令的最后一个选项（esc+.同样效果）

### EOF
```yaml
background: /assets/images/xida.jpg
date: 2013-11-28T19:52:15+08:00
hide: false
location: Nanning
summary: ""
tags:
- Campus
- Linux
weather: ""
```
