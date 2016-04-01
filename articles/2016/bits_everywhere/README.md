位操作在 Android 源码中的运用
===

> 设计者确定其设计已经达到了完美的标准不是不能再增加任何东西，而是不能再减少任何东西。

> -- Antoine de Saint-Exupéry 法国作家兼飞机设计师

《[编程珠玑][1]》开篇第一章描述了一个非常经典的问题：

**输入**：一个最多包含 n 个正整数的文件，每个数都小于 n，其中 n=10^7。如果在输入文件中有任何整数重复出现就是致命错误。没有其他数据与该整数相关联。

**输出**：按升序排列的输入整数列表。

**约束**：最多有（大约）1MB 的内存空间可用，有充足的磁盘存储空间可用。运行时间最多几分钟，运行时间为几秒钟就不需要进一步优化了。

请大伙花些时间思考一下该问题，知道问题的朋友们也请回顾一下，文章最后会给出参考答案。

## 位操作与集合
位操作我记得在大学的组成原理上讲过，这里直接引用[维基百科的介绍][2]：

> 位操作是程序设计中对位模式按位或二进制数的一元和二元操作。在许多古老的微处理器上，位运算比加减运算略快，通常位运算比乘除法运算要快很多。在现代架构中，情况并非如此：位运算的运算速度通常与加法运算相同（仍然快于乘法运算）。

常见的位运算有以下几种：

运算符 | 操作含义
-------|---------
``&``  | 与
``|``  | 或
``^``  | 异或/反(一个操作数时，大多数语言使用``~``)
``&^`` | 与非
``<<`` | 左移
``>>`` | 右移

下面通过一个栗子来回顾一下这几个操作，顺带引入**位集**的概念。

```go
// x, y, z are uint8
x = 1 << 1 | 1 << 5 // x = 00100010, set {1, 5}
y = 1 << 1 | 1 << 2 // y = 00000110, set {1, 2}

z = x & y           // z = 00000010, intersection {1}
z = x | y           // z = 00100110, union {1, 2, 5}
z = x ^ y           // z = 00100100, symmetric difference {2, 5}
z = x &^ y          // z = 00100000, difference {5}

z = x << 1          // z = 01000100, set {2, 6}
z = x >> 1          // z = 00010001, set {0, 4}
```

x, y, z 均为8位的无符号整数，这里的位集的概念可能大伙已经看出来了，也就是二进制数从低位到高位不为0的位组成的集合。而``&``，``|``, ``^``, ``&^`` 分别表示两个集合A，B的**交集**，**并集**，**对称差集**，**差集**；而``<<``，``>>``可以理解为对集合中的元素加1或减1，当元素超出了集合的范围时则将该元素从集合中移除。

## Mask 掩码
在网络原理中，我们知道，为了避免 ip 地址的浪费，而又有多个终端需要使用网络，每个终端都需要设定 ip 地址，这时便需要划分子网。

举个栗子，在家里我们给电信交钱办理了一个宽带，那么上网的时候电信会给我们分配一个 ip，然而我们有很多终端都要同时上网，比如笔记本，手机，平板等，这时便通过划分子网，由 DHCP 给每一个终端设置一个子网 ip （或者自己手动设置静态 ip），所有的 ip 都会跟随一个**子网掩码**，通过他可以算出这个 ip 所在的网络。

实践一下，查看此时笔记本的网络信息，得到 ip ``192.168.3.6``，子网掩码 ``0xffffff00(255.255.255.0)``，通过``&``操作

```
  1100 0000 1010 1000 0000 0011 0000 0110 (192.168.3.6)
& 1111 1111 1111 1111 1111 1111 0000 0000 (255.255.255.0)
-----------------------------------------
  1100 0000 1010 1000 0000 0011 0000 0000 (192.168.3.0)
```

最终算出来的网络地址是``192.168.3.0``，同样的方式，可以算出连接同一个路由器的手机或者 pad 的网络地址都是一样的。事实上，网络地址和子网掩码确定了一个子网范围，如``192.168.3.1 - 192.168.3.254``便是我现在所在网络的子网范围了。因为掩码与这个范围的任意 ip 地址``&``操作所得的结果都一样（也就是网络地址相同）。

这就是掩码（mask）的作用，**屏蔽不相关的位，保留感兴趣的位**。在以上的栗子中，屏蔽了主机位，保留了网络地址位。

Android 源码中大量使用了掩码。

## View.MeasureSpec
写过自定义控件的朋友对这个类一定不会陌生。Android 任意的 view 绘制到屏幕上都会经历 measure -> layout -> draw 的过程，measure 过程通过重载``View.onMeasure(int widthMeasureSpec, int heightMeasureSpec)``来测量自身的宽高。如何仅仅通过一个整型值``widthMeasureSpec``就能确定宽度呢？我们来看看源码。

```java
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

/**
 * Measure specification mode: The parent has not imposed any constraint
 * on the child. It can be whatever size it wants.
 */
public static final int UNSPECIFIED = 0 << MODE_SHIFT;

/**
 * Measure specification mode: The parent has determined an exact size
 * for the child. The child is going to be given those bounds regardless
 * of how big it wants to be.
 */
public static final int EXACTLY     = 1 << MODE_SHIFT;

/**
 * Measure specification mode: The child can be as large as it wants up
 * to the specified size.
 */
public static final int AT_MOST     = 2 << MODE_SHIFT;

/** Creates a measure specification based on the supplied size and mode. */
public static int makeMeasureSpec(int size, int mode) {
  if (sUseBrokenMakeMeasureSpec) {
    return size + mode;
  } else {
    return (size & ~MODE_MASK) | (mode & MODE_MASK);
  }
}

/** Extracts the mode from the supplied measure specification. */
public static int getMode(int measureSpec) {
  return (measureSpec & MODE_MASK);
}

/** Extracts the size from the supplied measure specification. */
public static int getSize(int measureSpec) {
  return (measureSpec & ~MODE_MASK);
}
```

通过``MODE_SHIFT``将32位的整数分成了2个部分，前2位为 mode，也就是``UNSPECIFIED``，``EXACTLY``，``AT_MOST``之一，后30位为 size（用来表示宽高足够足够大了......），而掩码``MODE_MASK``便是将 mode 和 size 分解和合成的关键。

举个栗子，这里**不讨论 view 测量的细节**，只考虑 MeasureSpec 本身。我们在 layout xml 中定义一个``TextView``，那么程序执行到这个 view 的``onMeasure(...)``方法时，他的``widthMeasureSpec``和``heightMeasureSpec``怎么形成的呢？

```xml
<TextView
  layout_width="255px"
  layout_height="wrap_content" />
```

以宽为栗，此时他的 size 为 255，mode 为``EXACTLY``也就是``1 << MODE_SHIFT``，那么``widthMeasureSpec``可以通过``255 | mode``得到，也就是 size 与 mode 的位集取并集，将两者合并在一起。源码中``size & ~MODE_MASK``是为了彻底屏蔽最高的2位，同理``mode & MODE_MASK``屏蔽后30位，这样程序更 robust。最终，``widthMeasureSpec``如下

```
  0000 0000 0000 0000 0000 0000 1111 1111 {0,1,2,3,4,5,6,7}
| 0100 0000 0000 0000 0000 0000 0000 0000 {30}
-----------------------------------------
  0100 0000 0000 0000 0000 0000 1111 1111 {0,1,2,3,4,5,6,7,30}
```

那么问题来了，在``onMeasure(...)``时我们通常都需要拿到具体的 mode 和 size 来进行下一步计算，通过调用``MeasureSpec.getMode(int)``和``MeasureSpec.getSize(int)``即可实现，实现方式也是使用掩码，具体的相信你已经知道怎么算了，这里留给朋友们思考。

## View.getVisibility() | View.setVisibility(int)
这两个方法相信每个 Android 开发者都再熟悉不过了。尽管在我们的代码中是对3个整形常量（visible, invisible, gone）做比对，然而 Android 源码中的实现可是不会单独的用一个变量去记录 view 的 visibility 状态的（然而如果我们自己则很可能会实现前者）。事实上，**view 自身绝大部分状态变量都记录在一个 int 值里面**，让我们看看源码：

```java
/** The view flags hold various views states. */
int mViewFlags;

static final int VISIBILITY_MASK = 0x0000000C;
public int getVisibility() { return mViewFlags & VISIBILITY_MASK; }

static final int ENABLED_MASK = 0x00000020;
public boolean isEnabled() { return (mViewFlags & ENABLED_MASK) == ENABLED; }

static final int CLICKABLE = 0x00004000;
public boolean isClickable() { return (mViewFlags & CLICKABLE) == CLICKABLE; }

static final int WILL_NOT_DRAW = 0x00000080;
public boolean willNotCacheDrawing() { return (mViewFlags & WILL_NOT_CACHE_DRAWING) == WILL_NOT_CACHE_DRAWING; }

// ...
```

以上获取状态的方法都是通过掩码来实现，那么设置状态呢？在 view 里有个``setFlags(int flags, int mask)``的包方法，方法比较长，简化之后其实也很好理解：

```java
void setFlags(int flags, int mask) {
  // ...
  int old = mViewFlags;
  mViewFlags = (mViewFlags & ~mask) | (flags & mask);

  int changed = mViewFlags ^ old;
  if (changed == 0) {
    return;
  }

  final int newVisibility = flags & VISIBILITY_MASK;
  if (newVisibility == VISIBLE) {
    // ...
  }

  /* Check if the GONE bit has changed */
  if ((changed & GONE) != 0) {
    // ...
  }

  /* Check if the VISIBLE bit has changed */
  if ((changed & INVISIBLE) != 0) {
    // ...
  }

  if ((changed & VISIBILITY_MASK) != 0) {
    // If the view is invisible, cleanup its display list to free up resources
    // ...
  }
  // ...
}
```

改变 view 的状态（比如调用``setVisibility(...)``）最终都会调用到``setFlags(int, int)``，``(mViewFlags & ~mask) | (flags & mask)``表示在保证其他的状态位不变的情况下与目标状态的并集，接下来通过比对现有状态和之前的状态的变化，对 view 的行为做出调整（如重绘，回调各种状态的 listener 等）

从这里我们也看出了使用位操作的好处，**避免定义大量的变量**，而是通过一个变量与掩码（常量）做位操作，节约内存的同时，也降低了 bug（维护 N 个变量与维护1个变量的差别）出现的可能。

换一种理解方式。假设有2个状态，我们很可能会定义两个变量表示；可以想象，系统变得越来越复杂，我们添加变量也会越来越多，这时我们很可能定义数组或列表或其他数据结构（索引为状态位）来表示这些变量。这实际上和位集是一个意思（掩码可以理解成索引），只不过位集占用内存小(至少1/32，本节的栗子)，运算速度快。

结论就是：**位集也是数据结构**。

## Bits Everywhere
实际上，在 Android 中位集无处不在，如``MotionEvent``中的 action，``ViewGroup.mGroupFlags``各种状态，如``FLAG_DISALLOW_INTERCEPT``, ``FLAG_INVALIDATE_REQUIRED``, ``Gravity``里的各种位置, ``Intent.setFlags(...)``各种标记位等等......

说到底，整个世界不是1与0的集合么？

## 答案
```c
#include <stdio.h>

#define BITSPERWORD 32
#define SHIFT 5
#define MASK 0x1f
#define N 10000000
#define a[1 + N / BITSPERWORD]

void set(int i) { a[i >> SHIFT] |= (1 << (i & MASK)); }

void clr(int i) { a[i >> SHIFT] &= ~(1 << (i & MASK)); }

int test(int i) { return a[i >> SHIFT] & (1 << (i & MASK)); }

int main() {
  int i;
  for (i = 0; i < N; i++) {
    clr(i);
  }

  while (scanf("%d", &i) != EOF) {
    set(i);
  }

  for (i = 0; i < N; i++) {
    if (test(i)) {
      printf("%d\n", i);
    }
  }

  return 0;
}
```

## 致谢
感谢@jsy @Liger 同学一同在深夜讨论网络原理以及对本文章的审阅。文章难免有错误和不当之处，欢迎提出指正。

今天开始跑步了，坚持。

## 参考
- 《编程珠玑》
- 《The Go Programming Language》
- 图片来自[signalvnoise][3]

## EOF
```json
{
  "tags": ["Android"],
  "render_option": 0,
  "date": "2016-03-31T22:37:55+08:00",
  "weather": "okay day",
  "summary": "编程珠玑 -> 位操作 -> 位集 -> 网络原理 -> Android 源码, bits everywhere!",
  "location": "Shenzhen",
  "background": "go101010.jpg"
}
```

[1]: https://book.douban.com/subject/3227098/
[2]: https://zh.wikipedia.org/wiki/%E4%BD%8D%E6%93%8D%E4%BD%9C
[3]: https://signalvnoise.com/posts/3897-go-at-basecamp
