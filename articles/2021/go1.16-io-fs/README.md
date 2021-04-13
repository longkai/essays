# Go 1.16 io/fs 设计与实现及正确使用姿势

## 摘要
go1.16新增了一个包，`io/fs`，用来统一标准库文件io相关的访问方式。本文通过对比新旧文件io的用法差异，带着问题查看源码：标准库是如何保证兼容性的，如何实现一个基于操作系统中的FS，以及如何做单元测试。相信看完后大家会掌握go文件操作的正确姿势。

## 背景
最近做的一个项目需要频繁和文件系统打交道，在本地写了不少单元测试。刚好近期go更新到了1.16，新增了`io/fs`包，抽象了统一的文件访问相关接口，对测试也更友好，于是在项目中实践了一下，体验确实不错。

为了让大家更好的了解它，使用它，今天我们来从聊聊它的设计与实现。

下一篇会通过`fs.FS`接口来设计和实现一个*对象存储FS*，敬请期待。

先来看看我们现在是怎么处理文件io的。

## go 1.16 前后的文件io对比
```go
// findTargetFile 查找dir目录下的所有文件，返回第一个文件名以ext为扩展名的文件内容
// 
// 假设一定存在至少一个这样的文件
func findExtFile(dir string, ext string) ([]byte, error) {
	entries, err := ioutil.ReadDir(dir)
	if err != nil {
		return nil, err
	}
	for _, e := range entries {
		if filepath.Ext(e.Name()) == ext && !e.IsDir() {
			name := filepath.Join(dir, e.Name())
			// 其实可以一行代码返回，这里只是为了展示更多的io操作
			// return ioutil.ReadFile(name)
			f, err := os.Open(name)
			if err != nil {
				return nil, err
			}
			defer f.Close()
			return ioutil.ReadAll(f)
		}
	}
	panic("never happen")
}
```

注释说得很清楚这函数是要干啥了，这里面有几个关键函数，`ioutil.ReadDir`用于读取目录下的所有文件（非递归），`os.Open`用于打开文件，最后`ioutil.ReadAll`用于将整个文件内容读到内存中。

比较直接，我们来看看在go1.16及之后对应的函数实现：

```go
func findExtFileGo116(dir string, ext string) ([]byte, error) {
	fsys := os.DirFS(dir)                 // 以dir为根目录的文件系统，也就是说，后续所有的文件在这目录下
	entries, err := fs.ReadDir(fsys, ".") // 读当前目录
	if err != nil {
		return nil, err
	}
	for _, e := range entries {
		if filepath.Ext(e.Name()) == ext && !e.IsDir() {
			// 也可以一行代码返回
			// return fs.ReadFile(fsys, e.Name())
			f, err := fsys.Open(e.Name()) // 文件名fullname `{dir}/{e.Name()}``
			if err != nil {
				return nil, err
			}
			defer f.Close()
			return io.ReadAll(f)
		}
	}
	panic("never happen")
}
```

代码主体基本没变，最大的不同就是在文件io调用上，我们使用了一个`os.DirFS`，这是go内建的基于目录树的os文件系统实现，接下来的几个操作都是基于这个fs。比如读取目录内容`fs.ReadDir(fsys, '')`还有打开文件`fsys.Open(file)`，实际上文件io都委托给这个fs了，添加了一层抽象。

> 注意到个别熟悉的老朋友`ioutil.ReadAll`变成了`io.ReadAll`，这也是go1.16的一个变化，ioutil包被废弃了，对应的函数在`io`和`os`中。

让我们来看看这个`dirFS`到底是什么，它的[定义](https://github.com/golang/go/blob/master/src/os/file.go#L626)是这样的：

```go
func DirFS(dir string) fs.FS {
	return dirFS(dir)
}

type dirFS string

func (dir dirFS) Open(name string) (fs.File, error) {
	if !fs.ValidPath(name) || runtime.GOOS == "windows" && containsAny(name, `\:`) {
		return nil, &PathError{Op: "open", Path: name, Err: ErrInvalid}
	}
	f, err := Open(string(dir) + "/" + name)
	if err != nil {
		return nil, err // nil fs.File
	}
	return f, nil
}
```

`os.DirFS`返回了一个`fs.FS`类型的接口实现，这个接口只有一个方法`Open`，实现上也很简单，直接调用`os.Open`得到一个`*os.File`对象，注意到返回类型是`fs.File`，所以这意味着前者实现了后者。

还有一个关键点，打开的文件名是以dir为根路径的，完整的文件名是`dir/name`，这点一定要注意，并且Open的name参数一定不能以`/`打头或者结尾，为什么会这样呢？让我们走进go1.16的`io/fs`包。

## Go 1.16 fs 包
Go1.16新增了一个`io/fs`包，定义了`fs.FS`的接口，抽象了一个只读的文件树，标准库的包已经适配了这个接口。也是对新增的`embed`包的抽象，所有的文件相关的都可以基于这个包来抽象，我觉得挺好的，方便了测试。不太好的地方就是*只读*的，无法写，当然这不妨碍我们去扩展接口。

同时，`io/ioutil`这个包被废弃了，转移到了`io`和`os`包中。事实证明，存在`util`这样的包是设计上懒的表现，可能会有越来越多无关的代码都放这里面，最后给使用者造成困惑。

先来看看fs这个包是什么，怎么用。

```go
// An FS provides access to a hierarchical file system.
//
// The FS interface is the minimum implementation required of the file system.
// A file system may implement additional interfaces,
// such as ReadFileFS, to provide additional or optimized functionality.
type FS interface {
	// Open opens the named file.
	//
	// When Open returns an error, it should be of type *PathError
	// with the Op field set to "open", the Path field set to name,
	// and the Err field describing the problem.
	//
	// Open should reject attempts to open names that do not satisfy
	// ValidPath(name), returning a *PathError with Err set to
	// ErrInvalid or ErrNotExist.
	Open(name string) (File, error)
}

// A File provides access to a single file.
// The File interface is the minimum implementation required of the file.
// A file may implement additional interfaces, such as
// ReadDirFile, ReaderAt, or Seeker, to provide additional or optimized functionality.
type File interface {
	Stat() (FileInfo, error)
	Read([]byte) (int, error)
	Close() error
}
```

FS只有一个方法，通过名字打开一个*文件*，符合go的最小接口原则，后面可以组合成更复杂的FS。注意这个`fs.File`不是我们熟悉的那个`*os.File`，而是一个接口，上一节也提及了。这接口实际上是`io.Reader`和`io.Closer`的组合。所以如果需要访问操作系统的文件，这接口的实现可以是这样的：

```go
type fsys struct {}

func (*fsys) Open(name string) (fs.File, error) { return os.Open(name) }
```

这实现和`os.dirFS`几乎一样。但是如果就只能打开一个文件，似乎也没什么x用（后面会打脸），如果要读目录呢？实际上`io/fs`已经提供了几种常用的FS：

```go
// ReadFileFS is the interface implemented by a file system
// that provides an optimized implementation of ReadFile.
type ReadFileFS interface {
	FS

	// ReadFile reads the named file and returns its contents.
	// A successful call returns a nil error, not io.EOF.
	// (Because ReadFile reads the whole file, the expected EOF
	// from the final Read is not treated as an error to be reported.)
	ReadFile(name string) ([]byte, error)
}

// ReadDirFS is the interface implemented by a file system
// that provides an optimized implementation of ReadDir.
type ReadDirFS interface {
	FS

	// ReadDir reads the named directory
	// and returns a list of directory entries sorted by filename.
	ReadDir(name string) ([]DirEntry, error)
}

// A GlobFS is a file system with a Glob method.
type GlobFS interface {
	FS

	// Glob returns the names of all files matching pattern,
	// providing an implementation of the top-level
	// Glob function.
	Glob(pattern string) ([]string, error)
}

// statFS, subFS, walkDir etc.
```

依次是用来读整个文件到内存，读目录和glob文件名匹配，这些fs都扩展了最基本的`fs.FS`。`io/fs`包里面定义的接口，标准库里文件io相关的操作都已经实现（只读），比如，一个通用的`fsys`可以是这样的：

```go
// Fsys is an OS File system.
type Fsys struct {
	fs.FS
	fs.ReadFileFS
	fs.ReadDirFS
	fs.GlobFS
	fs.StatFS
}

func (fs *Fsys) Open(name string) (fs.File, error) { return os.Open(name) }

func (fs *Fsys) ReadFile(name string) ([]byte, error) { return os.ReadFile(name) }

func (fs *Fsys) ReadDir(name string) ([]fs.DirEntry, error) { return os.ReadDir(name) }

func (fs *Fsys) Glob(pattern string) ([]string, error) { return filepath.Glob(pattern) }

func (fs *Fsys) Stat(name string) (fs.FileInfo, error) { return os.Stat(name) }
```

可以看到函数名，返回值都是一毛一样的，就是包不同！感兴趣的同学看看`zip.Reader.Open`的实现。

> 非也非也，想起了天龙八部里的包不同hhh。

> 注意到`ReadFile()`是不是似曾相识？对，就是以前常用的那个`io/ioutil.ReadFile()`，现在被移到到了io包中；同理`ioutil.ReadDir`也移到了`os`包里

我们对比下源码来看看到底改了什么东西，兼容性又是如何保证的。

在go1.15中，[函数ReadDir](https://github.com/golang/go/blob/go1.15.11/src/io/ioutil/ioutil.go#L93)签名是：

```go
func ReadDir(dirname string) ([]os.FileInfo, error) { /* deleted */ }
```

对应的`os.FileInfo`的[源码](https://github.com/golang/go/blob/go1.15.11/src/os/types.go#L21)：

```go
// A FileInfo describes a file and is returned by Stat and Lstat.
type FileInfo interface {
	Name() string       // base name of the file
	Size() int64        // length in bytes for regular files; system-dependent for others
	Mode() FileMode     // file mode bits
	ModTime() time.Time // modification time
	IsDir() bool        // abbreviation for Mode().IsDir()
	Sys() interface{}   // underlying data source (can return nil)
}
```

而在go1.16中，是[这样](https://github.com/golang/go/blob/go1.16.3/src/os/types.go#L21)的：

```go
// A FileInfo describes a file and is returned by Stat and Lstat.
type FileInfo = fs.FileInfo
```

然后我们再看看1.16中`fs.FileInfo`的[声明](https://github.com/golang/go/blob/go1.16.3/src/io/fs/fs.go#L150)：

```go
// A FileInfo describes a file and is returned by Stat.
type FileInfo interface {
	Name() string       // base name of the file
	Size() int64        // length in bytes for regular files; system-dependent for others
	Mode() FileMode     // file mode bits
	ModTime() time.Time // modification time
	IsDir() bool        // abbreviation for Mode().IsDir()
	Sys() interface{}   // underlying data source (can return nil)
}
```

这。。。不就是把代码从os包**剪切**到了fs包下，然后利用了*type alias*的特性，旧版本的类型名字没有变化，实际类型已经指向了`io/fs`包的类型，而`fs`包里的。这就是为什么标准库是在保证兼容性的前提下如何实现了这些`io/fs`接口的，我们其实也可以借鉴这方式来实现接口的平滑升级而不会造成不兼容。

那么搞出来这个意义何在呢？我觉得主要有两方面的原因：

- 屏蔽了fs具体的实现，我们可以按照需要替换成不同的文件系统实现，比如内存/磁盘/网络/分布式的文件系统
- 易于单元测试，实际也是上一点的体现

这比在代码里硬编码`os.Open`要更灵活，如果代码中全是`os`包，则将实现完全绑定在了操作系统的文件系统，一个最直接的问题是单元测试比较难做（好在go可以使用./testdata将测试文件打包在代码里）。但这本质上还是*集成测试*而非*单元测试*，因为单元测试不依赖外部系统全在内存里搞，比如时间和OS相关模块（比如fs）。我们需要mock的fs来做单元测试，把fs抽象成接口就好了，这也是`io.fs`这个包最大的作用吧，单元测试是接口第一使用者。

我们来看一个具体的栗子看看如何对fs进行单元测试。

## 食用栗子
在我们的玩具栗子中，想要实现一个查看文件内容是否包含`666`文本的逻辑。

```go
// finder.go
type Finder struct {
	fs fs.ReadFileFS
}

// fileContains666 reports whether name contains `666` text.
func (f *Finder) fileContains666(name string) (bool, error) {
	b, err := f.fs.ReadFile(name)
	return bytes.Contains(b, []byte("666")), err
}

// Example 使用食用栗子
func Example() {
	finder := &Finder{fs: &Fsys{}}
	ok, err := finder.fileContains666("path/to/file")
	// ...
}

// finder_test.go
var testFs = fstest.MapFS{
	"666.txt": {
		Data:    []byte("hello, 666"),
		Mode:    0456,
		ModTime: time.Now(),
		Sys:     1,
	},
	"no666.txt": {
		Data:    []byte("hello, world"),
		Mode:    0456,
		ModTime: time.Now(),
		Sys:     1,
	},
}

func TestFileContains666(t *testing.T) {
	finder := &Finder{fs: testFs}
	if got, err := finder.fileContains666("666.txt"); !got || err != nil {
		t.Fatalf("fileContains666(%q) = %t, %v, want true, nil", "666.txt", got, err)
	}

	if got, err := finder.fileContains666("no666.txt"); got || err != nil {
		t.Fatalf("fileContains666(%q) = %t, %v, want false, nil", "no666.txt", got, err)
	}
}
```

业务逻辑的结构体中Finder里有一个`fs`接口对象，该对象提供了读一个文件的功能，至于具体怎么读，Finder并不关心，它只关心能搞到文件的字节数组，然后看看里面是否包含666。正常使用的时候只需要在初始化时传一个操作系统的实现，也就是上一节中的`Fsys`就好了。

我们比较关心单元测试的实现，如何不依赖操作系统的文件系统来做呢？Go1.16提供了一个`fstest.MapFS`的fs的[实现](https://github.com/golang/go/blob/go1.16.3/src/testing/fstest/mapfs.go#L33)：

```go
// A MapFS is a simple in-memory file system for use in tests,
// represented as a map from path names (arguments to Open)
// to information about the files or directories they represent.
//
// deleted comment...
type MapFS map[string]*MapFile

// A MapFile describes a single file in a MapFS.
type MapFile struct {
	Data    []byte      // file content
	Mode    fs.FileMode // FileInfo.Mode
	ModTime time.Time   // FileInfo.ModTime
	Sys     interface{} // FileInfo.Sys
}

var _ fs.FS = MapFS(nil)
var _ fs.File = (*openMapFile)(nil)

// deleted...
```

这个MapFS把`io/fs`里的接口都实现了，全部在内存中，我们只需要提供文件名key，还有文件的内容来初始化这个fs，然后拿它去初始化待测的结构体就OK了，这样才是真正的单元测试。

除此之外，go1.16在fs的测试上还提供了一个方法，`TestFS`，看看[它](https://github.com/golang/go/blob/go1.16.3/src/testing/fstest/testfs.go#L38)，它可以用来检测一个文件系统的实现是否满足要求（walk当前及子目录树，打开所有的文件检测行为是否满足标准），不以及是否在文件系统中存在某些文件：

```go
// TestFS tests a file system implementation.
// It walks the entire tree of files in fsys,
// opening and checking that each file behaves correctly.
// It also checks that the file system contains at least the expected files.
// 
// deleted comment...
func TestFS(fsys fs.FS, expected ...string) error { /* deleted */ }
```

我们可以拿上一节的`Fsys`来试试，

```go
if err := fstest.TestFS(new(fsys), "finder_test.go"); err != nil {
	t.Fatal(err)
}
// TestFS found errors:
//     .: Open(/.) succeeded, want error
//     .: Open(./.) succeeded, want error
//     .: Open(/) succeeded, want erro
```

可以看到出错了，原因是我们没有按照`fs.FS`接口contract里面要求那样，去验证传入的path的合法性：

> Open should reject attempts to open names that do not satisfy ValidPath(name), returning a *PathError with Err set to ErrInvalid or ErrNotExist.

这也是为什么在前面提到Open的name参数不能以`/`开头或者结尾的原因。但是我们不禁疑问，为什么？源代码里没有明确说，我觉得有下面几个原因：

- 规范化，`/tmp/` 和 `/tmp`，由于`fs.FS`只有一个`Open`函数，对他来说它不关心是目录还是文件，所以不应该有结尾的`/` （可能有些同学觉得以`/`结尾的为目录）
- 扩展性，我们可以以某个目录dir作为base fs，然后衍生出子fs，也就是subFS，所以不能以根目录`/`打头

确实如此，你可能注意到了，`io/fs`里的确有这样一个接口:

```go
// A SubFS is a file system with a Sub method.
type SubFS interface {
	FS

	// Sub returns an FS corresponding to the subtree rooted at dir.
	Sub(dir string) (FS, error)
}
```

同理，`os.dirFS`也是如此。

所有为了通过fs的验收，还需要改改：

```go
func (f *fsys) Open(name string) (fs.File, error) {
	if ok := fs.ValidPath(name); !ok {
		return nil, &fs.PathError{Op: "open", Path: name, Err: os.ErrInvalid}
	}
	return os.Open(name)
}

func (f *fsys) ReadFile(name string) ([]byte, error) {
	if ok := fs.ValidPath(name); !ok {
		return nil, &fs.PathError{Op: "open", Path: name, Err: os.ErrInvalid}
	}
	return os.ReadFile(name)
}
```

test pass，这样这个fsys的实现就是满足go的要求的了！

但是，就这样结束了吗？我需要实现这么多个接口吗？为什么`os.dirFS`只实现了一个`fs.FS`，其余的接口，比如读dir去哪儿了？

## 包函数
注意到一个有意思的地方，`io/fs`包里面，每个接口，同时还包含了package的函数：

```go
func ReadFile(fsys FS, name string) ([]byte, error) { 
	if fsys, ok := fsys.(ReadFileFS); ok {
		return fsys.ReadFile(name)
	}

	file, err := fsys.Open(name)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	var size int
	if info, err := file.Stat(); err == nil {
		size64 := info.Size()
		if int64(int(size64)) == size64 {
			size = int(size64)
		}
	}

	data := make([]byte, 0, size+1)
	for {
		if len(data) >= cap(data) {
			d := append(data[:cap(data)], 0)
			data = d[:len(data)]
		}
		n, err := file.Read(data[len(data):cap(data)])
		data = data[:len(data)+n]
		if err != nil {
			if err == io.EOF {
				err = nil
			}
			return data, err
		}
	}
}

func ReadDir(fsys FS, name string) ([]DirEntry, error) { /* deleted */ }

func Stat(fsys FS, name string) (FileInfo, error) { /* deleted ... */ }

func Sub(fsys FS, dir string) (FS, error) { /* deleted ... */ }
```

这些函数有一个共性，名字和返回值都是和接口方法一样，只是多了一个参数，`fs FS`，我们以`ReadFile`为栗子说明一下。

首先会通过type assert判别fs是不是实现了`ReadFileFS`，如果有就会直接用它的实现返回；否则，就会打开这个文件，得到一个`fs.File`，然后调用`File.Stat`得到这文件的大小，最后将文件的内容*全部*读到内存里返回。

另外几个函数的实现都类似。也就是说，**其实并不需要我们将所有的fs都实现，只需要`fs.FS`的`Open`实现就好了**，只要满足Open返回的文件，是`fs.File`或者`fs.ReadDirFile`接口的实现：

```go
type File interface {
	Stat() (FileInfo, error)
	Read([]byte) (int, error)
	Close() error
}

type ReadDirFile interface {
	File

	ReadDir(n int) ([]DirEntry, error)
}

type DirEntry interface {
	Name() string

	IsDir() bool

	Type() FileMode

	Info() (FileInfo, error)
}
```

`io/fs`里的包函数会通过Open的文件类型来执行对应的文件io。我们可以通过`fstest.TestFS`来验证我们的猜想：

```go
var Fsys fs.FS = new(fsys2)

type fsys2 struct{}

func (f *fsys2) Open(name string) (fs.File, error) {
	if !fs.ValidPath(name) {
		return nil, &fs.PathError{Op: "open", Path: name, Err: os.ErrInvalid}
	}
	return os.Open(name)
}

func TestFs(t *testing.T) {
	if err := fstest.TestFS(Fsys, "fs_test.go"); err != nil {
		t.Fatal(err)
	}
}

// === RUN   TestFs
// --- PASS: TestFs (0.10s)
// PASS
```

no warning, no error. 是的，我们只需要实现Open就足矣。

你可能已经发现了，这个实现，不是和前面的`os.dirFS`一摸一样？！

是的。以上便是整个`io.fs`的设计和实现。

最后在提一下递归遍历目录的写法，以前我们是用`ioutil.WalkDir`，现在可以直接用`fs.WalkDir`，如：

```go
fs.WalkDir(Fsys, "/tmp", func(path string, d fs.DirEntry, err error) error {
	fmt.Println(path)
	return nil
})
```

## 小结
在日常的开发中，我们不要使用具体的fs实现，而是使用其接口`fs.FS`，然后在需要文件内容或者目录等更细的操作时，使用`io/fs`里的包函数，比如`fs.ReadDir`或者`fs.ReadFile`，在初始化的使用，传入`os.DirFS(dir)`给fs，在测试时，则使用`fstest.MapFS`。

比如：

```go
type BussinessLogic struct {
	fs fs.FS
}

func (l *BussinessLogic) doWithFile(name string) error {
	// 读文件
	b, err := fs.ReadFile(l.fs, name)
	// 读目录
	entries, err := fs.ReadDir(l.fs, name)
}

func NewBussinessLogic() *BussinessLogic {
	return &BussinessLogic{fs: os.DirFS("/")} // 注意这里是以根目录为基础的fs，按需初始化
} 

// 单元测试

func TestLogic(t *testing.T) {
	fs := fstest.MapFS{/* 初始化map fs */}
	logic := &BussinessLogic{fs: fs}
	// testing...
	if err := logic.doWithFile("path/to/file"); err != nil {
		t.Fatal(err)
	}
}
```

## EOF
```yaml
summary: go1.16新增了一个包，io/fs，用来统一标准库文件io相关的访问方式。本文通过对比新旧文件io的用法差异，带着问题查看源码：标准库是如何保证兼容性的，如何实现一个基于操作系统中的FS，以及如何做单元测试。
weather: cloudy
license: cc-40-by
location: Tencent Building
background: ./go.jpeg
tags: [go, source, fs]
date: 2021-04-12T20:00:00+08:00
```