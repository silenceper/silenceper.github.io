# 记一次问题排查：为什么在POD无法通过Service访问自己？


## 问题现象

创建一个nginx pod，并配置了service访问，service后端指向pod。

进入pod中使用service ip 或者service 域名，无法访问。

一开始以为是环境配置或者k8s版本（1.9）的问题，在其他1.13的kubernetes环境下也试了，还是同样的问题。

## 环境配置

使用的cni插件是flannel，但不是容器化安装，也不是标准化的通过kubelet指定cni plugin（--cni-bin-dir,--cni-conf-dir参数），而是通过dockerd 提供的`-bip`参数指定容器的子网，而这个值是从`/run/flannel/subnet.env`(flannel会将获取到的子网写入到该文件)


## 排查过程

**1、首先尝试通过pod ip尝试是否可访问，已验证是可通的。**

**2、尝试对docker0网桥进行抓包**

```
tcpdump -i docker0
```

神奇的在这里，再次尝试通过service 访问是居然可以通，发现只要tcpdump断开就不行了。
> 到这里的时候有点觉得诡异了

在pod内通过service访问的时候网络的流向应该是

```
pod内部访问service->docker0网桥->宿主机的iptables规则->docker0网桥->pod内部
```

查阅了相关资料后，看到kubelet有个`--hairpin-mod`参数：

文档说明：
>如果网络没有为“发夹模式”流量生成正确配置，通常当 kube-proxy 以 iptables 模式运行，并且 Pod 与桥接网络连接时，就会发生这种情况。Kubelet 公开了一个 hairpin-mode 标志，如果 pod 试图访问它们自己的 Service VIP，就可以让 Service 的端点重新负载到他们自己身上。hairpin-mode 标志必须设置为 hairpin-veth 或者 promiscuous-bridge。



可是我设置之后还是没有还是不行，翻了一下kubelet里面的代码，发现cni组件并没有取这个值做任何才做（在kubnet中有）

查看这个讨论：
[https://github.com/kubernetes/kubernetes/issues/45790](https://github.com/kubernetes/kubernetes/issues/45790)

**大致结论是，应该由cni插件来根据这个值来做对应的操作。**


还是没解决我的问题？ 

不过我看到hairpin开启的标志位是通过`/sys/devices/virtual/net/docker0/brif/veth-xxx/hairpin_mod` 内容设置为1开启的。

所以我将所有veth该文件内容设置`1`
```
for intf in /sys/devices/virtual/net/docker0/brif/*; do echo 1> $intf/hairpin_mod; done
```

可以访问了。😺


## 解疑：promiscuous-bridge 与 hairpin-veth

**为什么我无法访问**

bridge不允许包从收到包的端口发出，比如这种情况，在pod内通过docker0访问service，后续又通过docker0网桥进来，所以需要开启`hairpin_mod`

**为什么使用tcpdump 可以让访问可通？**

因为tcpdump要抓取所有经过该网卡，所以需要开启混杂模式。可以在/var/log/message看到`device docker0 entered promiscuous mode`的log。

>混杂模式（英語：promiscuous mode）是电脑网络中的术语。 是指一台机器的网卡能够接收所有经过它的数据流，而不论其目的地址是否是它。 一般计算机网卡都工作在非混杂模式下，此时网卡只接受来自网络端口的目的地址指向自己的数据。 当网卡工作在混杂模式下时，网卡将来自接口的所有数据都捕获并交给相应的驱动程序。

手动开关命令：
`ifconfig docker0 promisc on/off`


## 总结
其实我们集群通过这种比较另类的方式来分配POD IP也用了了很久了，之所以没出问题，应该是业务基本没遇到这种pod内通过service访问自己的情况。

所以还是要跟着标准的k8s方式来安装cni，避免入坑，比如flannel就已经提供给了`hairpinMode `参数来进行配置开启。
