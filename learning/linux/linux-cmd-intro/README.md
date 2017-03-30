linux命令的基本说明
===================

# 任何的linux都是以一种特定这种方式使用

```sh
ls	stdin	stdout	-file	-- opt	--help	--version

ls	[options] [files]
```

其中``[]``表示可选，替换里面的内容即可

下面说明这六个选项的功能（若受支持）

## stdin
命令从标准输入读取数据（如键盘）

## stdout
命令将数据送到标准输出（如屏幕）

## -file
1. 当用'-'符号代替**输入**文件名时，表示命令将从标准输入读取数据
2. 当用'-'符号代替**输出**文件名时，表示命令将向标准输出写入数据

```sh
# wc命令分别从file1，标准输入，file2读取数据
$ wc file1 - file2
```

## -- opt
如果选项包括'--'，则意味着选项到此结束，其后命令行出现的任何部分都不能作为选项

```sh
# 查看'-file'文件的信息
$ ls -- -file

# 给ls命令添加file选项（实际上是不存在这个选项的）
$ ls -file
```

## --help
表示向该命令传递``--help``选项可以打印出其帮助信息

```sh
# 查看ls的帮助信息
$ ls --help
```

## --version
表示向该命令传递``--version``选项可以打印其版本信息

```sh
# 查看ls的版本
$ ls --version
```

## 注意事项
1. 以上某些情况下mac的终端并不适用，毕竟mac并不是linux
2. 以上的命令选项说明参考《linux口袋书（第二版）》，可能你需要查阅这本书才能查看某个命令是否支持上述的某些选项了:-)

### EOF
```yaml
background: /assets/images/default.jpg
hide: false
license: cc-40-by
location: Nanning
summary: ""
tags:
- Campus
- Linux
weather: ""
date: 2013-11-28T19:52:15+08:00
```
