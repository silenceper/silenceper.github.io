# xfsquota：一个便捷的管理xfs磁盘配额的命令行工具

**源码地址：**[https://github.com/silenceper/xfsquota](https://github.com/silenceper/xfsquota)

## 动机

**在Linux有一个`xfs_quota`（在`xfsprogs`工具包下）命令行工具，为什么还用golang实现了？**

主要是要因为最近要实现磁盘quota的控制，同时觉得看了docker内的源码，都是利用cgo的方式来实现的，如果直接用`xfs_quota`的方式查看配额，无法直观的看到某一个目录下的配额，只能列出所有，并且没有具体目录，类似如下结果，在设置了docker 容器的quota之后查看每个容器的配额，只有`Project ID`无法判别到具体某个目录：

```shell
 # xfs_quota -x -c "report" /data
 Project quota on /data (/dev/vdb)
                               Blocks
Project ID       Used       Soft       Hard    Warn/Grace
---------- --------------------------------------------------
#0            7394256          0          0     00 [--------]
#2                  8   52428800   52428800     00 [--------]
#3                  8   52428800   52428800     00 [--------]
#4                  8   52428800   52428800     00 [--------]
#5               2360   52428800   52428800     00 [--------]
#6                  8   52428800   52428800     00 [--------]
#7                  8   52428800   52428800     00 [--------]
#8                  8   52428800   52428800     00 [--------]
#9              10568   52428800   52428800     00 [--------]
#10                 0   52428800   52428800     00 [--------]
#11                 0   52428800   52428800     00 [--------]
#12                 0   52428800   52428800     00 [--------]
#13                 0   52428800   52428800     00 [--------]
#14                 0   52428800   52428800     00 [--------]
#15                 0   52428800   52428800     00 [--------]
#16                 0   52428800   52428800     00 [--------]
#17                 0   52428800   52428800     00 [--------]
#18                 0   52428800   52428800     00 [--------]
#19                 0   52428800   52428800     00 [--------]
```

所有用cgo的方式，可以实现对某个目录的quota查看。

## 主要功能

- support set quota

- support get quota

- clean quota 

**Usage**

```shell
xfsquota is a tool for managing XFS quotas

Usage:
  xfsquota [command]

Available Commands:
  clean       clean quota information
  completion  Generate the autocompletion script for the specified shell
  get         Get quota information
  help        Help about any command
  set         Set quota information
  version     get version

Flags:
  -h, --help   help for xfsquota

Use "xfsquota [command] --help" for more information about a command.
```

### Set Quota

set quota size 1MiB ,inodes 20 for path `/data/test/quota`
```shell
> xfsquota set /data/test/quota  -s 1MiB -i 20

set quota success, path: /data/test/quota, size:1MiB, inodes:20
```

### Get Quota

get quota for path `/data/test/quota`
```shell
> xfsquota get /data/test/quota

quota Size(bytes): 1048576
quota Inodes: 20
diskUsage Size(bytes): 0
diskUsage Inodes: 1
```

### Clean Quota

```
xfsquota clean /data/test/quota
```


