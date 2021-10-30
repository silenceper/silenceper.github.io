# emptyDir 通过xfs_quota 强制限制大小

emptyDir支持三种类型的，通过设置 `medium` 字段  ：

- 文件：默认情况
- Memory：占用内存资源
- HugePages

## sizeLimit默认行为

同时支持通过sizeLimit设置限制的大小，但是这个大小默认情况下(`LocalStorageCapacityIsolation` 特性默认开启)并不是强制限制的，而是由eviction manager 扫描到超过设定的大小之后，再将pod进行驱逐，所以存在一种情况就是文件其实已经超过了限定的大小(可能已经影响到了系统上其他服务)，而驱逐是定时触发的，有一定的时间间隔。

我们希望能够达到强制的效果的话，就需要做一些hack。

## 通过xfs quota 限制

- kubelet root-dir 使用xfs文件系统，并附上  `project quota` 属性，例如：  `/dev/vdb /data xfs noatime,prjquota 1 2`
- node上 `xfs_quota` 工具使用较新版本，需要支持 `-f` 参数

其实在 k8s 1.15增加了一个特性 `LocalStorageCapacityIsolationFSQuotaMonitoring` (PR：[https://github.com/kubernetes/kubernetes/pull/66928](https://github.com/kubernetes/kubernetes/pull/66928)) ，这个特性就是通过XFS quotas来给kubelet目录设置配额，但是这里仅仅只是监控消耗，并没有强制限制，可以看下这段代码：

[https://github.com/kubernetes/kubernetes/blob/master/pkg/volume/util/fsquota/quota_linux.go#L342-L343](https://github.com/kubernetes/kubernetes/blob/master/pkg/volume/util/fsquota/quota_linux.go#L342-L343)

```go
func AssignQuota(m mount.Interface, path string, poduid types.UID, bytes *resource.Quantity) error {
.....
    //这里强制设置成了-1 表示无限制
		if ibytes > 0 {
			ibytes = -1
		}
		if err = setQuotaOnDir(path, id, ibytes); err == nil {
			quotaPodMap[id] = poduid
			quotaSizeMap[id] = ibytes
			podQuotaMap[poduid] = id
			dirQuotaMap[path] = id
			dirPodMap[path] = poduid
			podDirCountMap[poduid]++
			klog.V(4).Infof("Assigning quota ID %d (%d) to %s", id, ibytes, path)
			return nil
		}
		removeProjectID(path, id)
	}
	return fmt.Errorf("assign quota FAILED %v", err)
}
```

### hack

将其中 `ibytes > 0` 判断改为 `ibytes ≤ 0` 即，setQuota的时候将sizeLimit设置进去，就可以达到强制限制的效果。

测试一下：

pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: pod-test
  namespace: default
spec:
  containers:
  - args:
    - "100000"
    command:
    - sleep
    image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    volumeMounts:
    - name: data-test
      mountPath: /data/test/
  volumes:
  - name: data-test
    emptyDir:
       sizeLimit: "50Mi"
```

进入pod中写入文件测试：

```yaml
root@pod-test:/data/test# dd if=/dev/zero of=./bigfile bs=1M count=2000
dd: error writing './bigfile': No space left on device
51+0 records in
50+0 records out
52428800 bytes (52 MB, 50 MiB) copied, 0.0573529 s, 914 MB/s
```

达到我们的预期。

### 参考

[https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir](https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir)

