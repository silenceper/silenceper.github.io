# docker pull 翻墙下载镜像


我们一般通过设置`http_proxy`环境变量，使得http请求，可以走我们设置的proxy，（一些go get镜像无法下载可以这么用），但是对于`docker  pull`命令是不生效的，因为systemd引导启动的service默认不会读取这些变量，所以我们可以通过在service文件中加入环境变量解决：

# 设置变量 http_proxy

主要是通过在docker启动的时候指定`HTTP_PROXY `，`HTTPS_PROXY`。

```
[Service]    
Environment="HTTP_PROXY=http://proxy.example.com:80/" "HTTPS_PROXY=http://proxy.example.com:80/""NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"
```

其中`NO_PROXY`变量指的是那些http请求不走代理。


下面两种都是一样的效果，推荐方式二，可以避免直接修改`docker.service`文件，防止升级之后文件覆盖。

## 方式一： 直接修改docker.service 文件
docker service文件`/usr/lib/systemd/system/docker.service`，在Service段配置中加入。

## 方式二：通过override.conf文件(推荐)

文件路径：
`/etc/systemd/system/docker.service.d/override.conf`

可以直接通过`systemctl edit docker`打开文件进行编辑，加入上述配置，重启生效。

> 这种方式来之 [@Feng_Yu](https://github.com/abcfy2)的提醒，THK

## 重启docker

```
systemctl daemon-reload
systemctl restart docker
```


**TIPS: `polipo` 可以将`socks5`协议转换成`http`代理。**


# 参考资料
>
>
>[https://docs.docker.com/config/daemon/systemd/#httphttps-proxy](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)
>

