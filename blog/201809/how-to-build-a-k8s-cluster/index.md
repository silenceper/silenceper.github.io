# 在CentOS上搭建Kubernetes集群



以下是我自己在部署k8s集群上做的一些记录，部署了一个master，一个node节点。

# 环境准备

我在VirtualBox中建的两个CentOS容器，并且互通：

| IP |系统| 角色 | 配置 |
| ------ |-----| ------ | ------ |
| 192.168.99.101 |CentOS 7.5.1804| master | 1核2G |
| 192.168.99.102 |CentOS 7.5.1804| node | 1核2G |

安装版本：

| 组件 | 版本 |
| ------ | ------ |
| Kubernetes  | 1.10.7|
| Etcd |  3.1.10     |
| Docker | 17.12.1-ce|


# 证书准备
使用CFSSL工具进行证书生成

CA证书和秘钥文件主要用来做传输加密用的，将会生成如下文件：

- ca-key.pem
- ca.pem
- kubernetes-key.pem
- kubernetes.pem
- kube-proxy.pem
- kube-proxy-key.pem
- admin.pem
- admin-key.pem

使用证书的组件如下：

- etcd：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
- kube-apiserver：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
- kubelet：使用 ca.pem；
- kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem；
- kubectl：使用 ca.pem、admin-key.pem、admin.pem；
- kube-controller-manager：使用 ca-key.pem、ca.pem

在master节点上进行操作：

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

export PATH=/usr/local/bin:$PATH
```
## 创建ca证书

创建 `ca-config.json`文件：

```
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```

创建 `ca-csr.json`文件：

```
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
```

生成ca证书和私钥

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

## 生成kubernetes证书

创建`kubernetes-csr.json`文件：

```
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "192.168.99.101",
      "192.168.99.102",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

其中ip段改为自己的ip，包括etcd集群IP，k8s master集群ip，k8s服务的服务ip(一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.254.0.1)

生成证书和私钥：

```
[root@localhost ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```
## 创建admin证书
创建 `admin-csr.json`文件：

```
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```
生成证书

```
[root@localhost ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

## 创建kube-proxy证书

创建kube-proxy-csr.json文件：

```
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

```
生成命令：

```
[root@localhost ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

## 分发证书
将生成的证书和秘钥文件（后缀为`.pem`）拷贝到所有机器的`/etc/kubernetes/ssl` 目录下
> 命令略

# 安装kubectl文件
```
wget -c https://dl.k8s.io/v1.10.7/kubernetes-client-darwin-386.tar.gz
tar -xzvf kubernetes-client-linux-amd64.tar.gz
cp kubernetes/client/bin/kube* /usr/bin/
chmod a+x /usr/bin/kube*
```

## 创建 kubectl kubeconfig 文件

```
export KUBE_APISERVER="https://192.168.99.101:6443"
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
# 设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
# 设置默认上下文
kubectl config use-context kubernetes
```

> admin 证书中设定了 `group=system:masters`即（O=system:masters）
> 这个group与ckuster-admin ClusterRole绑定，所以具有整个集群的权限
> 如果需要单独指定权限，则可以自己设定user绑定ClusterRole
 
# 创建 kubeconfig 文件

## 创建 TLS Bootstrapping Token

```
$ export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')

$ cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

$ cp token.csv /etc/kubernetes/
```


## 创建 kubelet bootstrapping kubeconfig 文件

```
cd /etc/kubernetes
export KUBE_APISERVER="https://192.168.99.101:6443"

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
## 创建 kube-proxy kubeconfig 文件

```
export KUBE_APISERVER="https://192.168.99.101:6443"
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
## 分发kubeconfig文件
将两个 kubeconfig 文件分发到所有 Node 机器的 /etc/kubernetes/ 目录

```
cp bootstrap.kubeconfig kube-proxy.kubeconfig /etc/kubernetes/

```

# Etcd安装
我这里为了简单，ETCD节点只用了一个，并且安装在了master节点上，多个也是一样的，只是在`--initial-cluster`选择中将其他节点ip填入进去。


下载ETCD:

```
$ wget -c https://github.com/coreos/etcd/releases/download/v3.1.10/etcd-v3.1.10-linux-amd64.tar.gz
$ tar -zxvf etcd-v3.1.10-linux-amd64.tar.gz
$ mv etcd-v3.1.10-linux-amd64/etcd* /usr/local/bin

```
创建 etcd 的 systemd unit 文件

```
vim  /usr/lib/systemd/system/etcd.service
```
内容如下：

```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --peer-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls ${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
  --listen-peer-urls ${ETCD_LISTEN_PEER_URLS} \
  --listen-client-urls ${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
  --advertise-client-urls ${ETCD_ADVERTISE_CLIENT_URLS} \
  --initial-cluster-token ${ETCD_INITIAL_CLUSTER_TOKEN} \
  --initial-cluster infra1=https://192.168.99.101:2380 \
  --initial-cluster-state new \
  --data-dir=${ETCD_DATA_DIR}
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

创建`/etc/etcd/etcd.conf`文件:

```
# [member]
ETCD_NAME=infra1
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.99.101:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.99.101:2379"

#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.99.101:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.99.101:2379"
```

启动服务：

```
mv etcd.service /usr/lib/systemd/system/
mkdir /var/lib/etcd
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

验证服务：

```
etcdctl   --ca-file=/etc/kubernetes/ssl/ca.pem   --cert-file=/etc/kubernetes/ssl/kubernetes.pem   --key-file=/etc/kubernetes/ssl/kubernetes-key.pem   cluster-health
2018-09-07 15:41:30.782777 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2018-09-07 15:41:30.783295 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
member 171ef35542ebf92b is healthy: got healthy result from https://192.168.99.101:2379
cluster is healthy
```

`cluster is healthy `表示成功。

# Master节点部署

master节点上主要是三个组件：

- kube-apiserver
- kube-scheduler
- kube-controller-manager

> 同事只能又一个`kube-scheduler`,`kube-controller-manager`进程处于工作状态，如果运行多个，则需要通过选举产生一个leader。

下载server包：

```
wget -c https://dl.k8s.io/v1.10.7/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes
tar -xzvf  kubernetes-src.tar.gz
```

将二进制文件拷贝到指定路径

```
cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler} /usr/local/bin/

```

同时将 `server/bin/`目录下的`kube-proxy`,`kubelet` 三个文件复制到node节点的 `/usr/local/bin/` 目录

## 配置和启动 kube-apiserver

创建 `/etc/kubernetes/config`文件:

```
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver

KUBE_MASTER="--master=http://192.168.99.101:8080"

```

创建service文件`/usr/lib/systemd/system/kube-apiserver.service`：

```
[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/local/bin/kube-apiserver \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_ETCD_SERVERS \
        $KUBE_API_ADDRESS \
        $KUBE_API_PORT \
        $KUBELET_PORT \
        $KUBE_ALLOW_PRIV \
        $KUBE_SERVICE_ADDRESSES \
        $KUBE_ADMISSION_CONTROL \
        $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

apiserver的配置文件 `/etc/kubernetes/apiserver`:

```
###
## kubernetes system config
##
## The following values are used to configure the kube-apiserver
##
#
## The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=192.168.99.101 --bind-address=192.168.99.101 --insecure-bind-address=192.168.99.101"
#
## The port on the local server to listen on.
#KUBE_API_PORT="--port=8080"
#
## Port minions listen on
#KUBELET_PORT="--kubelet-port=10250"
#
## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://192.168.99.101:2379"
#
## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
#
## default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota"
#
## Add your own!
KUBE_API_ARGS="--authorization-mode=Node,RBAC --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --enable-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h"

```

启动kube-apiserver:

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

## 配置和启动 kube-controller-manager

创建 `/usr/lib/systemd/system/kube-controller-manager.service`文件：

```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/local/bin/kube-controller-manager \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

配置文件：`/etc/kubernetes/controller-manager`

```
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true"
```
启动 kube-controller-manager:

```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

## 配置和启动 kube-scheduler

创建`/usr/lib/systemd/system/kube-scheduler.service`文件：

```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/local/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

`/etc/kubernetes/scheduler`文件：

```
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"

```

启动 kube-scheduler:

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler

```

### 验证master节点

```
[root@localhost ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
```

# 安装Flannel网络插件
Flannel网络插件只需要安装在node节点上就好了，他主要作用是将不同node上的pod网络联通，所以我们在102这台机器上操作：

使用yum直接安装:

```
yum install -y flannel

```

创建 `/usr/lib/systemd/system/flanneld.service`文件：

```
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=/usr/bin/flanneld-start \
  -etcd-endpoints=${FLANNEL_ETCD_ENDPOINTS} \
  -etcd-prefix=${FLANNEL_ETCD_PREFIX} \
  $FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service

```

`/etc/sysconfig/flanneld`文件：

```
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://192.168.99.101:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"

```
在etcd中创建网络配置，在master节点上操作就好了，省得指定endpoints:

```
$ etcdctl   --ca-file=/etc/kubernetes/ssl/ca.pem   --cert-file=/etc/kubernetes/ssl/kubernetes.pem   --key-file=/etc/kubernetes/ssl/kubernetes-key.pem  mkdir /kube-centos/network

$ etcdctl   --ca-file=/etc/kubernetes/ssl/ca.pem   --cert-file=/etc/kubernetes/ssl/kubernetes.pem   --key-file=/etc/kubernetes/ssl/kubernetes-key.pem  mk /kube-centos/network/config '{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'

```


启动flannel

```
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld

```

# Docker安装
下载并安装：

```
wget -c https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-17.12.1.ce-1.el7.centos.x86_64.rpm
yum install docker-ce-17.12.1.ce-1.el7.centos.x86_64.rpm 
```

在`/usr/lib/systemd/system/docker.service`文件中增加一个环境变量：

```
EnvironmentFile=-/run/flannel/docker
```
并且启动命令为，主要是增加了一个`$DOCKER_NETWORK_OPTIONS`参数是flannel传入的用来修改`docker0`网桥IP
```
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS --log-driver=json-file --log-opt max-size=50m --log-opt max-file=5
```

启动：

```
systemctl daemon-reload
systemctl enable docker
systemctl start docker
systemctl status docker
```

# Node节点部署

`kube-proxy`，`kubelet` 文件在`kubernetes-server-linux-amd64.tar.gz`，复制到`/usr/local/bin`目录


## 安装配置kubelet
> 前提：关闭swap，否则无法启动
> 使用:`swapoff -a`或者注释fstab中swap配置

TIPS: 现在master节点上，赋予kubelet-bootstrap用户，system:node-bootstrapper cluster 角色，否则kubelet没有权限创建认证请求。

```
cd /etc/kubernetes
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

文件`/usr/lib/systemd/system/kubelet.service`：

```
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/bin/kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_ADDRESS \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

同时配置`/etc/kubernetes/kubelet`文件：

```
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=192.168.99.102"
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=192.168.99.102"

KUBELET_ARGS="--bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false"
```

启动：
```
mkdir /varlib/kubelet
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

### 通过kublet的TLS证书请求

```
[root@localhost kubernetes]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-vtB-rALjRudPIp4IvIHwOTOggoJC-213GpvDoYVqQlA   18s       kubelet-bootstrap   Pending
```

通过 CSR 请求:

```
[root@localhost kubernetes]# kubectl certificate approve node-csr-vtB-rALjRudPIp4IvIHwOTOggoJC-213GpvDoYVqQlA
certificatesigningrequest.certificates.k8s.io "node-csr-vtB-rALjRudPIp4IvIHwOTOggoJC-213GpvDoYVqQlA" approved
```

 
## 配置 kube-proxy

安装conntrack：

```
yum install -y conntrack-tools
```


文件`/usr/lib/systemd/system/kube-proxy.service`：

```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/kube-proxy \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

`/etc/kubernetes/proxy`文件：

```
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=192.168.99.102 --hostname-override=192.168.99.102 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"

```

启动kube-proxy:

```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```

支持集群的基本功能已经有了，可以建一个部署一个nginx应用试一下看一下。

# 安装kubedns插件

kubedns 为我们提供了服务发现的功能。

yaml文件通过这里下载：

[https://github.com/silenceper/k8s-install/tree/master/kube-dns
](https://github.com/silenceper/k8s-install/tree/master/kube-dns)

在master上执行：

`kubectl create -f .`


至此，k8s就搭建完成了！

>
>参考资料：
>
>https://jimmysong.io/kubernetes-handbook/practice/install-kubernetes-on-centos.html
>













