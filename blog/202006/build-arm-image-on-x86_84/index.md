# 在x86_64机器上构建arm64镜像



有几种办法可以打包出arm64的镜像

- 直接在arm机器上执行编译和打包
- 通过qemu模拟arm环境
- 利用docker提供的buildx（需要启用试验性特性）

我没有arm的机器~，所以我主要试了一下下面两种方式。


### 借助qemu-user-static镜像打包
文档：https://github.com/multiarch/qemu-user-static

开启arm平台支持：

```sh
$ docker run --rm --privileged multiarch/qemu-user-static:register --reset

$ docker build --rm -t "test/integration/ubuntu" -<<EOF
FROM multiarch/qemu-user-static:x86_64-aarch64 as qemu
FROM arm64v8/ubuntu
COPY --from=qemu /usr/bin/qemu-aarch64-static /usr/bin
EOF

$ docker run --rm -t "test/integration/ubuntu" uname -m
aarch64

```

在运行`qemu-user-static:register`镜像的时候，就通过内核中的binfmt_misc机制注入了哪些可执行文件可以被识别。


注意：需要将`qemu-aarch64-static`文件 copy 到`/usr/bin`目录。

> `binfmt_misc`是Linux内核说提供的一种扩展机制, 使得更多类型的文件得以成为＂可执行＂文件．Linux内核本身支持ELF、a.out、脚本（也就是上面所说的第一行#!指定解释器的方式）这集中＂可执行文件＂．但它还提供了一个称为`binfmt_misc`的内核模块, 通过这个模块可以动态注册一些＂可执行文件格式＂,注册之后我们就可以直接＂执行＂这个程序文件了．



### 通过buildx打包支持多架构的镜像
文档：[https://docs.docker.com/buildx/working-with-buildx/](https://docs.docker.com/buildx/working-with-buildx/)

首先需要开启这个实验性的功能，怎么开启看这里：[https://github.com/docker/cli/blob/master/experimental/README.md](https://github.com/docker/cli/blob/master/experimental/README.md)

- docker cloent: 在`~/.docker/config.json 加入``写入"experimental":"enabled"`
- dcoker daemon: 启动参数上加上--experimental或者`/etc/docker/daemon.json`写入`  "experimental": true`

**进行初始化：**

```bash
docker buildx create --name builderx
docker buildx use builderx
docker buildx inspect --bootstrap #这一步会从外网拉取`moby/buildkit`镜像，貌似还改不了
```

**接下来你就可以通过如下命令去构建多种架构的镜像：**

```bash
docker buildx build --platform linux/amd64,linux/arm64 -f Dockerfile -t silenceper/reverse-proxy .  --push
```
`--push`参数表示构建完成之后，并push到镜像仓库当中。


因为镜像仓库支持一个仓库上传多种架构，并且会根据构建平台pull相匹配架构的镜像，所以我们可以在一个Dockerfile，并且通过一个命令打包镜像并push。


这里用到的Dockerfile在这里：[https://github.com/silenceper/reverse-proxy/blob/master/Dockerfile](https://github.com/silenceper/reverse-proxy/blob/master/Dockerfile)

**通过docker的多阶段构建将编译和打包放在一起了，隔离了环境的差异。**

```go
FROM golang:alpine as builder
ADD . /go/src/github.com/silenceper/reverse-proxy/
RUN cd /go/src/github.com/silenceper/reverse-proxy/ \
  && go get -v \
  && CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine
MAINTAINER silenceper <silenceper@gmail.com>
COPY --from=builder /go/src/github.com/silenceper/reverse-proxy/app /bin/app
ENTRYPOINT ["/bin/app"]
EXPOSE 80
```




## 总结
最方便的肯定是直接通过buildx进行打包和编译，但是我的环境没法访问外网，又没法拉取外网的镜像，所以最终还是选择使用`qemu-user-static`打包出来了。



### 参考

- https://blog.fleeto.us/post/buildx-and-osx/
- https://juejin.im/post/5af86fb15188251b8015c102
- https://www.cnblogs.com/bamanzi/p/run-x86-linux-progs-with-qemu-user-on-arm.html


