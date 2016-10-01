从 Go 一窥 Java 函数式编程的演变
===

#### Before the Start
今天翻看Brian W. Kernighan所著的《[The Go Programming Language][1]》，发现一个比较有趣的问题：

有如下的计算机学科课程（一个 key 为课程，value 该课程的先修课程的 map 数据结构）

```go
var prereqs = map[string][]string{
	"algorithms": {"data structures"},
	"calculus":   {"linear algebra"},

	"compilers": {
		"data structures",
		"formal languages",
		"computer organization",
	},

	"data structures":       {"discrete math"},
	"databases":             {"data structures"},
	"discrete math":         {"intro to programming"},
	"formal languages":      {"discrete math"},
	"networks":              {"operating systems"},
	"operating systems":     {"data structures", "computer organization"},
	"programming languages": {"data structures", "computer organization"},
}
```

要求：计算满足每门课程的先修课程序列。

起初我觉得这个问题有点意思，于是在微信群里和同学讨论了一会。

#### Go
首先是书中 Go 的实现，使用的是[深度优先搜索(depth-first search)][2]算法。

```go
func topoSort(m map[string][]string) []string {
	var order []string
	seen := make(map[string]bool)
	var visitAll func(items []string)

	visitAll = func(items []string) {
		for _, item := range items {
			if !seen[item] {
				seen[item] = true
				visitAll(m[item])
				order = append(order, item)
			}
		}
	}

	var keys []string
	for key := range m {
		keys = append(keys, key)
	}

	sort.Strings(keys)
	visitAll(keys)
	return order
}
```

Go 的实现非常简练，算法并不太难，也并不是本文的重点；然而@Liger 同学并不会 Go，他只会 Java；猜测像他这样的同学不少，我自己虽然很清楚 Go 肯定比 Java 简洁，但是到底差别在哪呢？带着这个疑问并应@Liger 同学的要求，于是在 vim 下将以上代码「翻译」了一遍（话说没了 IDE 写 Java 还真有点像用笔在纸上写）。

#### Java 1
根据我自己的 Java 经验，大概会「翻译」成如下的样子：

```java
public class Testing {
  private List<String> order = new ArrayList<>();
  private Map<String, Boolean> seen = new HashMap<>();
  
  public List<String> topoSort(Map<String, List<String>> m) {
    List<String> keys = new ArrayList<>();
    for (String key : m.keySet()) {
      keys.add(key);
    }
    
    Collections.sort(keys);
    visitAll(keys, m);
    return order;
  }

  private void visitAll(List<String> items, Map<String, List<String>> m) {
    for (int i = 0; i < items.size(); i++) {
      String item = items.get(i);
      if (seen.get(item) == null || !seen.get(item)) {
        seen.put(item, true);
        if (m.get(item) != null) {
          visitAll(m.get(item), m);
        }
        order.add(item);
      }
    }
  }
}
```

虽然看起来也很简洁，但与 Go 相比还是有明显的差距。Java 用了完整的一个类来实现（2个方法+2个成员域）而 Go 仅仅是一个函数搞定一切。**由于需要使用递归，不得不定义递归方法，再加上为了简单地保证变量的作用域能活跃在两个方法中不得不将变量定义在了方法之外而成了成员域**。

从问题本身来说，这和面向对象没有半毛钱关系。Go 一个函数搞定的实现是简洁优雅的，Java 方面的面向对象实现略显啰嗦...之所以会这样最本质的原因在于 **Go 里面函数为 first citizen**；而在 Java 里一切皆为对象（基本类型不讨论），而方法（函数）并不是对象。

那么问题来了，**既然一切皆为对象，那么方法能不能转化为对象呢**？答案是，能。

#### Java 2
我们可以在接口中定义**一个**方法（函数），然后在``topoSort()``方法里 new 一个该接口的匿名内部类对象，那么该对象便可以调用自身而实现递归了，从而将整个实现封装在一个方法（函数）之内。

核心代码如下：

```java
interface Func {
  void func(List<String> items);
}

public static List<String> topoSort(final Map<String, List<String>> m) {
  final List<String>         order = new ArrayList<>();
  final Map<String, Boolean> seen  = new HashMap<>();
  
  List<String> keys = new ArrayList<>();
  for (String key : m.keySet()) {
    keys.add(key);
  }

  Collections.sort(keys);

  Func visitAll = new Func() {
    @Override public void func(List<String> items) {
      for (int i = 0; i < items.size(); i++) {
        String item = items.get(i);
        if (seen.get(item) == null || !seen.get(item)) {
          seen.put(item, true);
          if (m.get(item) != null) {
            this.func(m.get(item));
          }
          order.add(item);
        }
      }
    }
  };

  visitAll.func(keys);
  return order;
}
```

可以看到，定义了接口之后，简化为了一个 static 方法（这里说函数可能更准确）搞定，更符合实际问题的场景。

对比 1 的实现已经简洁了很多，但是相比 Go 仍有差距。

Go 是后起之秀，站在巨人的肩膀上，函数式编程，闭包等都带有现代语言的特点；而 Java 这20多年来也还是在慢慢的演变，JDK8 便提供了函数式编程的支持（感觉更像是语法糖？），具体的实现方式正如 2 那样利用接口，只不过这个接口被看做为函数式接口，可以用``@java.lang.FunctionalInterface``[标注][3]。

借助 [lambda 表达式][4]，我们还能更进一步，再简单一些 :)

#### Java 3
由于 lambda 表达式没有 this 引用（或者说 this 引用指向外层作用域对象）, 这里为了简洁（是的！）故修改了函数接口，将 Func 对象自身传入接口函数中意图实现递归。

```java
@FunctionalInterface
interface Func {
  void func(List<String> items, Func recursive);
}

public static List<String> topoSort(final Map<String, List<String>> m) {
  final List<String> order = new ArrayList<>();
  final Map<String, Boolean> seen = new HashMap<>();

  List<String> keys = m.keySet().stream().collect(Collectors.toList());

  Collections.sort(keys);

  Func visitAll = (items, recursive) -> items.forEach(item -> seen.computeIfAbsent(item, s -> {
    if (m.get(item) != null) {
      recursive.func(m.get(item), recursive);
    }
    return true;
  }));

  visitAll.func(keys, visitAll);
  return order;
}
```

Finally，我们看到 Java 代码终于变得和 Go 一样简洁了，看起来似乎更简洁以至于代码变得抽象，甚至完全不像 Java 代码了（不熟悉 Java8 的前提下）。

#### What I Learn From
在从面向对象向函数式演化的过程中，代码越来越简洁也能够反应问题本身（不需要面向对象，仅仅是一个 static 函数解决一个实际问题）。语言是工具，使用者如果跳出语言的限定的作用域，从解决实际问题的角度切入，这个时候，或许就发现世界又美好了一点点了 :)

虽然我现在大部分代码都使用 Java，但我更喜欢 Go，感觉更像是为解决实际问题而存在；而从 Java 的变化来看，终究是杂糅了些，并且并不能愉快的写 Java8(Android, 大部分项目)；总之，Java 8 之前的 Java 非常旧，啰嗦又耗电...

说到 Java，最后一直觉得，我们按照固有的模式把他越写越啰嗦。记得以前写 Spring 项目的时候，什么面向接口编程，到处都是接口，不管怎么样一上来就给你定义一大堆接口。起初我觉得这种方式挺好的，灵活构建各种模块，组件，架构，设计模式等等，但是现在觉得都真的是过了。对于大多数实际项目来说，然并卵；当真正需要抽象为接口的再定义也不迟，否则就定义为一个方法，快速迭代交付再继续憋；最后**接口更应该是接口的聚合，而不是方法的聚合**，就像 FunctionalInterface 那样，小而美。

#### EOF
```yaml
background: /assets/images/default.jpg
date: 2016-01-16T01:37:42+08:00
hide: false
license: cc-40-by
location: Shenzhen
summary: 今天翻看Brian W. Kernighan所著的《The Go Programming Language》，发现一个比较有趣的问题
tags:
- Golang
- Java
weather: rainy
```

[1]: http://www.gopl.io
[2]: https://en.wikipedia.org/wiki/Depth-first_search
[3]: https://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html
[4]: https://en.wikipedia.org/wiki/Lambda_calculus
