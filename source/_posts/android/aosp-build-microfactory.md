---
title: AOSP Microfactory
tags:
  - aosp
cover: >-
  https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250913190808770.png
date: 2025-09-13 19:09:06
---




# Microfactory

基于Android14.0.0_r73



简单介绍：

Microfactory是aosp的一个go语言编写的构建工具，且主要用于对go编译进行缓存，实现增量编译的效果。



# 源码



microfactory工具通过source build/blueprint/microfactory/microfactory.bash配置。

配置好以后shell中可以通过调用build_go函数触发并使用microfactory进行增量编译。

```shell
# build/blueprint/microfactory/microfactory.bash
function build_go
{
    # Increment when microfactory changes enough that it cannot rebuild itself.
    # For example, if we use a new command line argument that doesn't work on older versions.
    # 标记microfactory的版本
    local mf_version=3
	# microfactory路径，默认在 $topdir/build/blueprint/microfactory
    local mf_src="${BLUEPRINTDIR}/microfactory"
	# microfactory执行最初是源码执行，但是执行过程中会生成二进制文件。
	# 如果生成二进制文件则优先使用二进制文件
	# 其中二进制文件的路径为：$topdir/out/microfactory_${uname}
    local mf_bin="${BUILDDIR}/microfactory_$(uname)"
    local mf_version_file="${BUILDDIR}/.microfactory_$(uname)_version"
    # 需要构建生成的二进制文件
    # 外部会通过 调用build_go cmd 生成对于的二进制产物
    local built_bin="${BUILDDIR}/$1"
    local from_src=1

    if [ -f "${mf_bin}" ] && [ -f "${mf_version_file}" ]; then
        if [ "${mf_version}" -eq "$(cat "${mf_version_file}")" ]; then
            from_src=0
        fi
    fi

    local mf_cmd
    # 使用源码运行
    if [ $from_src -eq 1 ]; then
        # `go run` requires a single main package, so create one
        # go run需要一个main package,这里需要通过sed进行文件替换
        local gen_src_dir="${BUILDDIR}/.microfactory_$(uname)_intermediates/src"
        mkdir -p "${gen_src_dir}"
        # 将文件的package microfactory 替换为 package main， 并保存到gen_src_dir
        sed "s/^package microfactory/package main/" "${mf_src}/microfactory.go" >"${gen_src_dir}/microfactory.go"
        # 末尾添加main函数。
        printf "\n//for use with go run\nfunc main() { Main() }\n" >>"${gen_src_dir}/microfactory.go"

        mf_cmd="${GOROOT}/bin/go run ${gen_src_dir}/microfactory.go"
    else
        mf_cmd="${mf_bin}"
    fi

    rm -f "${BUILDDIR}/.$1.trace"
    # GOROOT must be absolute because `go run` changes the local directory
    # 进入GOROOT 运行microfactory
    # 添加参数： 
    # -b 
    # -pkg-path 
    # -trimpath
    # ${EXTRA_ARGS}
    # -o 
    # pkgName
    GOROOT=$(cd $GOROOT; pwd) ${mf_cmd} -b "${mf_bin}" \
            -pkg-path "github.com/google/blueprint=${BLUEPRINTDIR}" \
            -trimpath "${SRCDIR}" \
            ${EXTRA_ARGS} \
            -o "${built_bin}" $2

    if [ $? -eq 0 ] && [ $from_src -eq 1 ]; then
        echo "${mf_version}" >"${mf_version_file}"
    fi
}
```



通过上述的分析可以发现microfactory的源码存储在$topdir/build/blueprint/microfactory不过实际在编译的时候会使用sed做一些简单的替换。

所以实际的microfactory的源码可以通过查找$topdir/out/.microfactory\_$(uname)_intermediates/src 路径下查找



# 关于参数



microfactory作为一个构建工具，自然是可以传参的，那么参数有那些呢？可见下方

```go
// 编译时是否开启竞态检测器，对于于编译器的-race参数
flags.BoolVar(&config.Race, "race", false, "enable data race detection.")
// 是否开启详细输出，开启后会输出过程日志信息
flags.BoolVar(&config.Verbose, "v", false, "Verbose")
// 编译后的产物文件
flags.StringVar(&output, "o", "", "Output file")
// Microfactory 二进制文件路径(首次使用会将microfactory编译成二进制)
flags.StringVar(&mybin, "b", "", "Microfactory binary location")
// 对应编译器的-trimpath 参数，用于移除源码路径前缀
flags.StringVar(&config.TrimPath, "trimpath", "", "remove prefix from recorded source file paths")
// 对于于config.Path，方便程序解析依赖包的地址
flags.Var(&pkgMap, "pkg-path", "Mapping of package prefixes to file paths")
```







# 源码分析



- 模板路径：

$topdir/build/blueprint/microfactory/microfactory.go

- 源码路径：

$topdir/out/.microfactory_Linux_intermediates/src/microfactory.go



## main



没什么好讲的

```go
// 这段代码通过shell追加的
func main() { Main() }
```



1.解释二进制参数

2.准备脚本运行时环境

3.将microfactory方法构建为二进制文件(构建自身的过程中会构建传入的源文件)

4.构建传入的参数信息

```go
func Main() {
    // 创建了几个数据类
	var output, mybin string
	var config Config
	pkgMap := pkgPathMappingVar{&config}
	// 解析参数
	flags := flag.NewFlagSet("", flag.ExitOnError)
	flags.BoolVar(&config.Race, "race", false, "enable data race detection.")
	flags.BoolVar(&config.Verbose, "v", false, "Verbose")
	flags.StringVar(&output, "o", "", "Output file")
	flags.StringVar(&mybin, "b", "", "Microfactory binary location")
	flags.StringVar(&config.TrimPath, "trimpath", "", "remove prefix from recorded source file paths")
	flags.Var(&pkgMap, "pkg-path", "Mapping of package prefixes to file paths")
	err := flags.Parse(os.Args[1:])
	// 打印help信息
	if err == flag.ErrHelp || flags.NArg() != 1 || output == "" {
		fmt.Fprintln(os.Stderr, "Usage:", os.Args[0], "-o out/binary <main-package>")
		flags.PrintDefaults()
		os.Exit(1)
	}
	// trace文件路径
	tracePath := filepath.Join(filepath.Dir(output), "."+filepath.Base(output)+".trace")
    // 创建traceFile并将TraceFuc设置到了config中，估计是用来写日志的。
	if traceFile, err := os.OpenFile(tracePath, os.O_RDWR|os.O_CREATE|os.O_APPEND, 0666); err == nil {
		defer traceFile.Close()
		config.TraceFunc = func(name string) func() {
			fmt.Fprintf(traceFile, "%d B %s\n", time.Now().UnixNano()/1000, name)
			return func() {
				fmt.Fprintf(traceFile, "%d E %s\n", time.Now().UnixNano()/1000, name)
			}
		}
	}
    // 获取当前可执行文件的路径
	if executable, err := os.Executable(); err == nil {
		defer un(config.trace("microfactory %s", executable))
	} else {
		defer un(config.trace("microfactory <unknown>"))
	}
	// 如果通过-b传入了构建的路径，先进行一次check是否需要重建
    // 由于构建自身的过程中会构建output对于的二进制文件，所以如果触发rebuild后续的代码不会触发执行。
	if mybin != "" {
		if rebuildMicrofactory(&config, mybin) {
			return
		}
	}
	// 执行build
	if _, err := Build(&config, output, flags.Arg(0)); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```



## rebuildMicrofactory



首先需要说明一下microfactory是用来做源码的**增量编译**的，他是用于**构建源码**的工具

但是他在构建我们传入的源码**前**会先**构建自身**。而构建自身的逻辑就在如下rebuildMicrofactory中。

这个方法最核心的逻辑就是Build(config, mybin, "github.com/google/blueprint/microfactory/main")，接下来对Build方法进行分析

```go
// rebuildMicrofactory checks to see if microfactory itself needs to be rebuilt,
// and if does, it will launch a new copy and return true. Otherwise it will return
// false to continue executing.
func rebuildMicrofactory(config *Config, mybin string) bool {
    // 构建自身
	if pkg, err := Build(config, mybin, "github.com/google/blueprint/microfactory/main"); err != nil {
        // 如果构建失败
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	} else if !pkg.rebuilt {
        // 如果构建过程中没有出问题，且发现当前已经是构建过的状态，并且不需要继续构建，直接返回。
		return false
	}
	// 使用二进制文件重新执行脚本
	cmd := exec.Command(mybin, os.Args[1:]...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
    // 如果执行结果正常，至二级返回
	if err := cmd.Run(); err == nil {
		return true
	} else if e, ok := err.(*exec.ExitError); ok {
        // 使用子命令的返回码直接返回
		os.Exit(e.ProcessState.Sys().(syscall.WaitStatus).ExitStatus())
	}
    // 如果没有返回码直接exit(1)
	os.Exit(1)
	return true
}
```



## Build



```go
func Build(config *Config, out, pkg string) (*GoPackage, error) {
    // 创建一个package
	p := &GoPackage{
		Name: "main",
	}
	// 获取锁文件
	lockFileName := filepath.Join(filepath.Dir(out), "."+filepath.Base(out)+".lock")
	lockFile, err := os.OpenFile(lockFileName, os.O_RDWR|os.O_CREATE, 0666)
	if err != nil {
		return nil, fmt.Errorf("Error creating lock file (%q): %v", lockFileName, err)
	}
	defer lockFile.Close()
	// 加个文件锁
	err = syscall.Flock(int(lockFile.Fd()), syscall.LOCK_EX)
	if err != nil {
		return nil, fmt.Errorf("Error locking file (%q): %v", lockFileName, err)
	}
	// 获取package路径， config中存储了路径映射关系
	path, ok, err := config.Path(pkg)
	if err != nil {
		return nil, fmt.Errorf("Error finding package %q for main: %v", pkg, err)
	}
	if !ok {
		return nil, fmt.Errorf("Could not find package %q", pkg)
	}
	// 拼接输出路径
	intermediates := filepath.Join(filepath.Dir(out), "."+filepath.Base(out)+"_intermediates")
    // 创建文件夹路径
	if err := os.MkdirAll(intermediates, 0777); err != nil {
		return nil, fmt.Errorf("Failed to create intermediates directory: %v", err)
	}
	// 寻找依赖项
	if err := p.FindDeps(config, path); err != nil {
		return nil, fmt.Errorf("Failed to find deps of %v: %v", pkg, err)
	}
    // 执行编译
	if err := p.Compile(config, intermediates); err != nil {
		return nil, fmt.Errorf("Failed to compile %v: %v", pkg, err)
	}
    // 执行链接
	if err := p.Link(config, out); err != nil {
		return nil, fmt.Errorf("Failed to link %v: %v", pkg, err)
	}
	return p, nil
}
```



### config.Path



这个方法是用来做package路径查找的，这里会输入一个go的包名，但是会返回一个包名所处路径。

值得注意的是这里的映射关系(即c.paths)是通过解析shell参数获得的。

```go
// Path takes a package name, applies the path mappings and returns the resulting path.
//
// If the package isn't mapped, we'll return false to prevent compilation attempts.
func (c *Config) Path(pkg string) (string, bool, error) {
	if c == nil || c.paths == nil {
		return "", false, fmt.Errorf("No package mappings")
	}

	for _, pkgPrefix := range c.pkgs {
        // 如果相等直接返回value
		if pkg == pkgPrefix {
			return c.paths[pkgPrefix], true, nil
		} else if strings.HasPrefix(pkg, pkgPrefix+"/") { // 如果prefix相等，那么就把prefix替换为value并返回。
			return filepath.Join(c.paths[pkgPrefix], strings.TrimPrefix(pkg, pkgPrefix+"/")), true, nil
		}
	}
	// 没解析到，直接返回
	return "", false, nil
}
```





### FindDeps



最外层的FindDeps只是一层包装

```go
func (p *GoPackage) FindDeps(config *Config, path string) error {
    // 执行完成后打印
	defer un(config.trace("findDeps"))
	// 这里直接new了一个depSet
	depSet := newDepSet()
    // 调用实际的方法，并将寻找到的内容设置到depSet中
	err := p.findDeps(config, path, depSet)
    // 处理异常case
	if err != nil {
		return err
	}
    // 将解析结果设置到GoPackage.allDeps
	p.allDeps = depSet.packageList
    // 返回
	return nil
}
```



通过go标准库的ast解析能力对需要构建的文件的import/comment进行扫描。并将所有的依赖包记录到linkedDepSet中，方便后续编译 & 连接。

``` go

// 解析package中的所有依赖，并将以来叠加到linkedDepSet中 
func (p *GoPackage) findDeps(config *Config, path string, allPackages *linkedDepSet) error {
	// If this ever becomes too slow, we can look at reading the files once instead of twice
	// But that just complicates things today, and we're already really fast.
    // 对特定路径的go package进行ast遍历解析，只解析comment & import 
	foundPkgs, err := parser.ParseDir(token.NewFileSet(), path, func(fi os.FileInfo) bool {
		name := fi.Name()
        // 不递归解析文件夹，不解析_test.go皆为的文件，不解析以.或者__开头的文件
		if fi.IsDir() || strings.HasSuffix(name, "_test.go") || name[0] == '.' || name[0] == '_' {
			return false
		}
        // 如果当前运行的系统为mac系统，_darwin.go结尾的文件不进行解析
		if runtime.GOOS != "darwin" && strings.HasSuffix(name, "_darwin.go") {
			return false
		}
        // 如果当前运行的系统为linux, _linux.go结尾的文件不进行解析
		if runtime.GOOS != "linux" && strings.HasSuffix(name, "_linux.go") {
			return false
		}
		return true
	}, parser.ImportsOnly|parser.ParseComments)
    
    // 异常处理 
	if err != nil {
		return fmt.Errorf("Error parsing directory %q: %v", path, err)
	}

	var foundPkg *ast.Package
	// foundPkgs is a map[string]*ast.Package, but we only want one package
    // 确保此处只有一个package, 超过一个直接报错，可能是约定熟成吧
	if len(foundPkgs) != 1 {
		return fmt.Errorf("Expected one package in %q, got %d", path, len(foundPkgs))
	}
	// Extract the first (and only) entry from the map.
    // 由于这里只有一个package所以for循环其实也只会执行一次...
	for _, pkg := range foundPkgs {
		foundPkg = pkg
	}

    // 中间变量
	var deps []string
	localDeps := make(map[string]bool)

    // 遍历package中的所有文件
	for filename, astFile := range foundPkg.Files {
		ignore := false
        // 解析ast文件中的comment记录
		for _, commentGroup := range astFile.Comments {
			for _, comment := range commentGroup.List {
                // 这里及解析了下注解，如果注解满足某种规则，直接忽略解析当前文件的依赖。
                // 这里的规则我也懒得看了......
				if matches, ok := parseBuildComment(comment.Text); ok && !matches {
					ignore = true
				}
			}
		}
        // 如果发现注释满足某种规则直接跳过import解析逻辑
		if ignore {
			continue
		}

        // 解析import前，首先把文件名称append到files中
		p.files = append(p.files, filename)

        // 开始解析import声明
		for _, importSpec := range astFile.Imports {
            // 首先移除反转义字符
			name, err := strconv.Unquote(importSpec.Path.Value)
			if err != nil {
				return fmt.Errorf("%s: invalid quoted string: <%s> %v", filename, importSpec.Path.Value, err)
			}

            // 如果当前这个file已经被解析过了那就直接continue了不要重复解析
			if pkg, ok := allPackages.tryGetByName(name); ok {
                // 如果package不为空，顺便叠加到localDeps中，可能方便后续的处理？
				if pkg != nil {
					if _, ok := localDeps[name]; !ok {
						deps = append(deps, name)
						localDeps[name] = true
					}
				}
				continue
			}

			var pkgPath string
            // 获取package对应文件的绝对路径
			if path, ok, err := config.Path(name); err != nil {
				return err
			} else if !ok {
                // 如果解析不到路径，默认他是stdlib, 将其加入到忽略路径中去。
                // (如果实际编译的路径中没有这个包，会直接报错，所以这里应该只是为了防止他重复解析)
				// Probably in the stdlib, but if not, then the compiler will fail with a reasonable error message
				// Mark it as such so that we don't try to decode its path again.
				allPackages.ignore(name)
				continue
			} else {
                // 寻找到路径了，先保存一下
				pkgPath = path
			}

            // 创建一个package对象
			pkg := &GoPackage{
				Name: name,
			}
            // 叠加依赖
			deps = append(deps, name)
			allPackages.add(name, pkg)
			localDeps[name] = true
			// 递归解析被依赖的package
			if err := pkg.findDeps(config, pkgPath, allPackages); err != nil {
				return err
			}
		}
	}

    // 对解析到的文件进行排序
	sort.Strings(p.files)

    // 日志输出
	if config.Verbose {
		fmt.Fprintf(os.Stderr, "Package %q depends on %v\n", p.Name, deps)
	}

    // 对路径进行排序
	sort.Strings(deps)
	for _, dep := range deps {
        // 将解析到的依赖叠加到p.directDeps也就是直接依赖
		p.directDeps = append(p.directDeps, allPackages.getByName(dep))
	}

	return nil
}
```





### Compile



分为3个步骤：

1.编译前准备

​	a. 编译的command line准备

​	b. 当前构建的hash值计算

​	c. 通过拓扑排序优先让依赖的模块先触发构建

2.缓存逻辑

​	a.如果之前已经编译过尝试复用

​	b.将#1中计算的hash值与上次编译的hash进行对比如果一样则复用构建产物，不一样会重新触发一次构建

3.构建

​	a.如果#2处的构建失效了，那么就直接调用go编译器触发一次构建，中途会保存hash值方便缓存复用。

```go

func (p *GoPackage) Compile(config *Config, outDir string) error {
    // 加锁
	p.mutex.Lock()
	defer p.mutex.Unlock()
	// 如果已经编译过了，则输出编译结果
    if p.compiled {
		return p.failed
	}
    // 开始编译前先设置编译的flag
	p.compiled = true

	// Build all dependencies in parallel, then fail if any of them failed.
	var wg sync.WaitGroup
    // 先编译所有的直接依赖
	for _, dep := range p.directDeps {
		wg.Add(1)
		go func(dep *GoPackage) {
			defer wg.Done()
			dep.Compile(config, outDir)
		}(dep)
	}
    // 等待编译完成
	wg.Wait()
    // 如果直接依赖中有有任意一个失败则认为本次编译的结果失败
	for _, dep := range p.directDeps {
		if dep.failed != nil {
			p.failed = dep.failed
			return p.failed
		}
	}

    // 打印一下编译的输出。
	endTrace := config.trace("check compile %s", p.Name)
	// 填充部分变量
    // pkg路径
	p.pkgDir = filepath.Join(outDir, strings.Replace(p.Name, "/", "-", -1))
    // pkg输出路径
	p.output = filepath.Join(p.pkgDir, p.Name) + ".a"
    // hash文件路径
	shaFile := p.output + ".hash"
	// 创建一个hash计算器
	hash := sha1.New()
    // 将runtime.GOOS, runtime.GOARCH, goVersion写入hash计算器
	fmt.Fprintln(hash, runtime.GOOS, runtime.GOARCH, goVersion)
	// 拼接编译的shell参数
    // 同时会把编译参数拼接到hash计算器中
	cmd := exec.Command(filepath.Join(goToolDir, "compile"),
		"-N", "-l", // Disable optimization and inlining so that debugging works better
		"-o", p.output,
		"-p", p.Name,
		"-complete", "-pack", "-nolocalimports")
	if !isGo18 && !config.Race {
		cmd.Args = append(cmd.Args, "-c", fmt.Sprintf("%d", runtime.NumCPU()))
	}
	if config.Race {
		cmd.Args = append(cmd.Args, "-race")
		fmt.Fprintln(hash, "-race")
	}
	if config.TrimPath != "" {
		cmd.Args = append(cmd.Args, "-trimpath", config.TrimPath)
		fmt.Fprintln(hash, config.TrimPath)
	}
	for _, dep := range p.directDeps {
		cmd.Args = append(cmd.Args, "-I", dep.pkgDir)
		hash.Write(dep.hashResult)
	}
	for _, filename := range p.files {
		cmd.Args = append(cmd.Args, filename)
		fmt.Fprintln(hash, filename)

		// Hash the contents of the input files
		f, err := os.Open(filename)
		if err != nil {
			f.Close()
			err = fmt.Errorf("%s: %v", filename, err)
			p.failed = err
			return err
		}
		_, err = io.Copy(hash, f)
		if err != nil {
			f.Close()
			err = fmt.Errorf("%s: %v", filename, err)
			p.failed = err
			return err
		}
		f.Close()
	}
    // 计算hash值
    // Note：到此为止应该是计算出了hash值 & cmd参数
	p.hashResult = hash.Sum(nil)
	// 判断是否需要触发一次实际的编译流程
	var rebuild bool
    // 判断1： 如果elf文件都没有生成，那没选的，直接编译吧
	if _, err := os.Stat(p.output); err != nil {
		rebuild = true
	}
    // 判断2： 如果之前已经编译过了，对比下hash值怎么样，是否一致
    // 一致则复用，不一致则重新构建。
	if !rebuild {
		if oldSha, err := ioutil.ReadFile(shaFile); err == nil {
			rebuild = !bytes.Equal(oldSha, p.hashResult)
		} else {
			rebuild = true
		}
	}
    
	endTrace()
    // 如果选择复用缓存，后面的编译流程就不需要执行了
	if !rebuild {
		return nil
	}
	defer un(config.trace("compile %s", p.Name))
	// 编译前先把之前的构建产物删除
	err := os.RemoveAll(p.pkgDir)
	if err != nil {
		err = fmt.Errorf("%s: %v", p.Name, err)
		p.failed = err
		return err
	}
	// 重新创建output路径
	err = os.MkdirAll(filepath.Dir(p.output), 0777)
	if err != nil {
		err = fmt.Errorf("%s: %v", p.Name, err)
		p.failed = err
		return err
	}

    // 编译
	cmd.Stdin = nil
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if config.Verbose {
		fmt.Fprintln(os.Stderr, cmd.Args)
	}
	err = cmd.Run()
	if err != nil {
		commandText := strings.Join(cmd.Args, " ")
		err = fmt.Errorf("%q: %v", commandText, err)
		p.failed = err
		return err
	}
	// 将最终的hash值写入到文件中
	err = ioutil.WriteFile(shaFile, p.hashResult, 0666)
	if err != nil {
		err = fmt.Errorf("%s: %v", p.Name, err)
		p.failed = err
		return err
	}

	p.rebuilt = true

	return nil
}
```

​                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           

### Link



逻辑如下

1.缓存

同上compile流程，计算hash值是否一样，一样就复用，不一样就重新link

2.链接

​	a.删除上一次的构建产物

​	b.拼接链接command line

​	c.执行link并将hash值写入到文件中

```go
func (p *GoPackage) Link(config *Config, out string) error {
	if p.Name != "main" {
		return fmt.Errorf("Can only link main package")
	}
	endTrace := config.trace("check link %s", p.Name)

	shaFile := filepath.Join(filepath.Dir(out), "."+filepath.Base(out)+"_hash")

    // check是否需要触发构建
	if !p.rebuilt {
		if _, err := os.Stat(out); err != nil {
			p.rebuilt = true
		} else if oldSha, err := ioutil.ReadFile(shaFile); err != nil {
			p.rebuilt = true
		} else {
			p.rebuilt = !bytes.Equal(oldSha, p.hashResult)
		}
	}
	endTrace()
    // 不需要触发构建，直接return
	if !p.rebuilt {
		return nil
	}
	defer un(config.trace("link %s", p.Name))
	// 需要触发构建
    
    // 先移除上一此构建构建产物
    // 这里的sha文件和compile的sha不是同一个！！！
	err := os.Remove(shaFile)
	if err != nil && !os.IsNotExist(err) {
		return err
	}
	err = os.Remove(out)
	if err != nil && !os.IsNotExist(err) {
		return err
	}
	// 拼接commandline
	cmd := exec.Command(filepath.Join(goToolDir, "link"), "-o", out)
	if config.Race {
		cmd.Args = append(cmd.Args, "-race")
	}
	for _, dep := range p.allDeps {
		cmd.Args = append(cmd.Args, "-L", dep.pkgDir)
	}
	cmd.Args = append(cmd.Args, p.output)
	cmd.Stdin = nil
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if config.Verbose {
		fmt.Fprintln(os.Stderr, cmd.Args)
	}
    // 执行
	err = cmd.Run()
	if err != nil {
		return fmt.Errorf("command %s failed with error %v", cmd.Args, err)
	}
	// 将compile计算的hash值写入到新的hash文件中。
	return ioutil.WriteFile(shaFile, p.hashResult, 0666)
}
```

