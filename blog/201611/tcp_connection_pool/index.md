# 聊聊连接池


以下主要使用Golang作为编程语言

# 为什么需要连接池
我觉得使用连接池最大的一个好处就是**减少连接的创建和关闭，增加系统负载能力**，
之前就有遇到一个问题：[TCP TIME_WAIT连接数过多导致服务不可用](http://silenceper.com/blog/201601/tcp-time_wait%E8%BF%9E%E6%8E%A5%E6%95%B0%E8%BF%87%E5%A4%9A%E5%AF%BC%E8%87%B4%E6%9C%8D%E5%8A%A1%E4%B8%8D%E5%8F%AF%E7%94%A8/)，因为未开启数据库连接池，再加上mysql并发较大，导致需要频繁的创建链接，最终产生了上万的TIME_WAIT的tcp链接，影响了系统性能。

链接池中的的功能主要是管理一堆的链接，包括创建和关闭，所以自己在[fatih/pool](fatih/pool)基础上，改造了一下：[https://github.com/silenceper/pool](https://github.com/silenceper/pool) ，使得更加通用一些，增加的一些功能点如下：

- 连接对象不单单是`net.Conn`,变为了`interface{}`（池中存储自己想要的格式）
- 增加了链接的最大空闲时间（保证了当连接空闲太久，链接失效的问题）

主要是用到了`channel`来管理连接，并且能够很好的利用管道的顺序性，当需要使用的时候`Get`一个连接，使用完毕之后`Put`放回`channel`中。


# 连接失效问题
使用连接池之后就不再是短连接，而是长连接了，就引发了一些问题：

## 1、长时间空闲，连接断开？
因为网络环境是复杂的，中间可能因为防火墙等原因，导致长时间空闲的连接会断开，所以可以通过两个方法来解决：

- 客户端增加心跳，定时的给服务端发送请求
- 给连接池中的连接增加最大空闲时间，超时的连接不再使用

在[https://github.com/silenceper/pool]([https://github.com/silenceper/pool])就增加了一个这样最大空闲时间的参数，在连接创建或者连接被重新返回连接池中时重置，给每个连接都增加了一个连接的创建时间，在取出的时候对时间进行比较：[https://github.com/silenceper/pool/blob/master/channel.go#L85](https://github.com/silenceper/pool/blob/master/channel.go#L85)

## 2、当服务端重启之后，连接失效？

远程服务端很有可能重启，那么之前创建的链接就失效了。客户端在使用的时候就需要判断这些失效的连接并丢弃，在`database/sql`中就判断了这些失效的连接，使用这种错误表示`var ErrBadConn = errors.New("driver: bad connection")`

另外值得一提的就是在`database/sql`对这种`ErrBadConn`错误进行了重试，默认重试次数是两次，所以能够保证即便是链接失效或者断开了，本次的请求能够正常响应（继续往下看就是分析了）。


**连接失效的特征**

- 对连接进行read读操作时，返回`EOF`错误
- 对连接进行write操作时，返回`write tcp 127.0.0.1:52089->127.0.0.1:8002: write: broken pipe`错误

# database/sql 中的连接池
在`database/sql`中使用连接连接池很简单，主要涉及下面这些配置：

```go
	db.SetMaxIdleConns(10) //连接池中最大空闲连接数
	db.SetMaxOpenConns(20) //打开的最大连接数
	db.SetConnMaxLifetime(300*time.Second)//连接的最大空闲时间(可选)
```
> 注：如果`MaxIdleConns`大于0并且`MaxOpenConns`小于`MaxIdleConns `,那么会将`MaxIdleConns`置为`MaxOpenConns`

来看下db这个结构，以及字段相关说明：

```go
type DB struct {
	//具体的数据库实现的interface{},
	//例如https://github.com/go-sql-driver/mysql 就注册并并实现了driver.Open方法，主要是在里面实现了一些鉴权的操作
	driver driver.Driver
	//dsn连接
	dsn    string
	//在prepared statement中用到
	numClosed uint64

	mu           sync.Mutex // protects following fields
	//可使用的空闲的链接
	freeConn     []*driverConn
	//用来传递连接请求的管道
	connRequests []chan connRequest
	//当前打开的连接数
	numOpen      int
	//当需要创建新的链接的时候，往这个管道中发送一个struct数据，
	//因为在Open数据库的就启用了一个goroutine执行connectionOpener方法读取管道中的数据
	openerCh    chan struct{}
	//数据库是否已经被关闭
	closed      bool
	//用来保证锁被正确的关闭
	dep         map[finalCloser]depSet
	//stacktrace of last conn's put; debug only
	lastPut     map[*driverConn]string
	//最大空闲连接
	maxIdle     int
	//最大打开的连接
	maxOpen     int
	//连接的最大空闲时间
	maxLifetime time.Duration
	//定时清理空闲连接的管道
	cleanerCh   chan struct{}
}

```

看一个查询数据库的例子：

```
	rows, err := db.Query("select * from table1")
```

在调用`db.Query`方法如下：

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
	var rows *Rows
	var err error
	//这里就做了对失效的链接的重试操作
	for i := 0; i < maxBadConnRetries; i++ {
		rows, err = db.query(query, args, cachedOrNewConn)
		if err != driver.ErrBadConn {
			break
		}
	}
	if err == driver.ErrBadConn {
		return db.query(query, args, alwaysNewConn)
	}
	return rows, err
}
```
在什么情况下会返回，可以从这里看到：
[readPack](https://github.com/go-sql-driver/mysql/blob/master/packets.go#L35)，[writePack](https://github.com/go-sql-driver/mysql/blob/master/packets.go#L132)


继续跟进去就到了
``` go
func (db *DB) conn(strategy connReuseStrategy) (*driverConn, error) {
```
方法主要是创建tcp连接，并判断了连接的生存时间lifetime，以及连接数的一些限制，如果超过的设定的最大打开链接数限制等待`connRequest`管道中有连接产生(在`putConn`释放链接的时候就会往这个管道中写入数据)

**何时释放链接?**

当我们调用`rows.Close()`的时候，就会把当前正在使用的链接重新放回`freeConn`或者写入到`db.connRequests`管道中

```go
	//putConnDBLocked 方法

	//如果有db.connRequests有在等待连接的话，就把当前连接给它用
	if c := len(db.connRequests); c > 0 {
		req := db.connRequests[0]
		// This copy is O(n) but in practice faster than a linked list.
		// TODO: consider compacting it down less often and
		// moving the base instead?
		copy(db.connRequests, db.connRequests[1:])
		db.connRequests = db.connRequests[:c-1]
		if err == nil {
			dc.inUse = true
		}
		req <- connRequest{
			conn: dc,
			err:  err,
		}
		return true
	} else if err == nil && !db.closed && db.maxIdleConnsLocked() > len(db.freeConn) {
	//没人需要我这个链接，我就把他重新返回`freeConn`连接池中
		db.freeConn = append(db.freeConn, dc)
		db.startCleanerLocked()
		return true
	}

```


# 使用连接池管理Thrift链接
这里是使用连接池[https://github.com/silenceper/pool](https://github.com/silenceper/pool)，如何构建一个thrift链接

客户端创建Thrift的代码：

```go
type Client struct {
	*user.UserClient
}


//创建Thrift客户端链接的方法
factory := func() (interface{}, error) {
	protocolFactory := thrift.NewTBinaryProtocolFactoryDefault()
	transportFactory := thrift.NewTTransportFactory()

	var transport thrift.TTransport
	var err error
	transport, err = thrift.NewTSocket(rpcConfig.Listen)
	if err != nil {
		panic(err)
	}
	transport = transportFactory.GetTransport(transport)
	//defer transport.Close()
	if err := transport.Open(); err != nil {
		panic(err)
	}
	rpcClient := user.NewUserClientFactory(transport, protocolFactory)
	//在连接池中直接放置Client对象
	return &Client{UserClient: rpcClient}, nil
}
//关闭连接的方法
close := func(v interface{}) error {
	v.(*Client).Transport.Close()
	return nil
}

//创建了一个 初始化连接是
poolConfig := &pool.PoolConfig{
	InitialCap: 10,
	MaxCap:     20,
	Factory:     factory,
	Close:       close,
	IdleTimeout: 300 * time.Second,
}
p, err := pool.NewChannelPool(poolConfig)
if err != nil {
	panic(err)
}

//取得链接
conn, err := p.Get()
if err != nil {
	return nil, err
}
v, ok := conn.(*Client)

...使用连接调用远程方法

//将连接重新放回连接池中
p.Put(conn)

```

***

写完，听见外面的🐓开始打鸣了。

