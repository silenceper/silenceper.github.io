# 使用keda完成基于事件的弹性伸缩

> 文章中使用的是keda 1.5版本，2.0还未release
> 1.5版本支持deployment，job两种资源。而在2.0增加了StatefulSet以及自定义资源

[keda](https://keda.sh/) 是一个支持多种事件源来对应用进行弹性伸缩的控制器。
我觉得keda可以认为是基于HPA的external metrics的一种扩展，因为它利用了hpa中external metrics的能力，允许直接配置多个事件源：
![支持的事件源](/img/20200914/keda-scalers.png)

# 安装keda

从 https://github.com/kedacore/keda/releases 下载1.5版本的zip包，包含了yaml和crd:

```
├── 00-namespace.yaml
├── 01-service_account.yaml
├── 10-cluster_role.yaml
├── 11-role_binding.yaml
├── 12-operator.yaml
├── 20-metrics-cluster_role.yaml
├── 21-metrics-role_binding.yaml
├── 22-metrics-deployment.yaml
├── 23-metrics-service.yaml
├── 24-metrics-api_service.yaml
└── crds
    ├── keda.k8s.io_scaledobjects_crd.yaml
    └── keda.k8s.io_triggerauthentications_crd.yaml
```
安装：
```
kubectl apply -f ./crds/
kubectl apply -f .
```
以prometheus 和cron两个事件源看下如何使用

## 配置 ScaledObject
以Deployment为例，看下ScaledObject支持哪些变量
```
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: {scaled-object-name}
spec:
  scaleTargetRef:
    deploymentName: {deployment-name} # must be in the same namespace as the ScaledObject
    containerName: {container-name}  #Optional. Default: deployment.spec.template.spec.containers[0]
  pollingInterval: 30  # Optional. Default: 30 seconds
  cooldownPeriod:  300 # Optional. Default: 300 seconds
  minReplicaCount: 0   # Optional. Default: 0
  maxReplicaCount: 100 # Optional. Default: 100
  triggers:
  # {list of triggers to activate the deployment}
```

## 基于prometheus指标进行伸缩
我这里有个[http demo](https://github.com/silenceper/http-demo)的应用，上报了`gin_requests_total`指标到prometheus，通过ab请求http接口模拟压力上升的情况：

这里我将`minReplicaCount`设置为2，因为在没有流量的时候副本数将会被keda设置为0。
```
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    deploymentName: http-demo
  minReplicaCount: 2
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://192.168.99.100:31046/
      metricName: gin_requests_total
      threshold: '2'
      query: sum(rate(gin_requests_total{app="http-demo",code="200"}[2m]))
```

当创建该`ScaledObject `，我们看到同时创建了一个hpa资源：

```
➜  example kubectl get hpa
NAME                 REFERENCE              TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-http-demo   Deployment/http-demo   0/2 (avg)   2         100       2          70m
```
通过压测观察hpa资源变化：

```
> ab  -c 10 -n 1000 http://127.0.0.1:8080/
```

```
Every 1.0s: kubectl get hpa                                                                                      anymore.local: Mon Sep 14 14:23:59 2020

NAME                 REFERENCE              TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-http-demo   Deployment/http-demo   8/2 (avg)   2         100       4          73m
```
可以看得到随着流量上升，由hpa控制的副本数在上升，这就达到了我们根据流量扩容的目的，当流量降下来之后，副本数也减少。

# 组件组成

keda由两个组件组成：

- keda operator： 负责创建维护hpa对象资源，同时激活和停止hpa伸缩。在无事件的时候将副本数降低为0(如果未设置minReplicaCount的话)
- metrics server: 实现了hpa中external metrics，根据事件源配置返回计算结果。
![keda架构图](/img/20200914/keda-arch.png)

可以看得到HPA控制了副本1->N和N->1的变化。
keda控制了副本0->1和1->0的变化（起到了激活和停止的作用，对于一些消费型的任务副本比较有用，比如在凌晨启动任务进行消费）




# 扩展事件源(external-scalers)

对于在keda不支持的一些事件源，我们还可以使用keda提供的扩展机制来扩充自己的事件源。
[https://keda.sh/docs/1.5/concepts/external-scalers/](https://keda.sh/docs/1.5/concepts/external-scalers/)
主要是通过grpc实现一下接口实现：

```
type Scaler interface {
	GetMetrics(ctx context.Context, metricName string, metricSelector labels.Selector) ([]external_metrics.ExternalMetricValue, error)
	GetMetricSpecForScaling() []v2beta2.MetricSpec
	IsActive(ctx context.Context) (bool, error)
	Close() error
}
```

- IsActive 在ScaledObject / ScaledJob CRD中定义的pollingInterval上，每隔pollingInterval时间，调用IsActive，如果返回true，则将比例缩放为1(默认1)
- Close调用Close可以使缩放器清除连接或其他资源。
- GetMetricSpec 返回缩放器的HPA定义的目标值。
- GetMetrics 返回从GetMetricSpec引用的指标的值。

>在2.0中还多支持一种`PushScaler`形式的扩展，允许用户主动push来是否开启/停止基于事件伸缩

# 总结
之前用过k8s-prometheus-adapter项目来进行应用的自定义指标进行扩展，对比keda感觉keda操作更简单，配置更加动态化，因为抽象了hpa，用户直接操作`ScaledObject`即可，不需要关注hpa如何进行配置。而且支持将副本数设置为0，同时又拥有类似cronhpa(定时伸缩)的功能，扩展能力比较强。

> 阅读原文体验更加，文中链接支持跳转

