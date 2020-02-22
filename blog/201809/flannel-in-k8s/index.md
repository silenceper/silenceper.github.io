# k8s网络组件：flannel




Flannel是一个专门为k8s定制的网络解决方案，主要解决POD跨主机通信问题，这里主要讲述Flannel是如何实现的。

# 安装
集成在k8s上有两种方式，一种是利用etcd存储整个集群的网络配置，另外一种是利用kubernetes的api 获取网络配置信息，分别如下：

## ETCD 方式
这种方式在前面的[k8s集群安装-安装Flannel网络插件](http://silenceper.com/blog/201809/how-to-build-a-k8s-cluster/#安装flannel网络插件)已经介绍过，就是利用etcd存储网络配置信息，并且利用dockerd启动参数`--bip`来指定每个节点的网段信息，我们指定vxlan作为数据传输的backend。


## CNI插件方式

这种方式是利用cni插件，并且将网络配置信息存储在kubernetes api中：

### 配置网段

在`kube-controller-manager`启动脚本中加入`--allocate-node-cidrs=true`，`--cluster-cidr=10.244.0.0/16` 参数。

`10.244.0.0/16`为我们指定的集群的网段，这样每一个docker节点都会分别使用自网段`10.244.0.0/24`，`10.244.1.0/24`作为每个pod的网段，可以通过`kubectl get pod 192.168.99.101 -o yaml`命令查看`spec.podCIDR`字段。

如果不配置该步骤可能会flannel出现`error registering network: failed to acquire lease: node "192.168.99.101" pod`的错误

### 指定kublet网络插件
在kubelet中指定三个参数
- `--network-plugin=cni` : 网络插件使用cni
- `--cni-conf-dir=/etc/cni/net.d` cni配置文件 ，默认
- `--cni-bin-dir=/opt/cni/bin` cni可执行文件，默认

这样在kublet启动的时候，就会去`/etc/cni/net.d`目录查找配置文件，并解析，使用相应的插件配置网络

### 下载cni依赖插件
下载cni官方提供的组件：[cni-plugins-amd64-v0.7.1.tgz](https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz) ，并将可执行文件放在`/opt/cni/bin`目录。

这里不单单只会用到flannel文件，也会用到brage用来创建网桥以及host-local用来分配ip

### 安装flanneld组件
flannel组件直接以`daemonset`的方式安装在k8s集群中：

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```
其中`kube-flannel-cfg`configmap中的`Network`字段需要与`--cluster-cidr`配置的网段一致。


这一步主要是安装了flanneld以及将flannel配置文件写入`/etc/cni/net.d`目录


查看`kube-flannel.yml`中flanneld的启动参数为：

- `--ip-masq `: 需要为其配置SNAT
- `--kube-subnet-mgr` : 代表其使用kube类型的subnet-manager。该类型有别于使用etcd的local-subnet-mgr类型，使用kube类型后，flannel上各Node的IP子网分配均基于K8S Node的spec.podCIDR属性

>  我的是多网卡的机器，发现网络不通，所以我特别指定的flanneld用来对外通信的网卡
>  增加--iface 参数，指定可以进行外部通信的网卡
## 区别
- flanneld 容器化了
- flanneld 网段信息来源于kubernetes api，而不依赖etcd

# 原理


以上两种方式的对比可以发现，通过cni方式安装的flannel会多一个`cni0`的网桥，其实这个与`docker0`网桥的作用是一样的，创建了一对veth pair 分别联通容器内部与主机网络，并把其中一端接入`cni0`网桥。

整个通信过程如图：

![](https://ws1.sinaimg.cn/large/65209136gy1fvmsrceiq7j20oh0gv0ut.jpg)

> 图中的ip信息与本机不一致，主要说明通信过程

容器内部->docker0->flanneld->网络传输->对端flanneld->对端docker0->对端容器内

当数据包到吧flanneld后支持多种backend进行转发

## Flannel backend

- Host-gw: 同一网段直接通信，速度快
- udp： 用户空间的udp封装
- vxlan: 利用内核vxlan协议进行传输
- aws vpc: 利用aws平台内部提供的功能进行传输

更多插件参考文档：[https://github.com/coreos/flannel/blob/master/Documentation/backends.md](https://github.com/coreos/flannel/blob/master/Documentation/backends.md)


**如何选择？**

`vxlan`在功能上，和性能上基本满足我们的要求，可以直接跨网段间通信，同时性能上又比udp的方式高效




