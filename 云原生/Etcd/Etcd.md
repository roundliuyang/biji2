# Etcd



`etcd` 是 `CoreOS` 团队发起的一个管理配置信息和服务发现（`Service Discovery`）的项目，在这一章里面，我们将基于 `etcd 3.x` 版本介绍该项目的目标，安装和使用，以及实现的技术。

## 简介

`etcd` 是 `CoreOS` 团队于 2013 年 6 月发起的开源项目，它的目标是构建一个高可用的分布式键值（`key-value`）数据库，基于 `Go` 语言实现。我们知道，在分布式系统中，各种服务的配置信息的管理分享，服务的发现是一个很基本同时也是很重要的问题。`CoreOS` 项目就希望基于 `etcd` 来解决这一问题。

`etcd` 目前在 [github.com/etcd-io/etcd](https://github.com/etcd-io/etcd) 进行维护。

受到 [Apache ZooKeeper](https://zookeeper.apache.org/) 项目和 [doozer](https://github.com/ha/doozerd) 项目的启发，`etcd` 在设计的时候重点考虑了下面四个要素：

- 简单：具有定义良好、面向用户的 `API` ([gRPC](https://github.com/grpc/grpc))
- 安全：支持 `HTTPS` 方式的访问
- 快速：支持并发 `10 k/s` 的写操作
- 可靠：支持分布式结构，基于 `Raft` 的一致性算法

*Apache ZooKeeper 是一套知名的分布式系统中进行同步和一致性管理的工具。*

*doozer 是一个一致性分布式数据库。*

[*Raft*](https://raft.github.io/) *是一套通过选举主节点来实现分布式系统一致性的算法，相比于大名鼎鼎的 Paxos 算法，它的过程更容易被人理解，由 Stanford 大学的 Diego Ongaro 和 John Ousterhout 提出。更多细节可以参考* [*raftconsensus.github.io*](http://raftconsensus.github.io/)*。*

一般情况下，用户使用 `etcd` 可以在多个节点上启动多个实例，并添加它们为一个集群。同一个集群中的 `etcd` 实例将会保持彼此信息的一致性。



## 安装

`etcd` 基于 `Go` 语言实现，因此，用户可以从 [项目主页](https://github.com/etcd-io/etcd) 下载源代码自行编译，也可以下载编译好的二进制文件，甚至直接使用制作好的 `Docker` 镜像文件来体验。



### 二进制文件方式下载

编译好的二进制文件都在 [github.com/etcd-io/etcd/releases](https://github.com/etcd-io/etcd/releases/) 页面，用户可以选择需要的版本，或通过下载工具下载。

例如，使用 `curl` 工具下载压缩包，并解压。

```sh
$ curl -L https://github.com/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz -o etcd-v3.4.0-linux-amd64.tar.gz

# 国内用户可以使用以下方式加快下载
$ curl -L https://download.fastgit.org/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz -o etcd-v3.4.0-linux-amd64.tar.gz

$ tar xzvf etcd-v3.4.0-linux-amd64.tar.gz
$ cd etcd-v3.4.0-linux-amd64
```

解压后，可以看到文件包括

```
$ ls
Documentation README-etcdctl.md README.md READMEv2-etcdctl.md etcd etcdctl
```

其中 `etcd` 是服务主文件，`etcdctl` 是提供给用户的命令客户端，其他文件是支持文档。

下面将 `etcd` `etcdctl` 文件放到系统可执行目录（例如 `/usr/local/bin/`）。

```
$ sudo cp etcd* /usr/local/bin/
```

默认 `2379` 端口处理客户端的请求，`2380` 端口用于集群各成员间的通信。启动 `etcd` 显示类似如下的信息：

```
$ etcd
...
2017-12-03 11:18:34.411579 I | embed: listening for peers on http://localhost:2380
2017-12-03 11:18:34.411938 I | embed: listening for client requests on localhost:2379
```

此时，可以使用 `etcdctl` 命令进行测试，设置和获取键值 `testkey: "hello world"`，检查 `etcd` 服务是否启动成功：

```
$ ETCDCTL_API=3 etcdctl member list
8e9e05c52164694d, started, default, http://localhost:2380, http://localhost:2379

$ ETCDCTL_API=3 etcdctl put testkey "hello world"
OK

$ etcdctl get testkey
testkey
hello world
```

说明 etcd 服务已经成功启动了。

>默认环境变量中 **没有设置 `ETCDCTL_API` 的值**，而 `etcdctl` 默认使用的是旧版 API（v2），很多 v3 的命令格式在 v2 下是无法使用的。
>
>你可以设置一个全局环境变量，让它在每次打开终端时自动生效：
>
>echo 'export ETCDCTL_API=3' >> ~/.bashrc
>source ~/.bashrc
>
>然后你就可以直接用：
>
>etcdctl get testkey



### Docker 镜像方式运行

镜像名称为 `quay.io/coreos/etcd`，可以通过下面的命令启动 `etcd` 服务监听到 `2379` 和 `2380` 端口。

```sh
mkdir -p /tmp/etcd-data.tmp

docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --mount type=bind,source=/tmp/etcd-data.tmp,destination=/etcd-data \
  --name etcd-latest \
  quay.io/coreos/etcd:v3.5.9 \
  /usr/local/bin/etcd \
  --name s1 \
  --data-dir /etcd-data \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://0.0.0.0:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://0.0.0.0:2380 \
  --initial-cluster s1=http://0.0.0.0:2380 \
  --initial-cluster-token tkn \
  --initial-cluster-state new \
  --log-level info \
  --logger zap \
  --log-outputs stderr
```

打开新的终端按照上一步的方法测试 `etcd` 是否成功启动。





### macOS 中运行

```shell
$ brew install etcd

$ etcd

$ etcdctl member list
```



## 集群

下面我们使用 [Docker Compose](https://yeasy.gitbook.io/docker_practice/compose) 模拟启动一个 3 节点的 `etcd` 集群。

编辑 `docker-compose.yml` 文件

```yaml
version: "3.6"
services:

  node1:
    image: quay.io/coreos/etcd:v3.4.0
    volumes:
      - node1-data:/etcd-data
    expose:
      - 2379
      - 2380
    networks:
      cluster_net:
        ipv4_address: 172.16.238.100
    environment:
      - ETCDCTL_API=3
    command:
      - /usr/local/bin/etcd
      - --data-dir=/etcd-data
      - --name
      - node1
      - --initial-advertise-peer-urls
      - http://172.16.238.100:2380
      - --listen-peer-urls
      - http://0.0.0.0:2380
      - --advertise-client-urls
      - http://172.16.238.100:2379
      - --listen-client-urls
      - http://0.0.0.0:2379
      - --initial-cluster
      - node1=http://172.16.238.100:2380,node2=http://172.16.238.101:2380,node3=http://172.16.238.102:2380
      - --initial-cluster-state
      - new
      - --initial-cluster-token
      - docker-etcd

  node2:
    image: quay.io/coreos/etcd:v3.4.0
    volumes:
      - node2-data:/etcd-data
    networks:
      cluster_net:
        ipv4_address: 172.16.238.101
    environment:
      - ETCDCTL_API=3
    expose:
      - 2379
      - 2380
    command:
      - /usr/local/bin/etcd
      - --data-dir=/etcd-data
      - --name
      - node2
      - --initial-advertise-peer-urls
      - http://172.16.238.101:2380
      - --listen-peer-urls
      - http://0.0.0.0:2380
      - --advertise-client-urls
      - http://172.16.238.101:2379
      - --listen-client-urls
      - http://0.0.0.0:2379
      - --initial-cluster
      - node1=http://172.16.238.100:2380,node2=http://172.16.238.101:2380,node3=http://172.16.238.102:2380
      - --initial-cluster-state
      - new
      - --initial-cluster-token
      - docker-etcd

  node3:
    image: quay.io/coreos/etcd:v3.4.0
    volumes:
      - node3-data:/etcd-data
    networks:
      cluster_net:
        ipv4_address: 172.16.238.102
    environment:
      - ETCDCTL_API=3
    expose:
      - 2379
      - 2380
    command:
      - /usr/local/bin/etcd
      - --data-dir=/etcd-data
      - --name
      - node3
      - --initial-advertise-peer-urls
      - http://172.16.238.102:2380
      - --listen-peer-urls
      - http://0.0.0.0:2380
      - --advertise-client-urls
      - http://172.16.238.102:2379
      - --listen-client-urls
      - http://0.0.0.0:2379
      - --initial-cluster
      - node1=http://172.16.238.100:2380,node2=http://172.16.238.101:2380,node3=http://172.16.238.102:2380
      - --initial-cluster-state
      - new
      - --initial-cluster-token
      - docker-etcd

volumes:
  node1-data:
  node2-data:
  node3-data:

networks:
  cluster_net:
    driver: bridge
    ipam:
      driver: default
      config:
      -
        subnet: 172.16.238.0/24
```

使用 `docker-compose up` 启动集群之后使用 `docker exec` 命令登录到任一节点测试 `etcd` 集群。

```sh
/ # etcdctl member list
daf3fd52e3583ff, started, node3, http://172.16.238.102:2380, http://172.16.238.102:2379
422a74f03b622fef, started, node1, http://172.16.238.100:2380, http://172.16.238.100:2379
ed635d2a2dbef43d, started, node2, http://172.16.238.101:2380, http://172.16.238.101:2379
```



## 使用 etcdctl

`etcdctl` 是一个命令行客户端，它能提供一些简洁的命令，供用户直接跟 `etcd` 服务打交道，而无需基于 `HTTP API` 方式。这在某些情况下将很方便，例如用户对服务进行测试或者手动修改数据库内容。我们也推荐在刚接触 `etcd` 时通过 `etcdctl` 命令来熟悉相关的操作，这些操作跟 `HTTP API` 实际上是对应的。

`etcd` 项目二进制发行包中已经包含了 `etcdctl` 工具，没有的话，可以从 [github.com/etcd-io/etcd/releases](https://github.com/etcd-io/etcd/releases) 下载。

`etcdctl` 支持如下的命令，大体上分为数据库操作和非数据库操作两类，后面将分别进行解释。

```
NAME:
	etcdctl - A simple command line client for etcd3.

USAGE:
	etcdctl

VERSION:
	3.4.0

API VERSION:
	3.4


COMMANDS:
	get			Gets the key or a range of keys
	put			Puts the given key into the store
	del			Removes the specified key or range of keys [key, range_end)
	txn			Txn processes all the requests in one transaction
	compaction		Compacts the event history in etcd
	alarm disarm		Disarms all alarms
	alarm list		Lists all alarms
	defrag			Defragments the storage of the etcd members with given endpoints
	endpoint health		Checks the healthiness of endpoints specified in `--endpoints` flag
	endpoint status		Prints out the status of endpoints specified in `--endpoints` flag
	watch			Watches events stream on keys or prefixes
	version			Prints the version of etcdctl
	lease grant		Creates leases
	lease revoke		Revokes leases
	lease timetolive	Get lease information
	lease keep-alive	Keeps leases alive (renew)
	member add		Adds a member into the cluster
	member remove		Removes a member from the cluster
	member update		Updates a member in the cluster
	member list		Lists all members in the cluster
	snapshot save		Stores an etcd node backend snapshot to a given file
	snapshot restore	Restores an etcd member snapshot to an etcd directory
	snapshot status		Gets backend snapshot status of a given file
	make-mirror		Makes a mirror at the destination etcd cluster
	migrate			Migrates keys in a v2 store to a mvcc store
	lock			Acquires a named lock
	elect			Observes and participates in leader election
	auth enable		Enables authentication
	auth disable		Disables authentication
	user add		Adds a new user
	user delete		Deletes a user
	user get		Gets detailed information of a user
	user list		Lists all users
	user passwd		Changes password of user
	user grant-role		Grants a role to a user
	user revoke-role	Revokes a role from a user
	role add		Adds a new role
	role delete		Deletes a role
	role get		Gets detailed information of a role
	role list		Lists all roles
	role grant-permission	Grants a key to a role
	role revoke-permission	Revokes a key from a role
	check perf		Check the performance of the etcd cluster
	help			Help about any command

OPTIONS:
      --cacert=""				verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""					identify secure client using this TLS certificate file
      --command-timeout=5s			timeout for short running command (excluding dial timeout)
      --debug[=false]				enable client-side debug logging
      --dial-timeout=2s				dial timeout for client connections
      --endpoints=[127.0.0.1:2379]		gRPC endpoints
      --hex[=false]				print byte strings as hex encoded strings
      --insecure-skip-tls-verify[=false]	skip server certificate verification
      --insecure-transport[=true]		disable transport security for client connections
      --key=""					identify secure client using this TLS key file
      --user=""					username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"			set the output format (fields, json, protobuf, simple, table)
```



### 数据库操作

数据库操作围绕对键值和目录的 CRUD （符合 REST 风格的一套操作：Create）完整生命周期的管理。

etcd 在键的组织上采用了层次化的空间结构（类似于文件系统中目录的概念），用户指定的键可以为单独的名字，如 `testkey`，此时实际上放在根目录 `/` 下面，也可以为指定目录结构，如 `cluster1/node2/testkey`，则将创建相应的目录结构。

>注：CRUD 即 Create, Read, Update, Delete，是符合 REST 风格的一套 API 操作。



#### put

```sh
$ etcdctl put /testdir/testkey "Hello world"
OK
```



#### get

获取指定键的值。例如

```sh
$ etcdctl put testkey hello
OK
$ etcdctl get testkey
testkey
hello
```

支持的选项为

`--sort` 对结果进行排序

`--consistent` 将请求发给主节点，保证获取内容的一致性





#### del

删除某个键值。例如

```
$ etcdctl del testkey
1
```





### 非数据库操作

#### watch

监测一个键值的变化，一旦键值发生更新，就会输出最新的值。

例如，用户更新 `testkey` 键值为 `Hello world`。

```sh
$ etcdctl watch testkey
PUT
testkey
2
```



#### member

通过 `list`、`add`、`update`、`remove` 命令列出、添加、更新、删除 etcd 实例到 etcd 集群中。

例如本地启动一个 `etcd` 服务实例后，可以用如下命令进行查看。

```sh
$ etcdctl member list
422a74f03b622fef, started, node1, http://172.16.238.100:2380, http://172.16.238.100:23
```