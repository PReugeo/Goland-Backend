## edit

`go mod edit [-flag] [go.mod]` 用于编辑 go mod 文件的命令行指令，只负责编辑，一般用于脚本编写 go mod 文件，有以下 flag

```shell
-fmt 格式化 go mod 文件
-module 更改 mod module 路径
-requre=path@version || -droprequire=path 添加或者移出依赖
-exclude=path@version -dropexclude=path@version 添加或移出排除项
-replace=old[@v]=new[@v] -dropreplace=old[@v] 添加或删除一个替代项
// 以上更改会按照顺序更改，且可以重复
-go=version 指定 go 语言版本
-print 打印 go mod 文件
-json 将 go mod 文件打印为 json 格式
```

在源码中 go mod 定义以上这些 flag 的相关信息

```go
// cmd/go/internal/modcmd/edit.go
var (
	editFmt    = cmdEdit.Flag.Bool("fmt", false, "")
	editGo     = cmdEdit.Flag.String("go", "", "")
	editJSON   = cmdEdit.Flag.Bool("json", false, "")
	editPrint  = cmdEdit.Flag.Bool("print", false, "")
	editModule = cmdEdit.Flag.String("module", "", "")
	edits      []func(*modfile.File) // edits specified in flags
)
func init() {
	cmdEdit.Run = runEdit // break init cycle

	cmdEdit.Flag.Var(flagFunc(flagRequire), "require", "")
	cmdEdit.Flag.Var(flagFunc(flagDropRequire), "droprequire", "")
	cmdEdit.Flag.Var(flagFunc(flagExclude), "exclude", "")
	cmdEdit.Flag.Var(flagFunc(flagDropReplace), "dropreplace", "")
	cmdEdit.Flag.Var(flagFunc(flagReplace), "replace", "")
	cmdEdit.Flag.Var(flagFunc(flagDropExclude), "dropexclude", "")

	work.AddModCommonFlags(cmdEdit)
	base.AddBuildFlagsNX(&cmdEdit.Flag)
}
```

在初始化函数和变量中定义了以上的 flag，其中功能读写都注册了相关方法进行处理，而在主函数中依次对各个命令进行了分别处理（省略了相关判断方法）

```go
func runEdit(cmd *base.Command, args []string) {
    anyFlags :=
		*editModule != "" ||
			*editGo != "" ||
			*editJSON ||
			*editPrint ||
			*editFmt ||
			len(edits) > 0
    var gomod string
	if len(args) == 1 {
		gomod = args[0]
	} else {
		gomod = modload.ModFilePath()
	}
    if *editModule != "" || *editGo != "" {
        err := module.CheckImportPath(*editModule) && module.GoVersionRE.MatchString(*editGo)
        handlErr(err)
    }
    
    data, err := lockedfile.Read(gomod)
    modFile, err := modfile.Parse(gomod, data, nil)
    // 省略判空
    modFile.AddModuleStmt(*editModule)
    modFile.AddGoStmt(*editGo)
    for _, edit := range edits {
        edit(modFile)
    }
    
    modFile.SortBlocks()
	modFile.Cleanup() // clean file after edits
    // 打印相关操作
    editPrintJSON(modFile)
    modFile.Format()
	if *editPrint {
		os.Stdout.Write(out)
		return
	}
    // 处理命令行和编辑器同时编辑 go mod 文件情况
    // 锁定 mod 先前的缓存文件
    if unlock, err := modfetch.SideLock(); err == nil {
		defer unlock()
	}

	err = lockedfile.Transform(gomod, func(lockedData []byte) ([]byte, error) {
		if !bytes.Equal(lockedData, data) {
			return nil, errors.New("go.mod changed during editing; not overwriting")
		}
		return out, nil
	})
}
```

​    主函数中，对传入的 tag 进行了判断之后，将需要修改的会在打印前完成，在同时编辑的情况下，会加上 sideLock 来进行判断。

定义了很多对 flag 进行转化的相关方法例如 `parsePathVersion parsePath  parsePathVersionOptional` 等方法，其中核心均调用了 `CheckImportPath` 方法 

```go
// cmd/vendor/golang.org/x/mod/module/module.go
func CheckImportPath(path string) error {
	if err := checkPath(path, false); err != nil {
		return fmt.Errorf("malformed import path %q: %v", path, err)
	}
	return nil
}
// fileName 表示最后一个元素是否是文件
func checkPath(path string, fileName bool) error {
    // 检测 path 是否开头或者皆为错误
    if !utf8.ValidString(path) {
		return fmt.Errorf("invalid UTF-8")
	}
	if path == "" {
		return fmt.Errorf("empty string")
	}
	if path[0] == '-' {
		return fmt.Errorf("leading dash")
	}
	if strings.Contains(path, "//") {
		return fmt.Errorf("double slash")
	}
	if path[len(path)-1] == '/' {
		return fmt.Errorf("trailing slash")
	}
    // 对每个 path 元素均进行判断
    elemStart := 0
	for i, r := range path {
		if r == '/' {
			if err := checkElem(path[elemStart:i], fileName); err != nil {
				return err
			}
			elemStart = i + 1
		}
	}
	return nil
} 

func checkElem(elem string, fileName bool) error {
	if elem == "" {
		return fmt.Errorf("empty path element")
	}
	if strings.Count(elem, ".") == len(elem) {
		return fmt.Errorf("invalid path element %q", elem)
	}
	if elem[0] == '.' && !fileName {
		return fmt.Errorf("leading dot in path element")
	}
	if elem[len(elem)-1] == '.' {
		return fmt.Errorf("trailing dot in path element")
	}
	charOK := pathOK
	if fileName {
		charOK = fileNameOK
	}
	for _, r := range elem {
		if !charOK(r) {
			return fmt.Errorf("invalid char %q", r)
		}
	}
	short := elem
	if i := strings.Index(short, "."); i >= 0 {
		short = short[:i]
	}
	for _, bad := range badWindowsNames {
		if strings.EqualFold(bad, short) {
			return fmt.Errorf("%q disallowed as path element component on Windows", short)
		}
	}
	return nil    
}
```

最终会调用 `checkElem()` 方法，该方法对路径的每一个元素都进行校验判断是否为有效路径，而后会对 windows 系统的路径保留字进行特判。

edit 命令源码中主要是对 mod 文件读写和对写入的 flag 进行判断，真正对依赖进行删除添加的命令是 `go mod tidy`

## tidy

`go mod tidy` 作用为对项目代码中的依赖项进行检测，没使用的进行删除，使用的添加依赖至 `go.mod` 中并进行下载。

初始化方法:

```go
//cmd/go/internal/modcmd/tidy.go
func init() {
	cmdTidy.Run = runTidy // break init cycle
	cmdTidy.Flag.BoolVar(&cfg.BuildV, "v", false, "")
	work.AddModCommonFlags(cmdTidy)
}

func runTidy(cmd *base.Command, args []string) {
	if len(args) > 0 {
		base.Fatalf("go mod tidy: no arguments allowed")
	}

	modload.LoadALL()
	modload.TidyBuildList()
	modTidyGoSum() // updates memory copy; WriteGoMod on next line flushes it out
	modload.WriteGoMod()
}
```

首先对整个 tidy 命令进行了全局注册，在 `runTidy()` 方法中依次运行了 `modload.LoadAll() modLoad.TidyBuildList() modTidyGoSum() modload.WriteGoMod()`  首先对代码中的所有依赖进行检查，加载入一个 List 中，然后调用 TidyBuildList() 对依赖进行检测为最小依赖（删除多余依赖），然后写入 go.sum 文件中进行下载，最后写入 go.mod 文件中。

```go 
//cmd/go/internal/modload/load.go
func LoadALL() []string {
	return loadAll(true)
}
func loadAll(testAll bool) []string {
    InitMod()
    // 读取设置中 build tags
    loaded = newLoader(imports.AnyTags())
	loaded.isALL = true
	loaded.testAll = testAll
	if !testAll {
		loaded.testRoots = true
	}
    all := TargetPackages("...")
	loaded.load(func() []string { return all })
	checkMultiplePaths()
	WriteGoMod()
    // 简化遍历，判断 pkg 有效代码
    paths := append(paths, loaded.(pkgs...).path)
    return paths
}
```

其中主要方法即为 `loaded.load(TargetPackages("..."))` 该方法的工作为加载主模块中的所有包并且对当前模块路径进行检测其是否作用于自身，之后将其写入 `go.mod` 中

```go
//cmd/go/internal/modload/load.go #626
func (ld *loader) load(roots func() []string) {
    // 获取当前 module 的依赖图
    reqs := Reqs()
    // 获取构建列表
    buildsList, err = mvs.BuildList(Target, reqs)
    added := make(map[string]bool)
    for {
        ld.reset()
        if roots != nil {
			for _, path := range roots() {
				ld.work.Add(ld.pkg(path, true))
			}
		}
        // ld.doPkg() 处理每个包内依赖
        ld.work.Do(10, ld.doPkg)
		ld.buildStacks()
        for _, pkg := range ld.pkgs {
            if err, ok := pkg.err.(*ImportMissingError); ok && err.Module.Path != "" {
                ...
				numAdded++
				if !haveMod[err.Module] {
					fmt.Fprintf(os.Stderr, "go: found %s in %s %s\n", pkg.path, err.Module.Path, err.Module.Version)
					haveMod[err.Module] = true
					modAddedBy[err.Module] = pkg
					buildList = append(buildList, err.Module)
				}
				continue
            }
        }

        // 重新计算依赖包情况，在添加依赖后
        reqs = Reqs()
		buildList, err = mvs.BuildList(Target, reqs)
    }
    // 计算直接引用的依赖包
	ld.direct = make(map[string]bool)
	for _, pkg := range ld.pkgs {
		if pkg.mod == Target {
			for _, dep := range pkg.imports {
				if dep.mod.Path != "" {
					ld.direct[dep.mod.Path] = true
				}
			}
		}
	}
	// Add Go versions, computed during walk.
	ld.goVersion = make(map[string]string)
	for _, m := range buildList {
		v, _ := reqs.(*mvsReqs).versions.Load(m)
		ld.goVersion[m.Path], _ = v.(string)
	}
	if !ld.isALL && modFile != nil {
		for _, r := range modFile.Require {
			if !r.Indirect {
				ld.direct[r.Mod.Path] = true
			}
		}
	}    
}
```

整个 tidy 的加载所有包代码基本都在 `load()` 方法中，遍历每个包进行搜索其依赖，然后遍历 `go.mod` 文件，将其 `require` 的包也加入 `ld.direct` map 中，而后其调用 `TidyBUildList()` 从其中获取最小需要依赖，其核心是调用了 `Req()` 方法来进行最小依赖的检测，最后加入到 `load.buildList` 切片中，之后直接遍历该切片将依赖关系写入 go.mod 即可

```go
func TidyBuildList() {
    keep := []module.Version{Target}
	var direct []string
	for _, m := range buildList[1:] {
		if used[m] {
			keep = append(keep, m)
			if loaded.direct[m.Path] {
				direct = append(direct, m.Path)
			}
		} else if cfg.BuildV {
			if _, ok := index.require[m]; ok {
				fmt.Fprintf(os.Stderr, "unused %s\n", m.Path)
			}
		}
	}
	min, err := mvs.Req(Target, direct, &mvsReqs{buildList: keep})    
    buildList = append([]module.Version{Target}, min...)
}
// cmd/go/internal/mvs/mvs.go
func Req(target module.Version, base []string, reqs Reqs) ([]module.Version, error) {
    list, err := BuildList(target, reqs)
	var postorder []module.Version
	reqCache := map[module.Version][]module.Version{}
	reqCache[target] = nil
	var walk func(module.Version) error
    // 添加为较新的 module 依赖至 reqCache 和 postorder 中
	walk = func(m module.Version) error {
		_, ok := reqCache[m]
		if ok {
			return nil
		}
		required, err := reqs.Required(m)
		if err != nil {
			return err
		}
		reqCache[m] = required
		for _, m1 := range required {
			if err := walk(m1); err != nil {
				return err
			}
		}
		postorder = append(postorder, m)
		return nil
	}
    // 对 buildList 中的每条依赖均进行检查
    for _, m := range list {
        if err := walk(m); err != nil {
            return nil, err
        }
    }
    // 进行后序遍历，仅添加尚未隐含的    
	have := map[module.Version]bool{}
	walk = func(m module.Version) error {
		if have[m] {
			return nil
		}
		have[m] = true
		for _, m1 := range reqCache[m] {
			walk(m1)
		}
		return nil
	}
	max := map[string]string{}
	for _, m := range list {
		if v, ok := max[m.Path]; ok {
			max[m.Path] = reqs.Max(m.Version, v)
		} else {
			max[m.Path] = m.Version
		}
	}
    // 首次遍历添加必须依赖
	var min []module.Version
	for _, path := range base {
		m := module.Version{Path: path, Version: max[path]}
		min = append(min, m)
		walk(m)
	}    
    // 逆向遍历，添加别的依赖
	for i := len(postorder) - 1; i >= 0; i-- {
		m := postorder[i]
		if max[m.Path] != m.Version {
			// Older version.
			continue
		}
		if !have[m] {
			min = append(min, m)
			walk(m)
		}
	}
	sort.Slice(min, func(i, j int) bool {
		return min[i].Path < min[j].Path
	})
	return min, nil    
}
```

 整个 tidy 操作通过， `buildList` 获取当前 module 项目中的必须依赖，通过比对版本以及实际使用的依赖来修改（添加，删除依赖） go.mod 文件，完成整个 tidy 操作。

