# k8s源码阅读(一)：源码结构


# 环境
- k8s代码版本：[release-1.9](https://github.com/silenceper/kubernetes)
- 工具：vscode, dlv

下载代码，并放入gopath中，方便编译：

```
$ mkdir -p $GOPATH/src/k8s.io  && cd $GOPATH/src/k8s.io
$ git clone https://github.com/kubernetes/kubernetes # 有墙
```

# 代码结构

```
├── Godeps        godep依赖文件
├── api           swagger api文档
├── build         构建用到的命令
├── cluster       自动构建和快速构建k8s集群的脚本(进入维护阶段)
├── cmd           核心：命令入口
├── docs          文档
├── examples      一些部署样例yaml文件
├── hack          hack代码
├── logo          logo图片
├── pkg           核心：具体命令的逻辑代码
├── plugin        主要是kube-scheduler
├── staging       暂存区，用于分离仓库
├── test          test
├── third_party   一些第三方工具
├── translations  多语言文件
└── vendor        依赖代码

```


组件入口基本都在`/cmd/`目录下，并且可直接通过`go build`单独编译（通过`make xxxx`也行，`make help`查看具体指令） 

# DLV调试阅读

有两种方式，一种是直接通过`dlv`命令进行调试，比如调试kubelet组件：

```
$ cd cmd/kubelet
$ dlv debug . -- --logtostderr=true --v=5 --address=192.168.99.102 --hostname-override=192.168.99.102 --allow-privileged=true --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false
```
> 其中`--`后为程序运行参数

进入之后为通过`break`，`n`,`s`,`p`等指令操作，这种方式不太方便，尤其对于k8s这种大工程，推荐配合工具进行，比如vscode。

通过我配置`launch.json`文件的`args`参数，让`kubelet`运行起来：

```
"args": [
    "--address=192.168.186.219",
    "--hostname-override=192.168.186.219",
    "--cluster-domain=cluster.local",
    "--root-dir=/Users/silenceper/workspace/golang/src/k8s.io/kubelet",
    "--bootstrap-kubeconfig=/Users/silenceper/workspace/golang/src/k8s.io/kubelet/kubernetes/bootstrap.kubeconfig",
    "--kubeconfig=/Users/silenceper/workspace/golang/src/k8s.io/kubelet/kubernetes/kubelet.kubeconfig",
    "--cert-dir=/Users/silenceper/workspace/golang/src/k8s.io/kubelet/kubernetes/ssl",
],
```

通过调试，可以做到边看代码，变追踪kubelet的执行过程
> 因为os的原因，执行流程可能与目标平台的执行结果可能不一致，不过可以手动改代码达到预期结果，或者直接在Linux下调试

![11](https://ws1.sinaimg.cn/large/65209136gy1fv6z2jx1ajj21g20sc7b3.jpg)
