git初探
=======

## 新建git repository
```sh
$ mkdir ~/public_html
$ cd ~/public_html
$ echo 'My website is alive!' > index.html
$ git init
Initialized empty Git repository in /Users/longkai/public_html/.git/
```
执行后生成的隐藏``.git/``目录就是git维护的你的这个repository的整个版本库

你的``pulic_html/``被称为**工作目录**

## 向git repository中添加文件
使用``git add (file|files|dirs)``向repository中添加文件

一次可以添加一个或者多个文件，也可以添加一个目录
```sh
$ git add index.html 
```
命令执行成功后，git知道了``index.html``将要保留在repository中，但是git只是把``index.html``暂存了起来，作为在下一次提交之前的一个临时动作。

git将add和commit划分为两个过程是为了保持repository的稳定性，深思熟虑后的提交不至于repository太乱:-)

接下来执行``git status``来查看工作目录和版本库之间的状态
```sh
$ git status
# On branch master
#
# Initial commit
#
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#
#       new file:   index.html
#
```
运行的结果表示新文件``index.html``文件将在下一次提交后添加到repository中

git repository除了文件内容的变化外，git还会记录每次提交的元数据，包括提交log和作者
```sh
$ git commit -m "Initial contents of public_html"
[master (root-commit) 0d3625d] Initial contents of public_html
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
```
这里简单的使用了一句话的log，实际中你可以使用你自己喜欢的文本编辑器记录更详细的log，比如vim，如果默认不是vim，那么在bash中``export GIT_EDITOR=vim``

接下来我们再来看看工作目录和reposiroty之间的状态
```sh
$ git status
# On branch master
nothing to commit, working directory clean
```
看，clean working directory，这意味着工作目录现在没有修改，添加，删除的文件或内容啦！

## 配置提交的作者
刚才的提交我没有指名作者，因为我已经配置了默认的作者:-)
```sh
$ git config --global user.name "longkai"
$ git config --global user.email "im.longkai@gmail.com"
```

## 创建一个新的提交
```sh
$ cd ~/public_html
$ vim index.html
# edit file
$ cat index.html
<html>
	<body>
		welcome to my size!
    </body>
</html>
```
接下来直接提交
```sh
$ git commit index.html
# open vim to add log
$ git status
# On branch master
nothing to commit, working directory clean
```
也许你会觉得奇怪，为什么不是先add再commit呢？如果熟悉git后发现其实不是这样的，因为这个文件再提交之前已经假如到repository了，git已经认识她啦！

现在，我们已经有了2个提交

## 查看我的提交
一旦你有了提交后你便可以查阅你的提交

命令``git log``将整个提交列表倒序打印出来
```sh
$ git log
commit fcf1a7f60148aabf9c52a8a8869f9dde50fdf95e
Author: longkai <im.longkai@gmail.com>
Date:   Thu Nov 28 23:55:02 2013 +0800

    convert to html

commit 0d3625d1b212debc3e683928c791cc7cf9e6181d
Author: longkai <im.longkai@gmail.com>
Date:   Thu Nov 28 23:40:06 2013 +0800

    Initial contents of public_html
```
以上只是查看了提交的ID（如fcf1a7f60148aabf9c52a8a8869f9dde50fdf95e），作者，日期和log，想要进一步查看内容的变更，使用``git show``命令
```sh
$ git show fcf1a7f60148aabf9c52a8a8869f9dde50fdf95e
commit fcf1a7f60148aabf9c52a8a8869f9dde50fdf95e
Author: longkai <im.longkai@gmail.com>
Date:   Thu Nov 28 23:55:02 2013 +0800

    convert to html

diff --git a/index.html b/index.html
index 34217e9..8a9dac1 100644
--- a/index.html
+++ b/index.html
@@ -1 +1,5 @@
-My website is alive!
+<html>
+       <body>
+               welcome to my size!
+       </body>
+</html>
```
不带commit id则表示最近的一次提交。很酷，是吧！

另外，命令``git show-branch``则提供了更简洁的展现
```sh
$ git show-branch --more=7
[master] convert to html
[master^] Initial contents of public_html
```
由于我们只有2次提交，所以只显示了2条，不带任何参数则表示最近的一次提交

## 查看提交间的差异
想要查看任意两个提交之间的差异？用``git diff commit_id1 commit_id2``
```sh
$ git diff fcf1a7 0d3625
diff --git a/index.html b/index.html
index 8a9dac1..34217e9 100644
--- a/index.html
+++ b/index.html
@@ -1,5 +1 @@
-<html>
-       <body>
-               welcome to my size!
-       </body>
-</html>
+My website is alive!
```
可以看到，commit_id可以是开头的几个字母的简写~

## 在repository中删除和重命名文件
这个和普通的命很像，只是加上了``git``命令前缀

假设你的repository中有一个test.txt文件不想要他了
```sh
$ git rm test.txt 
rm 'test.txt'
$ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#       deleted:    test.txt
#
$ git commit -m "remove test.txt file"
[master daca0eb] remove test.txt file
 1 file changed, 1 deletion(-)
 delete mode 100644 test.txt
```
可以看到，执行``git rm test.txt``命令后，test.txt文件被删除到了暂存区，下一次提交就将被删除，这个和add非常像

假设你有一个``foo.txt``的文件，想要重命名为``bar.txt``
```sh
$ git mv foo.txt bar.txt
$ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#       renamed:    foo.txt -> bar.txt
#
$ git commit -m "rename foo.txt to bar.txt"
[master 2e1ca23] rename foo.txt to bar.txt
 1 file changed, 0 insertions(+), 0 
```
OK，就是这么简单！

## 复制你的repository
使用``git clone``命令能将一个完整的版本库复制，包裹所有的提交，变更，完全一样。

如果是从web上clone，那么还会有一些额外的信息复制下来，方便追踪与增加提交等，这里就先不涉及啦。
```sh
$ cd ~
clone public_html/ public_html2
Cloning into 'public_html2'...
done.
Checking connectivity... done
```

## 配置文件
git的配置文件是继承式的，下面分别从优先级高到低说明

>
```
.git/config
	repository相关的配置，可以使用--file选项对其进行操作（默认操作也是如此）
~/.gitconfig
	用户相关的配置，可以使用--global选项对其进行操作
/etc/gitconfig
	系统相关的配置，可以使用--system选项对其进行配置（如果你有足够的权限），另外，该文件可能根据操作系统的不同放在不同的目录
```	
>

可以使用``git config -l``命令查看当前适用的配置
```sh
$ git config -l
user.name=longkai
user.email=im.longkai@gmail.com
color.diff=auto
color.ui=true
alias.lg=log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
alias.st=status
alias.br=branch
alias.ci=commit
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
core.ignorecase=true
core.precomposeunicode=false
```
当然，你也可以直接查看``.git/config``文件的内容
```sh
$ cat .git/config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
        ignorecase = true
        precomposeunicode = false
```
发现怎么没有之前show的那么长呢？其实它是继承了用户配置的，使用``git config -l --global``查看即可！

使用``git config --unset``命令删除配置，当然，你也可以使用文本编辑器直接编辑配置文件！
```sh
$ git config --unset --global user.mail
```

## 配置别名
命令太长？别担心，正如shell提供的别名一样，你也可以给命令添加别名呢！
```sh
$ git status
# On branch master
nothing to commit, working directory clean
$ git config alias.s "status"
$ git s
# On branch master
nothing to commit, working directory clean
```
可以看到，我们使用``git s``完成了``git status``的命令，另外，我们使用的是项目相关的配置哦~

## 说明
本笔记参考自《version control with git 2nd》，感谢原作者！
