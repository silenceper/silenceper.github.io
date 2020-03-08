# singleflight包原理解析

`singleflight` 包主要是用来做并发控制，常见的比如防止 `缓存击穿` ，我们来模拟一下这种场景：
> 缓存击穿：缓存在某个时间点过期的时候，恰好在这个时间点对这个Key有大量的并发请求过来，这些请求发现缓存过期一般都会从后端DB加载数据并回设到缓存，这个时候大并发的请求可能会瞬间把后端DB压垮。 

```go
package main

import (
	"errors"
	"log"
	"sync"

	"golang.org/x/sync/singleflight"
)

var errorNotExist = errors.New("not exist")
func main() {
	var wg sync.WaitGroup
	wg.Add(10)

	//模拟10个并发
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			data, err := getData("key")
			if err != nil {
				log.Print(err)
				return
			}
			log.Println(data)
		}()
	}
	wg.Wait()
}

//获取数据
func getData(key string) (string, error) {
	data, err := getDataFromCache(key)
	if err == errorNotExist {
		//模拟从db中获取数据
		data, err = getDataFromDB(key)
		if err != nil {
			log.Println(err)
			return "", err
		}

		//TOOD: set cache
	} else if err != nil {
		return "", err
	}
	return data, nil
}

//模拟从cache中获取值，cache中无该值
func getDataFromCache(key string) (string, error) {
	return "", errorNotExist
}

//模拟从数据库中获取值
func getDataFromDB(key string) (string, error) {
	log.Printf("get %s from database", key)
	return "data", nil
}
```

其中通过 `getData(key)`方法获取数据，逻辑是：

1. 先尝试从cache中获取
1. 如果cache中不存在就从db中获取

我们模拟了10个并发请求，来同时调用 `getData` 函数，执行结果如下：

```bash
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
2020/03/08 17:13:11 get key from database
2020/03/08 17:13:11 data
```

可以看得到10个请求都是走的db，因为cache中不存在该值，当我们利用上 `singlefligth` 包， `getData` 改动一下：

```go
import "golang.org/x/sync/singleflight"
var g singleflight.Group
//获取数据
func getData(key string) (string, error) {
	data, err := getDataFromCache(key)
	if err == errorNotExist {
		//模拟从db中获取数据
		v, err, _ := g.Do(key, func() (interface{}, error) {
			return getDataFromDB(key)
			//set cache
		})
		if err != nil {
			log.Println(err)
			return "", err
		}

		//TOOD: set cache
		data = v.(string)
	} else if err != nil {
		return "", err
	}
	return data, nil
}
```

执行结果如下，可以看得到只有一个请求进入的db，其他的请求也正常返回了值，从而保护了后端DB。

```bash
2020/03/08 17:18:16 get key from database
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
2020/03/08 17:18:16 data
```


# 原理解析
`singleflight` 在 `golang.org/x/sync/singleflight` 项目下，对外提供了以下几个方法

```go
//Do方法，传入key，以及回调函数，如果key相同，fn方法只会执行一次，同步等待
//返回值v:表示fn执行结果
//返回值err:表示fn的返回的err
//第三个返回值shared：表示是否是真实fn返回的还是从保存的map[key]返回的，也就是共享的
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
//DoChan方法类似Do方法，只是返回的是一个chan
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
//暂时未用到：设计Forget 控制key关联的值是否失效，默认以上两个方法只要fn方法执行完成后，内部维护的fn的值也删除（即并发结束后就失效了）
func (g *Group) Forget(key string) {
```

代码也很简单，我们打开上看下，截取了部分：
[singleflight/singleflight.go](https://github.com/golang/sync/blob/master/singleflight/singleflight.go)

```go
package singleflight // import "golang.org/x/sync/singleflight"

import "sync"

// call is an in-flight or completed singleflight.Do call
type call struct {
	wg sync.WaitGroup

	// These fields are written once before the WaitGroup is done
	// and are only read after the WaitGroup is done.
    //val和err用来记录fn发放执行的返回值
	val interface{}
	err error

	// forgotten indicates whether Forget was called with this call's key
	// while the call was still in flight.
    // 用来标识fn方法执行完成之后结果是否立马删除还是保留在singleflight中
	forgotten bool

	// These fields are read and written with the singleflight
	// mutex held before the WaitGroup is done, and are read but
	// not written after the WaitGroup is done.
    //dups 用来记录fn方法执行的次数
	dups  int
    //用来记录DoChan中调用次数以及需要返回的数据
	chans []chan<- Result
}

// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}


// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
    //check map是否已经存在值
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err, true
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}
//执行fn方法，并且wg.Done
// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	c.val, c.err = fn()
	c.wg.Done()

	...
}
```

在Do方法中主要是通过waitgroup来控制的，主要流程如下：

1. 在Group中设置了一个map，如果key不存在，则实例化call(用来保存值信息)，并将key=>call的对应关系存入map中（通过mutex保证了并发安全）
1. 如果已经在调用中则key已经存在map，则wg.Wait
1. 在fn执行结束之后（在doCall方法中执行）执行wg.Done
1. 卡在第2步的方法得到执行，返回结果

其他的DoChan方法也是类似的逻辑，只是返回的是一个chan。


**文中用到的实例代码放在github上：**
[https://github.com/go-demo/singleflight-demo](https://github.com/go-demo/singleflight-demo)

