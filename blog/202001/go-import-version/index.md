# 如何在Go项目中输出版本信息？


我们经常在使用CLI工具的时候，都会有这样的参数输出：
```
➜  ~ docker version
Client: Docker Engine - Community
 Version:           18.09.2
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        6247962
 Built:             Sun Feb 10 04:12:39 2019
 OS/Arch:           darwin/amd64
 Experimental:      false
➜  ~
```
可以打印出构建时对应的版本信息，比如 Version，Go Version，Git Commit等，这个是如何实现的呢？

# 实现
主要是通过`ldflags`参数来实现在构建的时候对变量进行赋值。

比如下面一段代码：
```
package main

import (
	"flag"
	"fmt"
	"os"
)

//需要赋值的变量
var version = ""

//通过flag包设置-version参数
var printVersion bool

func init() {
	flag.BoolVar(&printVersion, "version", false, "print program build version")
	flag.Parse()
}

func main() {
	if printVersion {
		println(version)
		os.Exit(0)
	}
	fmt.Printf("example for print version")
}
```

构建命令：
```
go build -ldflags "-X main.version=v0.1" -o example
```

程序输出：
```
➜ ./example
version=v0.1
```

# 参数说明
1、-ldflags build命令中用于调用接链接器的参数
```
-ldflags '[pattern=]arg list'
	arguments to pass on each go tool link invocation.
```

2、-X 链接器参数，主要用于设置变量
```
-X importpath.name=value
    Set the value of the string variable in importpath named name to value.
    Note that before Go 1.5 this option took two separate arguments.
    Now it takes one argument split on the first = sign.
```


# 一个完整的例子
这里将version包单独做了一个包存放，只需要引入即可：
```
package main

import (
	"flag"

	"github.com/go-demo/version"
)

//通过flag包设置-version参数
var printVersion bool

func init() {
	flag.BoolVar(&printVersion, "version", false, "print program build version")
	flag.Parse()
}

func main() {
	if printVersion {
		version.PrintVersion()
	}
}
```

构建的shell如下(也可以放在Makefile中)：

```
#!/bin/sh
version="v0.1"
path="github.com/go-demo/version"
flags="-X $path.Version=$version -X '$path.GoVersion=$(go version)' -X '$path.BuildTime=`date +"%Y-%m-%d %H:%m:%S"`' -X $path.GitCommit=`git rev-parse HEAD`"
go build -ldflags "$flags" -o example example-version.go
```
TIPS: 如果值内容中含有空格，可以用单引号


最终版本输出:
```
➜ sh build.sh
➜ ./example -version
Version: v0.1
Go Version: go version go1.13.1 darwin/amd64
Git Commit: a775ecd27c5e78437b605c438905e9cc888fbc1c
Build Time: 2020-01-09 19:01:51
```

完整代码：[https://github.com/go-demo/version](https://github.com/go-demo/version)


