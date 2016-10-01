新的域名，新的站点
===

#### Revamp
一直觉得，只有**.com**的域名才是最有价值的域名，加上去年[namecheap][1]送的一年**.me**域名快到期了，所以，想着想着就把**xiaolongtongxue.com**这个域名给买了下来，顺便看着有ssl的促销活动，也把站点配置成了**https**，看着地址栏绿色的锁，感觉很棒~

在学校的时候做过一段时间的前端，但是一直没有花太多的时间在这方面，尤其是js。所以之前的主页，也只是一个壳而已。但是，出于学习的目的，使用了[webcomponents][2]这项技术，但是，貌似目前就只有比较新的Chrome完全支持了吧，其他的浏览器都是用[polyfill][3]这个去兼容的，猜测应该是做了大量的hack。尝试了几个其他的浏览器，不是加载慢就是错乱的页面，不支持这项技术。出于这种原因，一直都想着改版，但是时间和技术上的（主要是对js的不熟悉），也就一直耽搁了。

前段时间，在[Hacker News][4]上看到一个新项目，用go写的静态网站生成器，叫做[Hugo][5]，由于我也使用go，看了一下感觉还不错，心里也琢磨着拿这个项目来作为新主页可以一战啊。

于是，大伙看到了现在的站点。

#### Todo
1. 现在感觉虽然比之前好看多了，但是在使用Hugo的过程中，发现一些缺陷(比如路径，本地图片资源，url定位，修改了markdown源文件等）这应该也和当前的版本0.15还不成熟有关系；Hugo生成的是静态站点，这没什么不好，但是static is static，有些东西缺少dynamic可能就没那么灵活了，比如说以后可能需要一个backend的实现呢？所以，找个时间自己实现一套backend还是有想象力的。

2. 目前前端的展现也是套用社区的theme。始终觉得自己怎么**都应该把前端学好**，到时候写一个自己的theme。

3. Github WebHook, 每当有内容push到Github，都应该是自动的把内容更新到站点。

#### One More Thing
之前有用过longist.me这个域名作为翻墙服务器的同学注意了，这个域名将在16年1月底到期，记得切换域名到xiaolongtongxue.com吧，其余配置照旧:)

#### EOF
```yaml
background: /assets/images/default.jpg
date: 2015-12-22T02:59:14+08:00
hide: false
license: cc-40-by
location: Shenzhen
summary: ""
tags:
- Web
weather: a little cold
```

[1]: www.namecheap.com
[2]: webcomponents.org
[3]: webcomponents.org/polyfills/
[4]: https://news.ycombinator.com
[5]: https://gohugo.io "Hugo :: A fast and modern static website engine"
