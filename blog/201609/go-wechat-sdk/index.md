# 开源项目：wechat sdk



一直很想自己用golang写个微信的sdk，目标是简单好用，所以利用闲暇时间（周末，中秋😁），就做出来。

项目地址:[https://github.com/silenceper/wechat](https://github.com/silenceper/wechat)

目前实现了消息管理，微信网页授权，菜单管理，素材管理几个接口，看下他的基本使用：

以下是一个处理消息接收以及回复的例子：

```go

//配置微信参数
config := &wechat.Config{
    AppID:          "xxxx",
    AppSecret:      "xxxx",
    Token:          "xxxx",
    EncodingAESKey: "xxxx",
    Cache:          memCache
}
wc := wechat.NewWechat(config)

// 传入request和responseWriter
server := wc.GetServer(request, responseWriter)
server.SetMessageHandler(func(msg message.MixMessage) *message.Reply {

    //回复消息：演示回复用户发送的消息
    text := message.NewText(msg.Content)
    return &message.Reply{message.MsgText, text}
})

server.Serve()
server.Send()


```



