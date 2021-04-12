# Go 1.16 fs 设计与实现

## 摘要
go1.16新增了一个包，`io/fs`，用来统一标准库文文件相关的访问方式。本文通过查看`io/fs`的源码，标准库是如何保证兼容性的，如何实现一个基于操作系统中的FS，以及如何做文件相关的单元测试，来带大家熟悉go文件操作的正确姿势。

## 背景
最近做的一个项目需要频繁和文件系统打交道，在本地跑单元测试。刚好近期go更新到了1.16，新增了`io/fs`包，抽象了统一的文件访问相关接口，对测试也更友好，于是在项目中实践了以下，体验确实不错。

为了让大家更好的了解它，使用它，今天我们来从聊聊它的设计与实现。

下一篇会通过`fs.FS`接口来设计和实现一个`cosFS`，敬请期待。

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

这是最简单的形式，通过名字打开一个*文件*，符合go的最小接口原则，后面可以组合成更复杂的FS。FS只有一个方法，打开一个文件，注意这个`fs.File`不是我们熟悉的那个`os.File`，而是一个接口，后者*实现*了前者。这接口实际上是`io.Reader`和`io.Closer`的组合。所以如果需要访问操作系统的文件，这接口的实现可以是这样的：

```go
type fsys struct {}

func (*fsys) Open(name string) (fs.File, error) {
    return os.ReadFile(name)
}
```

但是如果就只能打开一个文件，似乎也没什么x用（后面会打脸），如果要读目录呢？实际上`io/fs`已经提供了几种常用的FS：

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

依次是用来读整个文件到内存，读目录和glob文件名匹配。`io/fs`包里面都是接口，没有实现，但是标准库里文件相关的操作都已经实现了这些接口，比如，一个通用的`fsys`可以是这样的：

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

可以看到函数名，返回值都是一毛一样的！就是包不同，正如go说的，标准库都实现了这些接口，我们完全可以不用理会这个新的包，直接使用具体的实现。

> 注意到`ReadFile()`是不是似曾相识？对，就是以前常用的那个`io/ioutil.ReadFile()`，现在被移到到了io包中；同理`ioutil.ReadDir`也移到了`os`包里

注意到在go1.15中，[函数](https://github.com/golang/go/blob/go1.15.11/src/io/ioutil/ioutil.go#L93)签名是：

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

// A FileMode represents a file's mode and permission bits.
// The bits have the same definition on all systems, so that
// information about files can be moved from one system
// to another portably. Not all bits apply to all systems.
// The only required bit is ModeDir for directories.
type FileMode = fs.FileMode
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

这。。。不是把代码**剪切**到了另一个包下，然后利用了*type alias*的特性保证了兼容性，旧版本的类型名字没有变化，实际类型已经指向了`io/fs`包的类型，而`fs`包里的。这就是为什么标准库是都如何实现了这些`io/fs`接口的的，我们其实也可以利用这方式来实现接口的平滑升级而不会造成不兼容。

那么搞出来这个意义何在呢？我觉得主要有两方面的原因：

- 屏蔽了fs具体的实现，我们可以按照需要替换成不同的文件系统实现，比如内存/磁盘/网络/分布式的文件系统
- 易于单元测试，实际也是上一点的体现

这比在代码里硬编码`os.Open`要更灵活，如果代码中全是`os`包，则将实现完全绑定在了操作系统的文件系统，一个最直接的问题是单元测试比较难做（好在go可以使用./testdata将测试文件打包在代码里）。但这本质上还是*集成测试*而非*单元测试*，因为单元测试不依赖外部系统全在内存里搞，比如时间和OS。我们需要一个mock的fs来做单元测试，把fs抽象成接口就好了，这也是`io.fs`这个包最大的作用吧，单元测试是接口第一个使用者。

我们来看一个具体的栗子。

## 食用栗子

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
	finder := Finder{fs: &Fsys{}}
	ok, err := finder.fileContains666("/path/to/file")
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

我们的栗子比较玩具，就是业务逻辑的结构体中增加了一个`fs`接口对象，该对象提供了读一个文件的功能，至于具体怎么读，Finder并不关心，它只关心能搞到文件的字节数组，然后看看里面是否包含666。正常使用的时候只需要在初始化时传一个操作系统的实现，也就是上一节中的`Fsys`就好了。

我们比较关心单元测试的实现，如何不依赖操作系统的文件系统来做呢？Go1.16提供了一个`fstest.MapFS`的fs的[实现](https://github.com/golang/go/blob/go1.16.3/src/testing/fstest/mapfs.go#L33)：

```go
// A MapFS is a simple in-memory file system for use in tests,
// represented as a map from path names (arguments to Open)
// to information about the files or directories they represent.
//
// The map need not include parent directories for files contained
// in the map; those will be synthesized if needed.
// But a directory can still be included by setting the MapFile.Mode's ModeDir bit;
// this may be necessary for detailed control over the directory's FileInfo
// or to create an empty directory.
//
// File system operations read directly from the map,
// so that the file system can be changed by editing the map as needed.
// An implication is that file system operations must not run concurrently
// with changes to the map, which would be a race.
// Another implication is that opening or reading a directory requires
// iterating over the entire map, so a MapFS should typically be used with not more
// than a few hundred entries or directory reads.
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

除此之外，go1.16在fs的测试上还提供了一个方法，`TestFS`，看看[它](https://github.com/golang/go/blob/go1.16.3/src/testing/fstest/testfs.go#L38)，它可以用来检测一个文件系统的实现是否满足要求（walk当前及子目录，打开所有的文件检测行为是否满足标准），是否在文件系统中存在：

```go
// TestFS tests a file system implementation.
// It walks the entire tree of files in fsys,
// opening and checking that each file behaves correctly.
// It also checks that the file system contains at least the expected files.
// As a special case, if no expected files are listed, fsys must be empty.
// Otherwise, fsys must only contain at least the listed files: it can also contain others.
// The contents of fsys must not change concurrently with TestFS.
//
// If TestFS finds any misbehaviors, it returns an error reporting all of them.
// The error text spans multiple lines, one per detected misbehavior.
//
// Typical usage inside a test is:
//
//	if err := fstest.TestFS(myFS, "file/that/should/be/present"); err != nil {
//		t.Fatal(err)
//	}
//
func TestFS(fsys fs.FS, expected ...string) error { /* deleted */ }
```

我们可以拿上一节的`Fsys`来试试，

```go
if err := fstest.TestFS(new(fsys), "fsys_test.go"); err != nil {
	t.Fatal(err)
}
// TestFS found errors:
//     .: Open(/.) succeeded, want error
//     .: Open(./.) succeeded, want error
//     .: Open(/) succeeded, want erro
```

可以看到出错了，原因是我们没有按照`fs.FS`接口contract里面要求那样，去验证传入的path的合法性：

> When Open returns an error, it should be of type *PathError with the Op field set to "open", the Path field set to name, and the Err field describing the problem.

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

## 包方法
注意到，`io/fs`包里面，每个接口，同时还包含了package的方法：

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

首先会通过type assert判别fs是不是实现了`ReadFileFS`，如果有就会直接用它的实现返回；否则，就会打开这个文件，得到一个`fs.File`，然后调用`File.Stat`得到这文件的大小，最后将文件的内容**全部**读到内存里返回。

另外几个函数都类似，也就是说，其实并不需要我们将所有的fs都实现，只需要`fs.FS`的`Open`实现就好了，我们可以通过`fstest.TestFS`来验证我们的猜想：

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

最后在提一下递归遍历目录的写法，以前我们是用`ioutil.WalkDir`，现在可以直接用`fs.WalkDir`，如：

```go
fs.WalkDir(Fsys, "/tmp", func(path string, d fs.DirEntry, err error) error {
	fmt.Println(path)
	return nil
})
```

## EOF
```yaml
summary: 本文通过查看`io/fs`的源码，标准库是如何保证兼容性的，如何实现一个基于操作系统中的FS，以及如何做文件相关的单元测试，来带大家熟悉go文件操作的正确姿势。
weather: cloudy
license: cc-40-by
location: Tencent Building
background: ./go.jpeg
tags: [go, source]
date: 2021-04-12T20:00:00+08:00
```