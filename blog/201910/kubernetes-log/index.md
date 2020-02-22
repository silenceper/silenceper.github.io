# Kubernetes容器日志收集方案


收集POD中container日志，日志还分为两种一种是容器标准输出日志和容器内日志。

# 方案

从日志的采集方式上，在我看来方案大致主要分为两种：

**（1）POD 里面安装logging agent**

  每个pod里面都要安装一个agent，无论是以放在本container还是以sidecar的方式部署，很明显会占用很多资源，基本不推荐

**（2）在节点上安装logging agent（推荐）**

  其实容器stdout,stderr的日志最终也是落在宿主机上，而容器内的路径可以通过配置volumeMount 在宿主机上配置映射即可，所以这种方式还是最可行的

> 当然应用还可以自己通过代码直接上报给日志服务，但是这种方式不够通用，还增加了业务代码的复杂性

# log-pilot 代码分析
这里以log-pilot为例，来分析下

代码地址： [https://github.com/AliyunContainerService/log-pilot](https://github.com/AliyunContainerService/log-pilot)

**基本流程:**

监听docker-service的事件（start，restart，destory），从container信息中提取pod_name,namespace信息（可通过docker inspect [container ID]查看到），并写入filebeat中的prospector.d中的配置文件。

**事件监听：**
[pilot/pilot.go#L134](https://github.com/AliyunContainerService/log-pilot/blob/0948b2c1feca97f849bd3a4eeda7bd40a907efca/pilot/pilot.go#L134)

```
	msgs, errs := p.client.Events(ctx, options)chenqz1987, 1 year ago: • fix issue #40 and update log-pilot capability…

	go func() {
		defer func() {
			log.Warn("finish to watch event")
			p.stopChan <- true
		}()

		log.Info("begin to watch event")

		for {
			select {
			case msg := <-msgs:
				if err := p.processEvent(msg); err != nil {
					log.Errorf("fail to process event: %v,  %v", msg, err)
				}
			case err := <-errs:
				log.Warnf("error: %v", err)
				if err == io.EOF || err == io.ErrUnexpectedEOF {
					return
				}
				msgs, errs = p.client.Events(ctx, options)
			}
		}
	}()
```

**对接不同收集器**

log-pilot支持`filebeat`和`fluentd`两种日志收集器，目的主要是刷新对应的配置文件。主要是实现了`Piloter`接口：

[pilot/piloter.go#L17](https://github.com/AliyunContainerService/log-pilot/blob/0948b2c1feca97f849bd3a4eeda7bd40a907efca/pilot/piloter.go#L17)

```
// Piloter interface for piloter
type Piloter interface {
	Name() string

	Start() error
	Reload() error
	Stop() error

	GetBaseConf() string
	GetConfHome() string
	GetConfPath(container string) string

	OnDestroyEvent(container string) error
}
```

至于在启动log-pilot选择哪种采集器进行采集，是通过在容器的[entrypoint](https://github.com/AliyunContainerService/log-pilot/blob/master/assets/entrypoint)文件中通过制定环境变量`PILOT_TYPE`的方式进行选择性渲染的。

**完善**

当前最新版本[0.9.7]

1、目前log-pilot还是只能通过监听docker-servier的event进行，对于containerd，cri-o还没办法适配。

> containerd目前没提供相应的event接口，可以选择从k8s api 通过List-Watch中获取

2、filebeat升级

> filebeat 目前用的版本是6.1.1 ，filebeat本身就是一个比较吃资源的进程，filebeat有些issue是[7820](https://github.com/elastic/beats/pull/7820)bug，可以将filebeat升级到一个新的版本。

3、资源限制
> 在k8s中通过制定limit，限制log-pilot的资源占用（主要还是日志采集器）



