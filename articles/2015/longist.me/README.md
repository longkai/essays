我的个人站点上线啦
===

我的个人站点上线了，网址是http://longist.me, 欢迎逛逛~

说来惭愧，本来今年年前就配置好了域名和Digital Ocean的服务器，想捣鼓一个啥啥啥出来玩玩的。但是迟迟都没有动手。一来自己还是懒，总以为自己还没准备好，但是准备好了再上路是有多难，甚至胎死腹中；二来也许是自己还是有点忙吧，下班已经挺晚了，不如以往爱折腾了吧。

不过，好在借着上周末，趁着欧冠决赛，突然心血来潮，花了点时间还是折腾出了一个网站，其实也就是一个界面而已。过程并不简单，感觉和一年前写web差了好多，好着力。一方面，自己也不想沿用以前的那套（jquery + bootstrap + j2ee），老是在做一样的东西没有什么长进。并且，写server的时候用java，写Android的时候还是用java，累觉不爱。做一件小小的事情还要很多代码，那确实说明这语言有缺陷或者用在了不合适的地方。还有js，感觉现在的web发展好快啊（也许和自己做得少有关吧）。貌似现在浏览器原生的js就已经很厉害了，不像那个时候在学校，还要兼容IE6...哈哈哈。

对我来说，这个项目的技术栈几乎是全新的。backend使用了Golang，这个语言虽然在大学就玩过了，但是没有在实际中写过它以至于很多东西都是不清不楚似懂非懂的，还得再系统的学学，正好借这个项目使使。frontend使用了Polymer，她的slogan是「Leverage the future of the web platform today」，确实挺酷的，前端开发简直...用过就知道。在刚刚结束的Google IO 2015中大版本升级到1.0了，好棒。

至于为什么使用这个两个，估计是很早就关注这两个项目了吧。不说这么多技术上的东西了，简单说一下网站的roadmap吧。

1. 静态站点，主要是展现article列表，然后redirect到github的markdown渲染。所以，基本上是没有做什么，但是至少是有个网站了，对吧...同时，还可以搭配微信的公众号再做一次redirect到github的markdown。 I love Github flavored markdown.

2. 利用Developer Program Member，通过github的webhook做钩子，hook到backend，每当有git push到github，便在backend更新内容。考虑直接把rendered的markdown持久化到backend，不redirect到github。

3. Android and iOS native app.

4. featrues, experiment, imagine, etc.

*I have been keeping learning something really cool and new while making it better and better.*

已经很久没有更新东西了，一直在走，似乎忘了什么。其实我还是想写东西，做一个手艺人，在vim上写最纯粹的文字。

--
2015-06-08

### EOF
```yaml
background: ""
date: 2015-06-08T23:19:48+08:00
hide: false
license: cc-40-by
location: Shenzhen
summary: ' 谨以此纪念原来的域名-.-'
tags:
- Web
weather: ""
```
