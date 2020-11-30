# 基本使用

GOMODULE 是 go1.11 的新特性：

模块是相关 Go 包的集合。modules 是源代码交换和版本控制的单元。 go 命令直接支持使用 modules，包括记录和解析对其他模块的依赖性。modules 替换旧的基于GOPATH的方法来指定在给定构建中使用哪些源文件。

如果使用官网进行 mod 下载会很慢，goproxy 提供了一个代理下载，使其在国内也能很快的下载 go 的依赖包

```shell
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.io,direct

# 设置不走 proxy 的私有仓库，多个用逗号相隔（可选）
go env -w GOPRIVATE=*.corp.example.com

# 设置不走 proxy 的私有组织（可选）
go env -w GOPRIVATE=example.com/org_name
```

go mod 的具体用法可以使用 `go help mod ` 获取 go 官方的帮助信息

```bash
$ go help mod
Go mod provides access to operations on modules.

Note that support for modules is built into all the go commands,
not just 'go mod'. For example, day-to-day adding, removing, upgrading,
and downgrading of dependencies should be done using 'go get'.
See 'go help modules' for an overview of module functionality.

Usage:

	go mod <command> [arguments]

The commands are:

	download    download modules to local cache(下载依赖包)
	edit        edit go.mod from tools or scripts（编辑 go.mod）
	graph       print module requirement graph （打印模块依赖图）
	init        initialize new module in current directory（在当前目录初始化新的模块）
	tidy        add missing and remove unused modules（更新 go.mod 拉取依赖取出不需要的依赖）
	vendor      make vendored copy of dependencies （将依赖复制到 vendor 文件夹下）
	verify      verify dependencies have expected content（验证依赖正确性）
	why         explain why packages or modules are needed（解释为什么需要该依赖）

Use "go help mod <command>" for more information about a command.
```

Go mod 下载的包默认会存在 `$GOPATH/pkg/mod` 中，多个项目可以共享 mod 库。

> 在需要更新依赖或者可以手动添加依赖或者升降级依赖时，可以使用`go get` 命令

mod 语法中有 `require` `replace ` 语法，其中第一个为依赖声明，第二个可以将依赖源或者版本进行替换，使项目中使用不同一个版本的依赖。



# 源码分析

首先是 `go mod help` 页面初始化代码

```go
// cmd/go/internal/modcmd/mod.go #12-33
var CmdMod = &base.Command{
	UsageLine: "go mod",
	Short:     "module maintenance",
	Long: `Go mod provides access to operations on modules.

Note that support for modules is built into all the go commands,
not just 'go mod'. For example, day-to-day adding, removing, upgrading,
and downgrading of dependencies should be done using 'go get'.
See 'go help modules' for an overview of module functionality.
	`,

	Commands: []*base.Command{
		cmdDownload,
		cmdEdit,
		cmdGraph,
		cmdInit,
		cmdTidy,
		cmdVendor,
		cmdVerify,
		cmdWhy,
	},
}
```

在 `CmdMod` 中定义了初始化打印的语句，和 mod 相关的操作命令定义在 `CmdMod.Commands` 中。

## Init

`mod init` 命令相关定义在 `// cmd/go/internal/modcmd/init.go` 文件中

```go
// cmd/go/internal/modcmd/init.go #35-54
func runInit(cmd *base.Command, args []string) {
	modload.CmdModInit = true
	if len(args) > 1 {
		base.Fatalf("go mod init: too many arguments")
	}
	if len(args) == 1 {
		modload.CmdModModule = args[0]
	}
	if os.Getenv("GO111MODULE") == "off" {
		base.Fatalf("go mod init: modules disabled by GO111MODULE=off; see 'go help modules'")
	}
	modFilePath := modload.ModFilePath()
    // 检测 module 文件是否存在
	if _, err := os.Stat(modFilePath); err == nil {
		base.Fatalf("go mod init: go.mod already exists")
	}
    // 检测 module path 中是否带了 @符号
	if strings.Contains(modload.CmdModModule, "@") {
		base.Fatalf("go mod init: module path must not contain '@'")
	}
	modload.InitMod() // does all the hard work
}
```

这个函数就是在执行 `go mod init [modulename]` 之后执行的操作，就是对参数进行一些判断，不通过就是通过自己封装的 `base.Fatalf` 停止操作并报错

```go
// cmd/go/internal/base/base.go
func Fatalf(format string, args ...interface{}) {
	Errorf(format, args...)
	Exit()
}

func Exit() {
	for _, f := range atExitFuncs {
		f()
	}
	os.Exit(exitStatus)
}
```

这个函数会将所有操作都遍历退出。在检测 mod 文件存在时使用了 `modFilePath := modload.ModFilePath()` 来获取 mod 文件的路径

```go
// cmd/go/internal/modload/init.go
func ModFilePath() string {
	if !HasModRoot() {
		die()
	}
	if cfg.ModFile != "" {
		return cfg.ModFile
	}
	return filepath.Join(modRoot, "go.mod")
}
func HasModRoot() bool {
	Init()
	return modRoot != ""
}
```

其中 `Init ` 函数就是真正的初始化判断 mod 文件使用，其作用为判断系统变量 `GO111MODULE` 以供使用

```go
// cmd/go/internal/modload/init.go #108
func Init() {
    // 判断 GO111MODULE 环境变量
    env := cfg.Getenv("GO111MODULE")
	switch env {
	default:
		base.Fatalf("go: unknown environment setting GO111MODULE=%s", env)
	case "auto", "":
		mustUseModules = false
	case "on":
		mustUseModules = true
	case "off":
		mustUseModules = false
		return
	}
    // 对 git 环境变量进行判断
    if os.Getenv("GIT_TERMINAL_PROMPT") == "" {
		os.Setenv("GIT_TERMINAL_PROMPT", "0")
	}
    if os.Getenv("GIT_SSH") == "" && os.Getenv("GIT_SSH_COMMAND") == "" {
		os.Setenv("GIT_SSH_COMMAND", "ssh -o ControlMaster=no")
	}
    // 如果执行 go mod init 语句直接在当前文件夹创建 mod 文件，否则找寻 mulde root 目录进行创建
    if CmdModInit {
        modRoot = base.Cwd
    } else {
        modRoot = findModuleRoot(base.Cwd)
        ...
    }
    // 当指定 -modfile=... 时，对 modfile 是否存在进行判断
    if cfg.ModFile != "" && !strings.HasSuffix(cfg.ModFile, ".mod") {
		base.Fatalf("go: -modfile=%s: file does not have .mod extension", cfg.ModFile)
	}
    // 判断是否可以使用缓存
    if c := cache.Default(); c == nil {
		// With modules, there are no install locations for packages
		// other than the build cache.
		base.Fatalf("go: cannot use modules with build cache disabled")
	}
    list := filepath.SplitList(cfg.BuildContext.GOPATH)
	if len(list) == 0 || list[0] == "" {
		base.Fatalf("missing $GOPATH")
	}
	gopath = list[0]
	if _, err := os.Stat(filepath.Join(gopath, "go.mod")); err == nil {
		base.Fatalf("$GOPATH/go.mod exists but should not")
	}
    ...
    // 将老文件夹中包转移到新的文件夹中
    oldSrcMod := filepath.Join(list[0], "src/mod")
	pkgMod := filepath.Join(list[0], "pkg/mod")
    infoOld, errOld := os.Stat(oldSrcMod)
	_, errMod := os.Stat(pkgMod)
	if errOld == nil && infoOld.IsDir() && errMod != nil && os.IsNotExist(errMod) {
		os.Rename(oldSrcMod, pkgMod)
	}
    ...
    if modRoot == "" {
        
    } else {
        modfetch.GoSumFile = strings.TrimSuffix(ModFilePath(), ".mod") + ".sum"
		search.SetModRoot(modRoot)
    }
}
```

在源码中还有一些初始化的过程被我省略了，主要看下源码中对于 mod 文件存在和系统变量的判断逻辑。

在初始化最后的 `InitMod` 函数也是调用了该 `Init（）` 函数，以及对 `cfg.BuildMod` 进行初始化

```go
// cmd/go/internal/modload/init.go #378
func InitMod() {
    Init()
	if modRoot == "" {
		Target = module.Version{Path: "command-line-arguments"}
		targetPrefix = "command-line-arguments"
		buildList = []module.Version{Target}
		return
	}
    if CmdModInit {
		// Running go mod init: do legacy module conversion
		legacyModInit()
		modFileToBuildList()
		WriteGoMod()
		return
	}
    gomod := ModFilePath()
	data, err := lockedfile.Read(gomod)
    if len(f.Syntax.Stmt) == 1 && f.Module != nil {
		// Entire file is just a module statement.
		// Populate require if possible.
		legacyModInit()
	}

	modFileToBuildList()
	setDefaultBuildMod()
    if cfg.BuildMod == "vendor" {
		readVendorList()
		checkVendorConsistency()
	} else {
		// TODO(golang.org/issue/33326): if cfg.BuildMod != "readonly"?
		WriteGoMod()
	}
}
```

可以看到，对于初始化 mod 文件使用 `go mod init` 后会直接对其进行写入文件，否则会进行 mod 文件的一些初始化工作然后写入文件中。初始化工作包括对过去 mod 文件的向下兼容（vendor，godep 等以前的管理工具包）适配 `legacyModInit` ，加入包的编译工作 `modFileToBuildList` 和写入 mod 文件 `WriteGoMod` 三个主要函数。

以上就是 `go mod init` 主要会进行的一些操作，主要就是判断 mod 文件会否存在和对以前的包管理机制做一些兼容。

## download

`go mod download` 的命令，用于下载依赖至 `$GOPATH/pkg/mod`，其核心的结构体为 `moduleJSON` 可以直接导入 json 文件作为下载

```go
// cmd/go/internal/modcmd/download.go #68
type moduleJSON struct {
	Path     string `json:",omitempty"`
	Version  string `json:",omitempty"`
	Error    string `json:",omitempty"`
	Info     string `json:",omitempty"`
	GoMod    string `json:",omitempty"`
	Zip      string `json:",omitempty"`
	Dir      string `json:",omitempty"`
	Sum      string `json:",omitempty"`
	GoModSum string `json:",omitempty"`
}
```

其核心初始化函数和 init 很像，函数为 `runDownload`

```go
// cmd/go/internal/modcmd/download.go #80
func runDownload(cmd *base.Command, args []string) {
    ...
    // 读取 json 文件或者是 -x 命令中的相关参数并进行下载
	var mods []*moduleJSON
	var work par.Work   
    for _, info := range modload.ListModules(args, listU, listVersions) {
    	m := &moduleJSON{
			Path:    info.Path,
			Version: info.Version,
		}
		mods = append(mods, m)
		if info.Error != nil {
			m.Error = info.Error.Err
			continue
		}
		work.Add(m)    
    }
    work.Do(10, func(item inferface{}) {
        m := item.(*moduleJSON)
        // initialization 为缩减源码，源码为
        // m.Info, er := modfetch.InfoFile(m.Path, m.Version)
        initialization(m.Info, m.GoMod, m.GoModSum, m.Zip, m.Sum)
        m.Dir, err = modfetch.Download(mod)
		if err != nil {
			m.Error = err.Error()
			return
		}
    })
    
    // 对 json 下载做处理
    if *downloadJSON {
		for _, m := range mods {
			b, err := json.MarshalIndent(m, "", "\t")
			if err != nil {
				base.Fatalf("%v", err)
			}
			os.Stdout.Write(append(b, '\n'))
			if m.Error != "" {
				base.SetExitStatus(1)
			}
		}
	} else {
		for _, m := range mods {
			if m.Error != "" {
				base.Errorf("%s", m.Error)
			}
		}
		base.ExitIfErrors()
	}
}
```

​    对于上面函数可以首先先忽略对 mod 文件的校验相关代码，看下载模块对 download 的初始化工作

```go
// cmd/go/internal/modcmd/download.go #107
for _, info := range modload.ListModules(args, listU, listVersions) {
    m := &moduleJSON{
        Path:    info.Path,
        Version: info.Version,
    }
    mods = append(mods, m)
    work.Add(m)
}
// cmd/go/internal/modload/list.go #22
func ListModules(args []string, listU, listVersion bool) []*modinfo.ModulePublic {
    mods := listModules(args, listVersions)
    if listU || listVersions {
        ...
        if m.Replace != nil {
            work.Add(m.Replace)
        }
        if listU {
            addUpdate(m)
        }
        if listVersions {
            addVersions(m)
        }
    }
    return mods
}
```

可以从代码看出，通过遍历来判断出 download 中的更新包以及需要下载或者替换的包来对包进行操作。

在代码中有个 `work` 的结构体频繁使用，其是 `par.Work` 是真正来执行 download 操作的函数，前面代码都是批量加入到 `work` 中，最后通过 `work.Do()` 来执行下载操作。

```go
// cmd/go/internal/par/work.go
// work 中的函数都是并行执行的，每个最多执行一次
type Work struct {
	f       func(interface{}) // function to run for each item
	running int               // total number of runners

	mu      sync.Mutex
	added   map[interface{}]bool // items added to set
	todo    []interface{}        // items yet to be run
	wait    sync.Cond            // wait when todo is empty
	waiting int                  // number of runners waiting for todo
}
// 添加条目进入 work.added map 中
func (w *Work) Add(item interface{}) {
	w.mu.Lock()
	w.init()
	if !w.added[item] {
		w.added[item] = true
		w.todo = append(w.todo, item)
		if w.waiting > 0 {
			w.wait.Signal()
		}
	}
	w.mu.Unlock()
}
// Do 执行对 added map 中每个条目执行 f 函数
func (w *Work) Do(n int, f func(item interface{})) {
	if n < 1 {
		panic("par.Work.Do: n < 1")
	}
	if w.running >= 1 {
		panic("par.Work.Do: already called Do")
	}

	w.running = n
	w.f = f
	w.wait.L = &w.mu

	for i := 0; i < n-1; i++ {
		go w.runner()
	}
	w.runner()
}
```

而 `work.Do()` 函数最终会开启数个 gorountine 并发执行 `w.runner()`  

```go
func (w *Work) runner() {
	for {
		// Wait for something to do.
		w.mu.Lock()
		for len(w.todo) == 0 {
			w.waiting++
			if w.waiting == w.running {
				// All done.
				w.wait.Broadcast()
				w.mu.Unlock()
				return
			}
			w.wait.Wait()
			w.waiting--
		}
		i := rand.Intn(len(w.todo))
		item := w.todo[i]
		w.todo[i] = w.todo[len(w.todo)-1]
		w.todo = w.todo[:len(w.todo)-1]
		w.mu.Unlock()

		w.f(item)
	}
}
```

`w.runner()` 方法建立了一个大循环，在其中循环等待加入事件，如果有事件则随机进行执行。

Download 中还有一个下载函数 `modfetch.Download()`

```go
// cmd/go/internal/modfetch/fetch.go
func Download(mod module.Version) (dir string, err error) {
    type cached struct {
        dir string
        err error
    }
    c := downloadCache.Do(mod, func() interface{} {
		dir, err := download(mod)
		if err != nil {
			return cached{"", err}
		}
		checkMod(mod)
		return cached{dir, nil}
	}).(cached)
	return c.dir, c.err
}

func download(mod module.Version) (dir string, err error) {
    dir, err = DownloadDir(mod)
    zipfile, err := DownloadZip(mod)
    
    unlock, err := lockVersion(mod)
	if err != nil {
		return "", err
	}
	defer unlock()
    parentDir := filepath.Dir(dir)
	tmpPrefix := filepath.Base(dir) + ".tmp-"
    err := os.MkdirAll(parentDir, 0777)
    
    tmpDir, err := ioutil.TempDir(parentDir, tmpPrefix)
	if err != nil {
		return "", err
	}
	defer func() {
		if err != nil {
			RemoveAll(tmpDir)
		}
	}()
    err := modzip.Unzip(tmpDir, mod, zipfile)
    return dir, nil
}
```

为了避免混淆文件，download 方法在执行了 `DownloadDir() DownloadZip()` 后才会将 version 进行锁定，之后就是创建文件夹，存放包依赖文件和删除 temp 文件夹的操作。

download 命令在源码中，主要通过了多协程来加快下载进程，并且对将包依赖存放在 mod cache 文件夹中。