# Cluster Autoscaler:集群自动扩缩容


Cluster AutoScaler 是一个自动扩展和收缩 Kubernetes 集群 Node 的扩展。当集群容量不足时，它会自动去 Cloud Provider （支持 GCE、GKE 和 AWS）创建新的 Node，而在 Node 长时间资源利用率很低时自动将其删除以节省开支。

在Kubernetes中关于弹性伸缩主要有三种格式：

- **HPA**：Horizontal Pod Autoscaling可以根据CPU利用率自动伸缩一个Replication Controller、Deployment 或者Replica Set中的Pod数量
- **VPA**：Vertical Pod Autoscaler（VPA）使用户无需为其pods中的容器设置最新的资源request。配置后，它将根据使用情况自动设置request，从而允许在节点上进行适当的调度，以便为每个pod提供适当的资源量。
- **CA**：自动伸缩NODE节点

## 部署
Cluster Autoscaler代码地址：[https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)

CA 提供了[Cloud Provider](https://github.com/kubernetes/autoscaler/blob/7cfa98b4ca308fbede1a2f1054ef3f00daf560dd/cluster-autoscaler/cloudprovider/cloud_provider.go#L31)接口供各个云厂商接入，主要是针对厂商自己的API，应对节点添加以及删除的请求。

目前云厂商的ECS产品都会有扩缩容的功能（例如阿里云ESS，就是CA中伸缩组的概念），CA就可以结合ESS完成集群的扩缩容功能。

各大厂商节点扩缩容安装：

- GCE: [https://kubernetes.io/docs/concepts/cluster-administration/cluster-management/](https://kubernetes.io/docs/concepts/cluster-administration/cluster-management/)
- GKE: [https://cloud.google.com/container-engine/docs/cluster-autoscaler](https://cloud.google.com/container-engine/docs/cluster-autoscaler)
- AWS: [https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)
- Azure: [https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/azure](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/azure)
- alicloud：[https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/alicloud/README.md](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/alicloud/README.md)
- baiducloud：[https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/baiducloud/README.md](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/baiducloud/README.md)

以阿里云为例，具体安装可以参考这篇文章：[阿里云上弹性伸缩kubernetes集群 - autoscaler
](https://yq.aliyun.com/articles/392170)

其中关键yaml文件如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: admin
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/google-containers/cluster-autoscaler:v1.1.0
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=alicloud
            - --skip-nodes-with-local-storage=false
            - --nodes=1:100:${AUTO_SCALER_GROUP}
          env:
          - name: ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: cloud-config
                key: access-key-id
          - name: ACCESS_KEY_SECRET
            valueFrom:
              secretKeyRef:
                name: cloud-config
                key: access-key-secret
          imagePullPolicy: "Always"
```

其中 `--nodes=1:100:${AUTO_SCALER_GROUP}`参数，表示扩容最大100个，缩容最小1个节点，后面`${AUTO_SCALER_GROUP}`表示伸缩组ID

## 工作原理

### 扩容
CA会定期(默认10s,通过参数`--scan-interval`设置)检测当前集群状态下是否存在pending的pod，然后经过计算，判断需要扩容几个节点，最终从node group中进行节点的扩容：

Node Group 就对应伸缩组的概念，可以支持配置支持多个伸缩组，通过策略来进行选择，目前支持的策略为：

- random：随机选择
- most-pods：选择能够创建pod最多的Node Group
- least-waste：以最小浪费原则选择，即选择有最少可用资源的 Node Group
- price：根据主机的价格选择，选择最便宜的
- priority：根据优先级进行选择，通过配置`cluster-autoscaler-priority-expander` configMap，具体参考：[https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/expander/priority](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/expander/priority)

### 缩容
集群缩容其实是一个可选的选项，通过参数`--scale-down-enabled`控制是否开启缩容。

CA定期会检测集群状态，判断当前集群状态下，哪些节点资源利用率小于50%(通过参数`--scale-down-utilization-threshold`控制)。

资源利用率计算是通过判断集群cpu，mem 中request值占用率计算的，只要有一个指标超了就可能会出发缩容，之所以说是可能会触发扩容，是因为要保证被驱逐节点上的POD能够正确调度到其他节点上。

**哪些NODE不会缩容：**

- 当您设置了严格的 PodDisruptionBudget 的 Pod 不满足 PDB 时，不会缩容。
- Kube-system 下的 Pod。通过参数`--skip-nodes-with-system-pods`控制
- 节点上有非 deployment，replica set，job，stateful set 等控制器创建的 Pod。
- Pod 有本地存储。通过参数`--skip-nodes-with-local-storage`控制
- Pod 不能被调度到其他节点上。例如资源不满足等


## 实现Cluster Provider
如果你使用的以上主流的几个云厂商的主机资源/k8s集群，可以直接使用官方的CA版本，只需要通过参数`--cloud-provider`控制来自哪个云厂商就可以。

如果想要对接自己的IaaS层，只需要实现其中的接口就可以了，接口定义如下：

```
// CloudProvider contains configuration info and functions for interacting with
// cloud provider (GCE, AWS, etc).
type CloudProvider interface {Marcin Wielgus, 3 years ago: • Cluster-autoscaler: cloud provider interface
	// Name returns name of the cloud provider.
	Name() string

	// NodeGroups returns all node groups configured for this cloud provider.
	//返回所有伸缩组
	NodeGroups() []NodeGroup

	// NodeGroupForNode returns the node group for the given node, nil if the node
	// should not be processed by cluster autoscaler, or non-nil error if such
	// occurred. Must be implemented.
	NodeGroupForNode(*apiv1.Node) (NodeGroup, error)

	// Pricing returns pricing model for this cloud provider or error if not available.
	// Implementation optional.
	Pricing() (PricingModel, errors.AutoscalerError)

	// GetAvailableMachineTypes get all machine types that can be requested from the cloud provider.
	// Implementation optional.
	GetAvailableMachineTypes() ([]string, error)

	// NewNodeGroup builds a theoretical node group based on the node definition provided. The node group is not automatically
	// created on the cloud provider side. The node group is not returned by NodeGroups() until it is created.
	// Implementation optional.
	NewNodeGroup(machineType string, labels map[string]string, systemLabels map[string]string,
		taints []apiv1.Taint, extraResources map[string]resource.Quantity) (NodeGroup, error)

	// GetResourceLimiter returns struct containing limits (max, min) for resources (cores, memory etc.).
	GetResourceLimiter() (*ResourceLimiter, error)

	// GPULabel returns the label added to nodes with GPU resource.
	GPULabel() string

	// GetAvailableGPUTypes return all available GPU types cloud provider supports.
	GetAvailableGPUTypes() map[string]struct{}

	// Cleanup cleans up open resources before the cloud provider is destroyed, i.e. go routines etc.
	Cleanup() error

	// Refresh is called before every main loop and can be used to dynamically update cloud provider state.
	// In particular the list of node groups returned by NodeGroups can change as a result of CloudProvider.Refresh().
	Refresh() error
}
```
其中NodeGroup接口定义：

```
// NodeGroup contains configuration info and functions to control a set
// of nodes that have the same capacity and set of labels.
type NodeGroup interface {
	// MaxSize returns maximum size of the node group.
	MaxSize() int

	// MinSize returns minimum size of the node group.
	MinSize() int

	// TargetSize returns the current target size of the node group. It is possible that the
	// number of nodes in Kubernetes is different at the moment but should be equal
	// to Size() once everything stabilizes (new nodes finish startup and registration or
	// removed nodes are deleted completely). Implementation required.
	TargetSize() (int, error)

	// IncreaseSize increases the size of the node group. To delete a node you need
	// to explicitly name it and use DeleteNode. This function should wait until
	// node group size is updated. Implementation required.
	//扩容伸缩组
	IncreaseSize(delta int) error

	// DeleteNodes deletes nodes from this node group. Error is returned either on
	// failure or if the given node doesn't belong to this node group. This function
	// should wait until node group size is updated. Implementation required.
	//从伸缩组中删除节点
	DeleteNodes([]*apiv1.Node) error

	// DecreaseTargetSize decreases the target size of the node group. This function
	// doesn't permit to delete any existing node and can be used only to reduce the
	// request for new nodes that have not been yet fulfilled. Delta should be negative.
	// It is assumed that cloud provider will not delete the existing nodes when there
	// is an option to just decrease the target. Implementation required.
	DecreaseTargetSize(delta int) error

	// Id returns an unique identifier of the node group.
	Id() string

	// Debug returns a string containing all information regarding this node group.
	Debug() string

	// Nodes returns a list of all nodes that belong to this node group.
	// It is required that Instance objects returned by this method have Id field set.
	// Other fields are optional.
	Nodes() ([]Instance, error)

	// TemplateNodeInfo returns a schedulernodeinfo.NodeInfo structure of an empty
	// (as if just started) node. This will be used in scale-up simulations to
	// predict what would a new node look like if a node group was expanded. The returned
	// NodeInfo is expected to have a fully populated Node object, with all of the labels,
	// capacity and allocatable information as well as all pods that are started on
	// the node by default, using manifest (most likely only kube-proxy). Implementation optional.
	TemplateNodeInfo() (*schedulernodeinfo.NodeInfo, error)

	// Exist checks if the node group really exists on the cloud provider side. Allows to tell the
	// theoretical node group from the real one. Implementation required.
	Exist() bool

	// Create creates the node group on the cloud provider side. Implementation optional.
	Create() (NodeGroup, error)

	// Delete deletes the node group on the cloud provider side.
	// This will be executed only for autoprovisioned node groups, once their size drops to 0.
	// Implementation optional.
	Delete() error

	// Autoprovisioned returns true if the node group is autoprovisioned. An autoprovisioned group
	// was created by CA and can be deleted when scaled to 0.
	Autoprovisioned() bool
}

```

其中`IncreaseSize`，`IncreaseSize`才是关键的方法，分别对用扩容伸缩组和增加节点。

在调用`IncreaseSize`接口后，需要完成节点自动添加进集群。参考阿里云是通过配置shell脚本到伸缩组中的ECS启动脚本中，当ECS启动，自动执行该脚本。当然我们也可以自定义通过其他方式加入集群。





