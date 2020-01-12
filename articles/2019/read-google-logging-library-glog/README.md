
# 阅读Google glog源码

最近把一个新写的Go项目的日志模块从标准库中的`log`切换到了Google的[glog](https://github.com/golang/glog)。它解决了标准库中很多痛点，如给日志分级，写入文件并切分等。大名鼎鼎的[Kubernetes](https://kubernetes.io/)就是使用它作为日志模块。看了下代码，3个文件，1k多行，非常适合学习，于是便仔细阅读了一下源码和大家分享。

> 由于glog自身存在的一些问题以及基本上停止开发了，Kubernetes fork glog为[klog](https://github.com/kubernetes/klog)，修正了一些问题并新增了单个文件日志等特性，但是变化基本不大。
> 
> 默认情况下不在`init()`函数里去读取`flags`变量，除此一位还有一个新的`logr`日志接口，看起来挺有意思的，可以关注下。
> 
> 另外，我还给klog[贡献了代码](./contribute-klog)，嘿嘿。
> 
> 2019-10-21

## What

先简要介绍一下glog，让大家有个概念。你可以给每条日志指定一个级别（Severity Level），内置4个，从低到高分别是：

-   `INFO`
-   `WARNING`
-   `ERROR`
-   `FATAL`

其中使用`FATAL`会写入日志后会终止进程。使用方式很简单，`log.Info()`，`log.Errorf()`等等，具体使用和标准库一样，只需要选择一个level即可。

每个级别都对应一个文件，级别越低的会包含级别更高的日志，也就是说`INFO`会包含所有level的日志，而`ERROR`只会包含其自身以及`FATAL`的日志。

以上便是基本使用方式，注意在使用前需要调用`flag.Parse()`，至于为什么，就需要我们进入源码看看了。

## How

glog有几个常用的参数：

```sh
-alsologtostderr
    log to standard error as well as files
-log_dir string
    If non-empty, write log files in this directory
-logtostderr
    log to standard error instead of files
-v value
    log level for V logs
```

需要注意的是，这些参数的设置不是在代码里，而是命令行，原因是glog使用的是`flag`包在`init()`里初始化配置的，这就是为什么在使用前一定要调用`flag.Parse()`的原因。

初始化后，我们就可以使用`log.Xxxx()`来愉快地打印日志了。

所有的`log.Xxxx`最终都会进入到`output()`函数中：

```go
// output writes the data to the log files and releases the buffer.
func (l *loggingT) output(s severity, buf *buffer, file string, line int, alsoToStderr bool) {
  l.mu.Lock()
  if l.traceLocation.isSet() {
    if l.traceLocation.match(file, line) {
      buf.Write(stacks(false))
    }
  }
  data := buf.Bytes()
  if !flag.Parsed() {
    os.Stderr.Write([]byte("ERROR: logging before flag.Parse: "))
    os.Stderr.Write(data)
  } else if l.toStderr {
    os.Stderr.Write(data)
  } else {
    if alsoToStderr || l.alsoToStderr || s >= l.stderrThreshold.get() {
      os.Stderr.Write(data)
    }
    if l.file[s] == nil {
      if err := l.createFiles(s); err != nil {
        os.Stderr.Write(data) // Make sure the message appears somewhere.
        l.exit(err)
      }
    }
    switch s {
    case fatalLog:
      l.file[fatalLog].Write(data)
      fallthrough
    case errorLog:
      l.file[errorLog].Write(data)
      fallthrough
    case warningLog:
      l.file[warningLog].Write(data)
      fallthrough
    case infoLog:
      l.file[infoLog].Write(data)
    }
  }
  if s == fatalLog {
    // If we got here via Exit rather than Fatal, print no stacks.
    // ....
  }
  l.putBuffer(buf)
  l.mu.Unlock()
  if stats := severityStats[s]; stats != nil {
    atomic.AddInt64(&stats.lines, 1)
    atomic.AddInt64(&stats.bytes, int64(len(data)))
  }
}
```

在`output()`之前，还会调用`formatHeader()`对日志前缀进行格式化，后面会谈到。可以看到，这函数流程还是很清晰的。根据用户的配置参数确定写入到哪里，写入的级别，以及分别写入不同级别的日志文件。

核心的代码就是这样，下面分享一些代码里有趣和值得学习的地方，也可能解决你的一些疑问。

## Dive in Code

### 并发处理

日志是最常用的组件之一了，每时每刻可能都有成千上万条日志在打印，如何正确地保证并发安全是日志库可靠高效的前提。

glog使用了多种方式以保证并发安全，主要有两方面。

首先，对于一些基础的配置变量的读写，使用了`sync/atomic`包以保证对这些共享变量读写的**原子性**，比如，

```go
// get returns the value of the severity.
func (s *severity) get() severity {
  return severity(atomic.LoadInt32((*int32)(s)))
}

// set sets the value of the severity.
func (s *severity) set(val severity) {
  atomic.StoreInt32((*int32)(s), int32(val))
}
```

为什么不使用goroutine+channel呢，个人觉得很大程度上因为日志是属于相对底层的业务，是强同步的，使用原子包代码更容易读写。如果使用channel，代码看起来就像各种channel飞来飞去，反而增加了复杂度。go虽然推荐我们使用channel而来做同步共享变量的访问，但是也没说不要用锁或者原子对不对，否则也不会提供原子和锁这些库了。所以没有最好的，只有合适的方案。

其次，对于日志库的核心共享变量，使用了**互斥锁**，比如，

```go
// loggingT collects all the global state of the logging setup.
type loggingT struct {
  // ...

  // Level flag. Handled atomically.
  stderrThreshold severity // The -stderrthreshold flag.

  // freeList is a list of byte buffers, maintained under freeListMu.
  freeList *buffer
  // freeListMu maintains the free list. It is separate from the main mutex
  // so buffers can be grabbed and printed to without holding the main lock,
  // for better parallelization.
  freeListMu sync.Mutex

  // ...

  // mu protects the remaining elements of this structure and is
  // used to synchronize logging.
  mu sync.Mutex
  // ...
  verbosity Level      // V logging level, the value of the -v flag/
}
```

结构体`loggingT`是全局共享的，所有的日志函数都会用到它，他是如何保证并发安全呢？可以看到结构体定义了`mu`，用来同步所有日志请求。在函数`output()`我们也看到了，函数一开始就是调用`l.mu.Lock()`去获取锁的。

另外值得注意的是，锁的命名和注释很有意思。`mu`用来默认用来锁住所有的结构体变量，`xxxMu`用来锁`xxx`变量，并加上注释，另外在需要持有锁的函数注释上也都加上了诸如，`l.mu is held.`的字样。

这些写法很值得借鉴。

### 变量分组

代码里有这样一处，看了很久才看懂，还以为是无法编译的：

```go
// Stats tracks the number of lines of output and number of bytes
// per severity level. Values must be read with atomic.LoadInt64.
var Stats struct {
  Info, Warning, Error OutputStats
}

var severityStats = [numSeverity]*OutputStats{
  infoLog:    &Stats.Info,
  warningLog: &Stats.Warning,
  errorLog:   &Stats.Error,
}
```

原来还可以使用var利用struct对变量进行分组，这样我们就可以根据变量所属的类型进行分组了，对于变量特别多的时候代码可能会更清晰。

### 何时同步日志到文件中？

IO通常是耗时的，对于日志来说更是如此，如果频繁的磁盘IO，日志一定会对业务造成性能影响。glog优化的方式和我们想的一样，将日志先放入内存缓冲区，定时器间隔到了，文件大小达到阀值或者程序即将调用`exit()`，glog就会讲缓冲区的日志同步到文件中，默认间隔为30秒。

```go
func init() {
  // ...
  flag.Var(&logging.traceLocation, "log_backtrace_at", "when logging hits line file:N, emit a stack trace")

  // Default stderrThreshold is ERROR.
  logging.stderrThreshold = errorLog

  logging.setVState(0, nil, false)
  go logging.flushDaemon()
}

// flushDaemon periodically flushes the log file buffers.
func (l *loggingT) flushDaemon() {
  for _ = range time.NewTicker(flushInterval).C {
    l.lockAndFlushAll()
  }
}

// lockAndFlushAll is like flushAll but locks l.mu first.
func (l *loggingT) lockAndFlushAll() {
  l.mu.Lock()
  l.flushAll()
  l.mu.Unlock()
}
```

### 自适应的吞吐量

glog在很多地方都使用了一些技巧来提升性能，捡个有趣的例子聊聊。

日志buffer，即每条写入日志所分配的缓冲区。频繁的分配和回收这些缓冲区对性能肯定造成影响，glog使用了2个简短的函数**优雅**地解决了这个问题。

```go
// buffer holds a byte Buffer for reuse. The zero value is ready for use.
type buffer struct {
  bytes.Buffer
  tmp  [64]byte // temporary byte array for creating headers.
  next *buffer
}

// getBuffer returns a new, ready-to-use buffer.
func (l *loggingT) getBuffer() *buffer {
  l.freeListMu.Lock()
  b := l.freeList
  if b != nil {
    l.freeList = b.next
  }
  l.freeListMu.Unlock()
  if b == nil {
    b = new(buffer)
  } else {
    b.next = nil
    b.Reset()
  }
  return b
}

// putBuffer returns a buffer to the free list.
func (l *loggingT) putBuffer(b *buffer) {
  if b.Len() >= 256 {
    // Let big buffers die a natural death.
    return
  }
  l.freeListMu.Lock()
  b.next = l.freeList
  l.freeList = b
  l.freeListMu.Unlock()
}
```

每次调用`log.Xxxx()`时，都会向日志对象索取一个buffer用来写当前的这条日志。`FreeList`的数据结构就是一个链表，`getBuffer()`每次取头节点返回并从链表中删除，链表为空则创建新的。`putBuffer()`则在日志写入缓冲区后返还给日志对象，该buffer则又会设置成链头（过大的buffer会被丢弃）。值得注意的是这里使用了细粒度锁`freeListMu`带来更好的并发性能。

这段代码的漂亮之处在于他的日志吞吐量是**自适应**的。业务日志很大，则buffer会分配多些，反之则少些。

### 提升性能的小技巧

越是基础的模块，越要注重性能，在一些写法上可能和我们通常的业务不太一样。

我们以每次写入日志时的都要开始写header为例，header即日期时间格式化，文件名，调用日志所在行数等。

```go
// formatHeader formats a log header using the provided file name and line number.
func (l *loggingT) formatHeader(s severity, file string, line int) *buffer {
  now := timeNow()
  if line < 0 {
    line = 0 // not a real line number, but acceptable to someDigits
  }
  if s > fatalLog {
    s = infoLog // for safety.
  }
  buf := l.getBuffer()

  // Avoid Fprintf, for speed. The format is so simple that we can do it quickly by hand.
  // It's worth about 3X. Fprintf is hard.
  _, month, day := now.Date()
  hour, minute, second := now.Clock()
  // Lmmdd hh:mm:ss.uuuuuu threadid file:line]
  buf.tmp[0] = severityChar[s]
  buf.twoDigits(1, int(month))
  buf.twoDigits(3, day)
  buf.tmp[5] = ' '
  buf.twoDigits(6, hour)
  buf.tmp[8] = ':'
  buf.twoDigits(9, minute)
  buf.tmp[11] = ':'
  buf.twoDigits(12, second)
  buf.tmp[14] = '.'
  buf.nDigits(6, 15, now.Nanosecond()/1000, '0')
  buf.tmp[21] = ' '
  buf.nDigits(7, 22, pid, ' ') // TODO: should be TID
  buf.tmp[29] = ' '
  buf.Write(buf.tmp[:30])
  buf.WriteString(file)
  buf.tmp[0] = ':'
  n := buf.someDigits(1, line)
  buf.tmp[n+1] = ']'
  buf.tmp[n+2] = ' '
  buf.Write(buf.tmp[:n+3])
  return buf
}
```

可以看到，写入的时候并没有使用`Pirntf()`这样依赖于反射的方式，而是手写buffer。虽然代码行数多了一些，但是换来了3倍的性能提升，对于频繁调用的日志来说是非常值得的。

另外，在这里还有几段值得品味的代码。

```go
// Some custom tiny helper functions to print the log header efficiently.

const digits = "0123456789"

// twoDigits formats a zero-prefixed two-digit integer at buf.tmp[i].
func (buf *buffer) twoDigits(i, d int) {
  buf.tmp[i+1] = digits[d%10]
  d /= 10
  buf.tmp[i] = digits[d%10]
}

// nDigits formats an n-digit integer at buf.tmp[i],
// padding with pad on the left.
// It assumes d >= 0.
func (buf *buffer) nDigits(n, i, d int, pad byte) {
  j := n - 1
  for ; j >= 0 && d > 0; j-- {
    buf.tmp[i+j] = digits[d%10]
    d /= 10
  }
  for ; j >= 0; j-- {
    buf.tmp[i+j] = pad
  }
}
```

再一次说明了数据结构和算法的重要性哈哈。

### 什么时候切分日志文件？

glog切分日志的策略是，当日志文件达到一定的大小（而不是按照日期）后会将日志写入新的文件。我觉得这方式也挺好的，按照日志来每个文件大小都不一样，再说了每个日志文件都会带有第一条日志的时间方便我们根据时间来查找日志。

```go
// MaxSize is the maximum size of a log file in bytes.
var MaxSize uint64 = 1024 * 1024 * 1800

func (sb *syncBuffer) Write(p []byte) (n int, err error) {
  if sb.nbytes+uint64(len(p)) >= MaxSize {
    if err := sb.rotateFile(time.Now()); err != nil {
      sb.logger.exit(err)
    }
  }
  n, err = sb.Writer.Write(p)
  sb.nbytes += uint64(n)
  if err != nil {
    sb.logger.exit(err)
  }
  return
}
```

这就很清楚了，每次将日志写入到缓冲区前都会检查待缓冲区大小是否已达到设置的`MaxSize`，默认为1.8G。

### 桥接标准库日志

glog还提供了一个重定向标准库日志的功能。

```go
func CopyStandardLogTo(name string) {
  sev, ok := severityByName(name)
  if !ok {
    panic(fmt.Sprintf("log.CopyStandardLogTo(%q): unrecognized severity name", name))
  }
  // Set a log format that captures the user's file and line:
  //   d.go:23: message
  stdLog.SetFlags(stdLog.Lshortfile)
  stdLog.SetOutput(logBridge(sev))
}
```

本质上就是`logBridge`实现了`Write()`接口，然后将标准库的输出写到了`logBridge`中，最后转发到`output()`中。

### `V()`函数实现自定义日志级别（条件日志）

glog默认只提供了4种级别，如果我们想扩展（比如增加debug，verbose），但是不想改源码怎么办？glog提供了`V()`函数，只需要，

```go
if glog.V(1) {
  glog.Infof(...)
}

// or

glog.V(1).Infof(...)
```

我们可以通过命令行参数`--v=100`设置V参数，默认为0，当调用`V(n)`，n大于等于设置的v时，日志才会打印，这样就实现了条件日志。但是要注意，使用后者写入日志的level只能是**INFO**。

`V()`函数的具体实现还是挺有技巧的，感兴趣可以自己看看，这里不展开了。

### 利用函数变量进行mock测试

测试日志写入需要header，格式化当前的时间，**单元测试是不能依赖外部**的，时间就是最好的例子。glog使用了**函数变量**来解决这问题：

```go
// prod file
var timeNow = time.Now // Stubbed out for testing.

// test file
func TestHeader(t *testing.T) {
  setFlags()
  defer logging.swap(logging.newBuffers())
  defer func(previous func() time.Time) { timeNow = previous }(timeNow)
  timeNow = func() time.Time {
    return time.Date(2006, 1, 2, 15, 4, 5, .067890e9, time.Local)
  }
  // ...
}
```

可以看到，实际生产代码用真实的时间函数；测试代码mock了这个函数返回一个固定的时间，在任何平台任何时间测试结果都是一样的。这样就达到了单元测试不依赖于外部环境的要求。

别忘了用defer还原预设值噢！

## EOF

```yaml
title: 阅读Google glog源码
summary: 阅读Google glog源码 
weather: rainy
license: cc-40-by
location: 22, 114
background: https://cdn.jsdelivr.net/gh/golang/go/doc/gopher/fiveyears.jpg
tags: [golang, source]
date: 2019-09-02T22:55:27+08:00
```
