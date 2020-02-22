# k8s网络组件：calico


前提已经安装好k8s集群

# 安装

calico 安装其实很简单，已经集成在两个yaml文件中

calico 版本: v3.2.3

## 安装必看
- 如果安装过`flannel`组件，需要先去除docker启动项中 `$DOCKER_NETWORK_OPTIONS`参数
- 删除已有的`/etc/cni/net.d`，`/opt/cni/bin`文件
- `kubelet`和`kube-apiserver`启动项中需要加上`--allow-privileged=true`，保证`calico-node`需要以特权模式运行
- 这里calico文件采用了`CrossSubnet`模式

## 指定k8s使用CNI插件
在kubelet启动参数中加入以下参数:

- `--network-plugin=cni`: 指定使用cni插件
- `--cni-bin-dir=/opt/cni/bin`： 存放网络配置的可执行文件
- `--cni-conf-dir=/etc/cni/net.d`: 存放网络配置文件，如果有多个文件，则只会选择一个(根据文件名排序)

## 部署

安装文件放在: [https://github.com/silenceper/k8s-install/tree/master/calico](https://github.com/silenceper/k8s-install/tree/master/calico)

主要包含两个文件`calico.yaml`，`rbac.yaml`:


需要做如下参数修改：

将其中的`ETCD_LVS_HOST`修改为集群的etcd地址，并且将`TLS_ETCD_KEY`,`TLS_ETCD_CERT `,`TLS_ETCD_CA`替换为证书的内容的base64后的结果：


```
# 其中证书的路径根据自己环境更改为正确的路径
TLS_ETCD_CA=$(cat /etc/kubernetes/ssl/ca.pem | base64 |tr -d "\n")
TLS_ETCD_KEY=$(cat /etc/kubernetes/ssl/etcd-key.pem | base64 |tr -d "\n")
TLS_ETCD_CERT=$(cat /etc/kubernetes/ssl/etcd.pem | base64 |tr -d "\n")

# 替换内容
sed -i "s#TLS_ETCD_CA#$TLS_ETCD_CA#g" calico.yaml
sed -i "s#TLS_ETCD_CERT#$TLS_ETCD_CERT#g" calico.yaml
sed -i "s#TLS_ETCD_KEY#$TLS_ETCD_KEY#g" calico.yaml

# 替换ETCD_LVS_HOST地址，更换为自己的etcd地址
sed -i "s#ETCD_LVS_HOST#192.168.99.101#g" calico.yaml
```

**部署**

```
kubectl apply -f calico.yaml
kubectl apply -f rbac.yaml
```

## calicoctl 安装

calicoctl 工具，主要用来配置calico，例如配置calico 网络模式等

下载地址：[https://github.com/projectcalico/calicoctl/releases](https://github.com/projectcalico/calicoctl/releases)

将下载和二进制文件放在`/usr/local/bin`目录下

**calicoctl 配置文件**

将以下内容写入文件`/etc/calico/calicoctl.cfg`:

```
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  etcdEndpoints: https://192.168.99.101:2379
  etcdKeyFile: /etc/kubernetes/ssl/kubernetes-key.pem
  etcdCertFile: /etc/kubernetes/ssl/kubernetes.pem
  etcdCACertFile: /etc/kubernetes/ssl/ca.pem
```
测试成功：

```
[root@192-168-99-101 calico]# calicoctl get node -o wide
NAME             ASN         IPV4                IPV6
192-168-99-101   (unknown)   192.168.99.101/24
192-168-99-102   (unknown)   192.168.99.102/24
```

###修改ipipMode

ipipMode有三种：

- Never： bgp模式
- Always：IPIP模式
- CrossSubnet：自动选择，同一网段选择bgp模式，跨网段IPIP模式


通过`calicoctl get ippool -o yaml`命令可以查看到当前使用的ipipMode为`CrossSubnet`，通过如下方式可以修改：

`calictl apply -f ippool.yaml`

其中`ippool.yaml`文件内容：

```
apiVersion: projectcalico.org/v3
items:
- apiVersion: projectcalico.org/v3
  kind: IPPool
  metadata:
    name: default-ipv4-ippool
  spec:
    cidr: 172.23.0.0/16
    ipipMode: Always
    natOutgoing: true
kind: IPPoolList
```



 






