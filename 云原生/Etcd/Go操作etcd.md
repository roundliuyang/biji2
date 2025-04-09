# Go操作etcd



## 一、etcd介绍

etcd是使用Go语言开发的一个开源的、高可用的分布式key-value存储系统，可以用于配置共享和服务的注册和发现。

etcd的特点：

- 完全复制：集群中的每个节点都可以使用完整的存档
- 高可用性：etcd可用于避免硬件的单点故障或网络问题
- 一致性：每次读取都会返回跨多主机的最新写入
- 简单：包括一个定义良好、面向用户的API（gRPC）
- 安全：实现了带有可选的客户端证书身份验证的自动化TLS
- 快速：每秒10000次写入的基准速度
- 可靠：使用Raft[算法](https://zhida.zhihu.com/search?content_id=188283545&content_type=Article&match_order=1&q=算法&zhida_source=entity)实现了强一致、高可用的服务存储目录



## 二、快速入门



### 2.1 安装

我们不使用*etcd/clientv3*，**因为它与grpc最新版本不兼容**，这里我们使用官方最新推荐的方式*etcd/client/v3*

```go
go get go.etcd.io/etcd/client/v3
```



### 2.2 PUT/GET操作

```go
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/client/v3"
	"log"
	"time"
)

func main() {

	client, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatalln(err)
	}

	// PUT
	ctx, cancel := context.WithTimeout(context.Background(),5*time.Second)
	defer cancel()
	key := "/ns/service"
	value := "127.0.0.1:8000"
	_, err = client.Put(ctx, key, value)
	if err != nil {
		log.Printf("etcd put error,%v\n", err)
		return
	}

	// GET
	getResponse, err := client.Get(ctx, key)
	if err != nil {
		log.Printf("etcd GET error,%v\n", err)
		return
	}

	for _,kv := range getResponse.Kvs {
		fmt.Printf("%s=%s\n",kv.Key,kv.Value)
	}

	// /ns/service=127.0.0.1:8000
}
```



### 2.3 watch监听操作

[go.etcd.io/etcd/client/v3](https://link.zhihu.com/?target=http%3A//go.etcd.io/etcd/client/v3) 允许我们对PUT、DELETE等操作进行监听，用来获取未来更改的通知

下面代码表明：

- 创建一个watcher监听key
- 每隔2秒重新put一次key-value
- watcher函数进行监听，并打印监听结果

```go
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/client/v3"
	"log"
	"strconv"
	"time"
)

func main() {

	client, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatalln(err)
	}
	
	key := "/ns/service"

	// 监听变化
	go watcher(client,key)

	value := "127.0.0.1:800"
	ctx := context.Background()

	// 每隔2秒重新PUT一次
	for i:=0 ;i<100;i++ {
		time.Sleep(2*time.Second)
		_, err := client.Put(ctx, key, value+strconv.Itoa(i))
		if err != nil {
			log.Printf("put error %v",err)
		}
	}

}


func watcher(client *clientv3.Client,key string) {

	// 监听这个chan
	watchChan := client.Watch(context.Background(), key)

	for watchResponse := range watchChan {

		for _, event := range watchResponse.Events {
			fmt.Printf("Type:%s,Key:%s,Value:%s\n",event.Type,event.Kv.Key,event.Kv.Value)
			/**
			Type:PUT,Key:/ns/service,Value:127.0.0.1:8000
			Type:PUT,Key:/ns/service,Value:127.0.0.1:8001
			Type:PUT,Key:/ns/service,Value:127.0.0.1:8002
			Type:PUT,Key:/ns/service,Value:127.0.0.1:8003
			...
			 */
		}
	}
}
```



### 2.4 租约（有效期）

除了常规的操作外，我们还可以设置KEY-VALUE的过期时间，这里引入一个概念叫租约，将设置过期时间这一过程称为租约，租约期限到期则KEY-VALUE将删除

下面的代码表示：

- 创建一个租约，有效期为5秒
- 使用这个租约PUT一个新的KEY-VALUE
- 使用watcher监听这个KEY
- 5秒后将监听到DELETE事件，租约过期

```go
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/client/v3"
	"log"
	"time"
)

func main() {

	client, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatalln(err)
	}

	key := "/ns/service"
	value := "127.0.0.1:800"
	ctx := context.Background()

	// 获取一个租约 有效期为5秒
	leaseGrant, err := client.Grant(ctx, 5)
	if err != nil {
		log.Printf("put error %v",err)
		return
	}

	// PUT 租约期限为5秒
	_, err = client.Put(ctx, key, value, clientv3.WithLease(leaseGrant.ID))
	if err != nil {
		log.Printf("put error %v",err)
		return
	}

	// 监听变化 5秒后将监听到DELETE事件
	watcher(client,key)

}


func watcher(client *clientv3.Client,key string) {

	// 监听这个chan
	watchChan := client.Watch(context.Background(), key)

	for watchResponse := range watchChan {
		for _, event := range watchResponse.Events {
			fmt.Printf("Type:%s,Key:%s,Value:%s\n",event.Type,event.Kv.Key,event.Kv.Value)
			// Type:DELETE,Key:/ns/service,Value:
		}
	}

}
```



### 2.5 续租 KeepAlive

通过Grant，我们可以创建一个带有过期的时间的租约，但是，有些时候我们可能需要续租。通俗一点可以理解为：只要这个租约和ETCD保持(连接)KeepAlive，那么就不会过期，如果断开，则会在5秒后过期。又或者可以理解为**[心跳机制](https://zhida.zhihu.com/search?content_id=188283545&content_type=Article&match_order=1&q=心跳机制&zhida_source=entity)**。

```go
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/client/v3"
	"log"
	"time"
)

func main() {

	client, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatalln(err)
	}

	key := "/ns/service"
	value := "127.0.0.1:800"
	ctx := context.Background()

	// 获取一个租约 有效期为5秒
	leaseGrant, err := client.Grant(ctx, 5)
	if err != nil {
		log.Printf("grant error %v",err)
		return
	}

	// PUT 租约期限为5秒
	_, err = client.Put(ctx, key, value, clientv3.WithLease(leaseGrant.ID))
	if err != nil {
		log.Printf("put error %v",err)
		return
	}

	// 续租
	keepaliveResponseChan, err := client.KeepAlive(ctx, leaseGrant.ID)
	if err != nil {
		log.Printf("KeepAlive error %v",err)
		return
	}
	
	for {
		ka := <-keepaliveResponseChan
		fmt.Println("ttl:", ka.TTL)
		//ttl: 5
		//ttl: 5
		//ttl: 5
	}
}
```

