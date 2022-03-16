# 彻底解决 YAML 多行文本格式化丢失问题

本文通过一起奇怪的现象，通过阅读源码查找原因，最后给出解决办法。

## TL; DR

先说结论，如果想要保持*literal*多行文本输入和输出的格式：

- 文本不要以空格结尾
- 不要换行前再带个空格
- 不要在文本中添加不可见特殊字符

否则，请用带引号的*flow*版本。

好了，知道了*how*，好奇*why*的同学可以继续往下读。

## 问题

我们将业务部署到k8s时，应用的配置文件通常写在ConfigMap中，然后以文件的形式挂载到Pod中。归功于[YAML的多行文本块](https://yaml-multiline.info)功能，我们可以将复杂的配置用类似如下的方式写：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: awesome-cm
data: 
  application.yaml: |
    spring:
      application:
        name: awesome-app
      profiles:
        active: local
```

这样就解决了任意格式的文本配置文件（json/yaml/toml/等等）挂载的问题，很方便。

但是，你很可能遇到过这样的问题，比如当调试`kubectl get`或修改`kubectl edit`ConfigMap时，拿到的yaml是这个样子的：

```yaml
apiVersion: v1
data:
  application.yaml: "spring:\n  application:\n    name: awesome-app\n  profiles:
    \n    active: local\n"
kind: ConfigMap
metadata:
  name: awesome-cm
```

限于篇幅删掉了一些自动生成的东西，可以看到，**提交之前本来就格式化好的yaml配置，再拿回来时格式丢失了，可读性很差，更不用说去修改了**，YAML对空格和缩进很敏感，稍不注意就报错。

为什么会这样呢？

## 原因

本着好奇和学习的心态，就带着问题去看了k8s的yaml源码包，`sigs.k8s.io/yaml`，这个库其实最终fork自`gopkg.in/yaml.v2`，k8s社区在此基础上将JSON和YAML更好地转换。毕竟真正处理的时候用的还是JSON，只是YAML更易于编写和阅读。我们会调用`yaml.Marshal`函数去将对象序列化为文本，这函数的[实现](https://github.com/go-yaml/yaml/blob/v2/yaml.go#L199)如下：

```go
func Marshal(in interface{}) (out []byte, err error) {
	defer handleErr(&err)
	e := newEncoder()
	defer e.destroy()
	e.marshalDoc("", reflect.ValueOf(in))
	e.finish()
	out = e.out
	return
}
```

`marshalDoc`负责将`in`序列化，最后`encoder`将结果写入`out`。整个序列化过程相当复杂我们也不关心，直接跳到我们感兴趣的地方，[文本序列化](https://github.com/go-yaml/yaml/blob/v2/encode.go#L300)：

```go
func (e *encoder) stringv(tag string, in reflect.Value) {
	var style yaml_scalar_style_t
	s := in.String()
	canUsePlain := true

	// ###### <deleted code>

	switch {
	case strings.Contains(s, "\n"):
		style = yaml_LITERAL_SCALAR_STYLE
	case canUsePlain:
		style = yaml_PLAIN_SCALAR_STYLE
	default:
		style = yaml_DOUBLE_QUOTED_SCALAR_STYLE
	}
	e.emitScalar(s, "", tag, style)
}
```

中间删掉了一些不重要的代码，关键的`switch`中，给文本输出选择一个style，这里简单来说就是如果包含换行符，就用多行文本块`yaml_LITERAL_SCALAR_STYLE`，再次不带引号的文本`yaml_PLAIN_SCALAR_STYLE`，最后默认带双引号`yaml_DOUBLE_QUOTED_SCALAR_STYLE`（我们的栗子）。

> 其实也很好理解，YAML会自动帮我们将多行文本展示为块的形式，其次对单行文本来说，直接展示就行了，最后单行文本里可能有转义字符啥的，就需要用双引号给扩住了。

> 这里再补充一下专业的说法：YAML中的文本有两种格式，一种叫block，也就是我们例子中带有格式的多行文本，通常用`|`来表示；另一种是flow形式，也就是栗子中用引号括起来的多行文本。详见 https://yaml-multiline.info

很明显，我们的栗子中本来就是有换行符的，但是为什么输出双引号类型呢？往下看，`emitScalar`经过几次调用，会走到解决我们疑惑的[函数](https://github.com/go-yaml/yaml/blob/v2/emitterc.go#L984)`yaml_emitter_analyze_scalar`：

```go
// Check if a scalar is valid.
func yaml_emitter_analyze_scalar(emitter *yaml_emitter_t, value []byte) bool {
	// ###### <deleted code>
	var (
		special_characters = false

		trailing_space = false
		space_break    = false
	)

	if len(value) == 0 {
		emitter.scalar_data.block_allowed = false
		return true
	}

	for i, w := 0, 0; i < len(value); i += w {
		if !is_printable(value, i) || !is_ascii(value, i) && !emitter.unicode {
			special_characters = true
		}

		if is_space(value, i) {
			if i+width(value[i]) == len(value) {
				trailing_space = true
			}
		}
	}

	emitter.scalar_data.block_allowed = true

	if trailing_space {
		emitter.scalar_data.block_allowed = false
	}

	if space_break || special_characters {
		emitter.scalar_data.block_allowed = false
	}
	return true
}
```

这函数很复杂，巨多变量，我们只保留了关心的几个，`special_characters, trailing_space, space_break`分别表示包含：

- 特殊字符（主要是无法打印的的ascii）
- 整个文本（value）以空格结尾
- 空格紧跟换行符` \n`

> 划重点，后面要考！

虽然默认`block_allowed`是`true`，但出现以上这三种情况之一，都会置为`false`，也就是说无法使用多行文本块的形式，因为在最终选取style的[函数](https://github.com/go-yaml/yaml/blob/v2/emitterc.go#L798)里：

```go
// Determine an acceptable scalar style.
func yaml_emitter_select_scalar_style(emitter *yaml_emitter_t, event *yaml_event_t) bool {
	// ###### <deleted code>
	if style == yaml_LITERAL_SCALAR_STYLE || style == yaml_FOLDED_SCALAR_STYLE {
		if !emitter.scalar_data.block_allowed || emitter.flow_level > 0 || emitter.simple_key_context {
			style = yaml_DOUBLE_QUOTED_SCALAR_STYLE
		}
	}

	emitter.scalar_data.style = style
	return true
}
```

在上上一步`stringv`中，我们已经选取了`yaml_LITERAL_SCALAR_STYLE`，这里通过上一步的分析，作出最终的选择，由于`emitter.scalar_data.block_allowed == false`，所以最终又回到了`yaml_DOUBLE_QUOTED_SCALAR_STYLE`，也即双引号的形式。

最终输出的在[这函数](https://github.com/go-yaml/yaml/blob/v2/emitterc.go#L891)里：

```go
// Write a scalar.
func yaml_emitter_process_scalar(emitter *yaml_emitter_t) bool {
	switch emitter.scalar_data.style {
	case yaml_PLAIN_SCALAR_STYLE:
		return yaml_emitter_write_plain_scalar(emitter, emitter.scalar_data.value, !emitter.simple_key_context)

	case yaml_SINGLE_QUOTED_SCALAR_STYLE:
		return yaml_emitter_write_single_quoted_scalar(emitter, emitter.scalar_data.value, !emitter.simple_key_context)

	case yaml_DOUBLE_QUOTED_SCALAR_STYLE:
		return yaml_emitter_write_double_quoted_scalar(emitter, emitter.scalar_data.value, !emitter.simple_key_context)

	case yaml_LITERAL_SCALAR_STYLE:
		return yaml_emitter_write_literal_scalar(emitter, emitter.scalar_data.value)

	case yaml_FOLDED_SCALAR_STYLE:
		return yaml_emitter_write_folded_scalar(emitter, emitter.scalar_data.value)
	}
	panic("unknown scalar style")
}
```

这里包含了所有合法的yaml标量（非array和结构）类型，在`yaml_emitter_write_literal_scalar`函数里可以看到熟悉的`|`，这里就不赘述了。

好了，可以破案了，为什么失去了格式化？

相信同学你应该看出来了，尽管我们是多行文本，一开始预选的style也是literal block多行的，由于我们不小心在换行之前添加了一个空格，触发了前面代码中的`space_break`规则，`block_allowed`为`false`，导致最终选取了双引号的flow style。

```yaml
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"application-local.yaml":"spring:\n  application:\n    name: awesome-app\n  profiles: \n    active: local\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"awesome-cm","namespace":"apaas-design"}}
```

> kubectl会以json形式附加本次apply过去的内容，可以看到 ` profiles: \n`，不小心在换行前多打了个空格。

最后，为什么yaml要区分block和flow两种形式的文本输出呢？这里说说我的想法：

block形式下，特殊的字符无法被转义，特别是行末的空格，制表符等等，容易造成输入输出不一致，虽然**我们99.9%不会在行末添加不可见字符给自己找麻烦**。但是，*人是会犯错误的*，在写yaml的时候容易在行末不小心多按一下空格，反正又不可见...

flow形式下的文本就没这问题，他可以在引号里将特殊符号转义，这样就保证了输入输出的一致性，就是可读性太差了些。

> 最后再注明下，YAML 空格接换行是合法的语法，在非标量（比如后面接个结构/数组）的情况下，完全没有问题，本文仅讨论多行文本。

## 结论

对于看起来表现行为很奇怪的问题，几次尝试无果后，可以去看看源码，虽然可能很复杂，但是只看感兴趣的地方，带着问题看源码，其实也还好，解决问题的同时还能学到一些东西，甚至[贡献代码](https://github.com/go-yaml/yaml/pull/830)。


## EOF

```yaml
summary: 提交之前本来就格式化好的yaml配置，再拿回来时格式丢失了，可读性很差，更不用说去修改了，为什么这样，又该怎么解决？
weather: rainy
license: cc-40-by
location: Shenzhen
background: ./photo-1470252649378-9c29740c9fa8.jpeg
tags: [k8s]
date: 2022-03-14T11:15:00+08:00
```