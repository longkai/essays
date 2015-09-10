再学vi
===
## 吐槽&引子
昨天去了东部华侨城的大侠谷玩，people mountain people sea，再一次观赏了这一悲剧。到处都是人啊车的，围的水泄不通，一个木质过山车的游乐项目，从山上排到了山下，目测有2，300米长，还是并排的。一个野战cs，看着通告牌上写着「预计等待时间70分钟」，根本就没有玩的心思了。走在路上，听到有人说，我今天一个项目都没玩过，说的太好了。这种时间来这种地方，真是醉了。长长的队伍中，时不时就出现插队的人，真的很讨厌，人太多了，安保人员无暇顾及，你跟他说，请不要插队，他根本不理你，还乐呵呵的招手示意同伴赶紧过来，我这里有位置。呵呵，打死插队🐶。。理性告诉我们，要忍耐，可是谁来维护公正公平呢？当年小贝被侵犯怒了，直接将对手踢倒，自己吃到了红牌被罚下场，导致对方多打一人最后落败，后来被媒体口诛笔伐然而对手却「逍遥法外」。联想到现实社会上那些种种，只是觉得，骚年，你想太多了。不去评价别人，做自己觉得对的事，就好了。

回到市区后，直接去了深圳图书馆，把网上预借的那本书给借了。这里得赞一下深图，网上预约成功后，可以选择取书地点（包括24小时的社区图书馆），成功预借后还有短信提醒，cool。

那本书叫做《Learning the vi and Vim Editors》。

## vi
说起来，我也算是vi的老用户了，除了IDE和office（先在vi上写好纯文本在贴到office增加样式）其他的文本编辑几乎全都使用vi。大二寒假刚开始学习Linux的时候，看的是鸟哥的那本经典的书，里面就谈到了vi的种种好处，于是就下载了saltiger同学慷慨提供了原版电子书学习（学生时代穷啊...），话说我最喜欢原版的动物书了！看了part 1，练习了几天就开始使用vi直到现在。期间接触过其他的编辑器或者IDE，比如Sublime Text，Atom，Eclipse，Intellij IDEA都在寻找并使用相关的vi模式插件，甚至是bash和chrome都有相应的vi模式。由于自己主要是面向J2EE，Android相关的开发，离不开IDE，但是编辑器中总是是少不了vi模式，vi在编辑文本上极大的提升了效率。最近开始写golang程序了，配置vim相关的插件，感觉在vim上写程序比起在IDE上能够思考得更多。另外，省电啊，不卡啊，怎么着也节能一些吧，一直很想吐槽java，chrome，android这些，虽然看起来好像makes your life easier(both for programming and user-experence)，但是从能耗的方面来说，是不及格的。写程序高效一些，节能一些，也是一种境界啊。以前看到一个Google的工程师的status，大致是「I write less entropy code」，cool。

## 读书笔记
借这本书的意图主要是想更深入的学习vim的一些高级知识以及未曾了解的东西，简而言之，「温故而知新」。今天看了part 1的前三节，发现自己当初也仅仅是看了这三节的内容，不少东西都忘了以至现在连正则查找的命令都含糊了。正好，在这里写一写读书笔记吧，看看能坚持多久...

### 1. The vi Text Editor
1. ``ZZ``   like ``:wq``, quit vi, saving edits.

### 2. Simple Editing
1.  ``W``    move the cursor forward one word at a time, not counting syumbols and punctuation.
2.  ``cc``   like ``S``, change entire current line.
3.  ``C``    like ``c$``, change the characters from the current cursor position to the end of the line.
4.  ``R``    replace character for the current line until hitting **ESC**.
5.  ``~``    toggle character case.
6.  ``D``    like ``d$``, delete from the cursor position to the end of the line.
7.  ``.``    repeat command.
8.  ``U``    undoes all edits on a single line, **as long as the cursor remains on that line**. (todo: it is weird if you use twice repeatly)
9.  ``8i*``  numberic args for insert command, i.e. insert 8 ``*`` before the cursor position.
10. ``J``   join two lines.

### 3. Moving Around in a Hurry
1. ``z ENTER``  move cursor line to top of screen and screen.
2. ``z .``      move cursor line to middle of screen and screen.
3. ``z -``      move cursor line to bottom of screen and screen.
4. ``^L``       redrawing the screen.
5. ``H``        move to home -- the top line on screen.
6. ``M``        move to middle line on screen.
7. ``L``        move to last line on screen.
8. ``ENTER``    move to the first character of next line.
9. ``+``        move to the first character of next line.
10. ``-``       move to the next character of next line.
11. ``^``       move to the first nonblank character of current line.
12. ``e``       move to the end of word.
13. ``E``       move to the end of word (ingore punctuation).
14. ``(``       move to beginning of current sentence.
15. ``)``       move to beginning of sentence.
16. ``{``       move to beginning of current paragraph.
17. ``}``       move to beginning of next paragraph.
18. ``[[``      move to beginning of current section.
19. ``]]``      move to beginning of next section.
20. ``fx``      find the cursor to the **next** instance of the characters __x__ of current line.
21. ``Fx``      find the cursor to the **previous** instance of the characters __x__ of current line.
22. ``tx``      find character **before** next occurrence of __x__ in the line.
23. ``Tx``      find character **after** next occurrence of __x__ in the line.
24. ``;``       repeat previous find command in same direction.
25. ``,``       repeat previous find command in opposite direction.
26. ``''``      return to beginning of line containing previous mark.
27. <code>``</code>   return to previous mark or context.


## one more thing
今天又一个大学舍友来深圳发展了，明天另一个大学舍友将外派到北京出差。friends come and go, live your life, wish us all the best.
