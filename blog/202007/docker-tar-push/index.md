# 将镜像tar包通过API直接push到registry仓库


为了实现docker tar包能够直接通过页面上传，调研了一下registry的api，以及如何解析tar包（其实就是docker daemon程序实现的部分）。

要想实现，首先要了解docker tar包中结构组成：
## tar包结构
拿了一个包含一个镜像的tar包进行解压：

```
.
├── 1ecf8bc84a7c3d60c0a6bbdd294f12a6b0e17a8269616fc9bdbedd926f74f50c
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── 6f4ec1f3d7ea33646d491a705f94442f5a706e9ac9acbca22fa9b117094eb720.json
├── aaac5bde2c2bcb6cc28b1e6d3f29fe13efce6d6b669300cc2c6bfab96b942af4
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── b63363f0d2ac8b3dca6f903bb5a7301bf497b1e5be8dc4f57a14e4dc649ef9bb
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── c453224a84b8318b0a09a83052314dd876899d3a1a1cf2379e74bba410415059
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── dd8ef1d42fbcccc87927eee94e57519c401b84437b98fcf35505fb6b7267a375
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── manifest.json
└── repositories
```
文件说明：

**manifest.json文件**：

```
[
    {
        "Config":"6f4ec1f3d7ea33646d491a705f94442f5a706e9ac9acbca22fa9b117094eb720.json",
        "RepoTags":[
            "alpine:filebeat-6.8.7-arm64"
        ],
        "Layers":[
            "aaac5bde2c2bcb6cc28b1e6d3f29fe13efce6d6b669300cc2c6bfab96b942af4/layer.tar",
            "dd8ef1d42fbcccc87927eee94e57519c401b84437b98fcf35505fb6b7267a375/layer.tar",
            "c453224a84b8318b0a09a83052314dd876899d3a1a1cf2379e74bba410415059/layer.tar",
            "b63363f0d2ac8b3dca6f903bb5a7301bf497b1e5be8dc4f57a14e4dc649ef9bb/layer.tar",
            "1ecf8bc84a7c3d60c0a6bbdd294f12a6b0e17a8269616fc9bdbedd926f74f50c/layer.tar"
        ]
    }
]
```
manifest.json 包含了对这个tar包的描述信息，比如image config文件地址，tags说明，镜像layer信息，在解析的时候也是根据这个文件去获取关联的文件。

**image config文件**

比如：`6f4ec1f3d7ea33646d491a705f94442f5a706e9ac9acbca22fa9b117094eb720.json`文件：
>内容太多就不贴了
这里包含了镜像运行的信息，比如env，执行参数，以及镜像历史等。

**layer层文件：layer.tar**

镜像的每一层的文件信息都打包在了一个单独的`layer.tar`包中，也是在上传的时候需要用到的。

## registry api

**流程：**

- 1、获取鉴权信息
- 2、检查layer.tar是否已经存在
- 3、上传layer.tar
- 4、上传image config
- 5、上传manifest（非包中的`manifest.json`而是[Manifest struct](https://github.com/docker/distribution/blob/master/manifest/schema2/manifest.go#L63)）

> 这里只列出了在上传的时候需要用的api

1、获取鉴权信息

这里跟registry仓库选择的鉴权方式有关，可选basic auth或者token鉴权。
这里我是以最基本的basic auth为例子，如果是token鉴权的方式参考这里：
https://docs.docker.com/registry/spec/auth/jwt/

2、检查上传

```
HEAD /v2/image/blobs/<digest>
```
若返回200 OK 则表示存在，不用上传


3、开始上传服务
我这里是以分块上传的方式进行

```
POST /v2/image/blobs/uploads/
```
如果post请求返回202 accepted，一个url会在location字段返回.

```
     202 Accepted
     Location: /v2/\<image>/blobs/uploads/\<uuid>
     Range: bytes=0-<offset>
     Content-Length: 0
     Docker-Upload-UUID: <uuid> # 可以用来查看上传状态和实现断点续传
```
4、分块上传
根据上一步获取的url，以`PATCH`的方式提交分块数据。
如果是最后一块数据上传，则以PUT的方式提交，如下：

```
 > PUT /v2/<name>/blob/uploads/<uuid>?digest=<digest>
 > Content-Length: <size of chunk>
 > Content-Range: <start of range>-<end of range>
 > Content-Type: application/octet-stream
   <Last Layer Chunk Binary Data>
```
上传layer，image config，manifests信息都是一样的。

> 我暂时只用到了以上api，更多的api参考：https://docs.docker.com/registry/spec/api/

## 实现
上面说API没啥感觉，还是看代码比较明白，所以实现了这么一个工具：[https://github.com/silenceper/docker-tar-push
](https://github.com/silenceper/docker-tar-push
)

**Usage:**

```
push your docker tar archive image without docker.

Usage:
  docker-tar-push [flags]

Flags:
  -h, --help              help for docker-tar-push
      --log-level int     log-level, 0:Fatal,1:Error,2:Warn,3:Info,4:Debug (default 3)
      --password string   registry auth password
      --registry string   registry url
      --skip-ssl-verify   skip ssl verify
      --username string   registry auth username
```

**Example:**

```
docker-tar-push alpine:latest --registry=http://localhost:5000
```

**参考**

- https://www.jianshu.com/p/6a7b80122602
- https://github.com/Razikus/dockerregistrypusher
- https://docs.docker.com/registry/spec/auth/jwt/
- https://github.com/docker/distribution/
