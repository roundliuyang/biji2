# 性能测试 —— MySQL 基准测试



## 1. 概述

MySQL 作为我们日常开发中，使用最多的数据库（基本没有之一），但我们很多开发人员对 MySQL 的性能规格了解得非常少。所以，本文我们想一起来，对 MySQL 本身做一个性能基准测试。

在开始基准测试之前，我们比较快捷的知道，MySQL 大体的性能规格，从各大云厂商提供的 MySQL 云服务。

> 艿艿：此处，是不是默默的宣传了下国内的云服务。

- 阿里云 MySQL 5.6 ： https://help.aliyun.com/document_detail/53637.html
- 阿里云 MySQL 5.7 ： https://help.aliyun.com/document_detail/109376.html
- 华为云 MySQL 5.6 ： https://support.huaweicloud.com/pwp-rds/rds_swp_mysql_02.html
- 华为云 MySQL 5.7 ： https://support.huaweicloud.com/pwp-rds/rds_swp_mysql_03.html
- 腾讯云 MySQL 未知版本 ： https://cloud.tencent.com/document/product/236/8842
- 百度云 MySQL 5.6 ：https://cloud.baidu.com/doc/RDS/Performance_whitepaper.html 只提供测试方法，不提供性能规格
- UCloud MySQL ： 未提供性能规格文档
- 美团云 MySQL ： 未提供性能规格文档

## 2. 性能指标

通过我们看各大厂商提供的指标，我们不难发现，主要是 4 个指标：

- TPS ：Transactions Per Second ，即数据库每秒执行的事务数，以 commit 成功次数为准。
- QPS ：Queries Per Second ，即数据库每秒执行的 SQL 数（含 insert、select、update、delete 等）。
- RT ：Response Time ，响应时间。包括平均响应时间、最小响应时间、最大响应时间、每个响应时间的查询占比。比较需要重点关注的是，前 95-99% 的最大响应时间。因为它决定了大多数情况下的短板。
- Concurrency Threads ：并发量，每秒可处理的查询请求的数量。

> 如果对基准测试不是很理解的胖友，可以看下 [《详解 MySQL 基准测试和 sysbench 工具》](http://www.cnblogs.com/kismetv/p/7615738.html#t1) 的第一部分**基准测试简介**。

总结来说，实际就是 2 个维度：

- 吞吐量
- 延迟

## 3. 测试工具

MySQL 的性能测试工具还是比较多的，使用最多的是 sysbench 和 mysqlslap 。本文，我们也会使用这两个工具，进行 MySQL 性能基准测试。

具体的介绍，我们放在下面的章节中。如果对其他测试工具感兴趣，可以看看如下两篇文章：

- [《数据库性能测试》](https://dbaplus.cn/news-11-148-1.html) 强烈推荐，提供了很多的 MySQL 硬件方面的性能优化的方向。
- [《测试 MySQL 性能的几款工具》](https://yq.aliyun.com/articles/551688)

考虑到有些胖友可能不知道如何安装 MySQL 5.7 版本，可以参考 [《在 CentOS7 上使用yum安装 MySQL 5.7》](https://blog.frognew.com/2017/05/yum-install-mysql-5.7.html) 文章。



## 4. sysbench

> FROM [《性能测试工具 sysbench》](https://www.oschina.net/p/sysbench)
>
> sysbench 是一个模块化的、跨平台、多线程基准测试工具，主要用于评估测试各种不同系统参数下的数据库负载情况。它主要包括以下几种方式的测试：
>
> 1. CPU 性能
> 2. 磁盘 IO 性能
> 3. 调度程序性能
> 4. 内存分配及传输速度
> 5. POSIX 线程性能
> 6. 数据库性能([OLTP](https://baike.baidu.com/item/OLTP) 基准测试)
>
> 目前 sysbench 主要支持 MySQL、PgSQL、Oracle 这 3 种数据库。

sysbench 也是目前 DBA 最喜欢用来做 MySQL 性能的测试工具。

另外，我们可以发现，云厂商不约而同的使用 sysbench 作为基准测试工具。所以啊，sysbench 可能是比较正确的选型。

下面，我们就开始我们的 sysbench 基准测试之旅。考虑到现在很多公司都是采用阿里云为主，所以我们就参考阿里云的[测试方法](https://help.aliyun.com/document_detail/53632.html) 。



### 4.1 测试环境



>艿艿：经过几轮的压测，测试出来的 TPS/QPS 巨差，因为没标准去类比，所以我就删除了下面这段，重新去跟老婆大人申请零花钱，买了阿里云服务器去测试。如下是几轮的测试结果：
>
>线程数 / 单表数据量 / 表数 / QPS / TPS
>
>- 32 / 1W / 10 / 4000 / 200
>- 32 / 10W / 10 / 1000 / 50
>- 32 / 100W / 10 / 100 / 5
>
>可能和我没做 MySQL 服务器调优配置有关系，走的默认配置。 也可能和服务器的硬盘太差有关系，IOPS 才 100 。如果对 IOPS 的计算逻辑不了解的胖友，感兴趣的，可以看看 [《(转）MySQL TPS 和 QPS 的统计和 IOPS》](https://jackyrong.iteye.com/blog/1747517) 。
>
>作为一个热爱死磕的艿艿，在 V2EX 的讨论，找到一个通病相连的 [《MySQL 的性能指标在什么情况下是正常的呢？》](https://www.v2ex.com/t/318906) 。
>
>因为抠门，我们拿了手头的刀片机，作为测试服务器。并且，上面实际还跑了蛮多其他服务的，嘿嘿。具体配置如下：
>
>- 系统 ：CentOS Linux release 7.4.1708 (Core)
>
>- CPU ：4 Intel(R) Xeon(R) CPU E5-2407 0 @ 2.20GHz
>
>- 内存 ：4 * 16G DDR3 1333 MHZ
>
>- 磁盘 ：512 GB 希捷机械硬盘
>
>  > 通过 `hdparm -i /dev/sda` 命令查看，应该是 ST500DM002-1BD142 型号。
>  >
>  > T T 瞬间心凉凉，即没有 SSD ，也没有 raid ，穷苦
>
>- MySQL ：Ver 14.14 Distrib 5.7.26, for Linux (x86_64) using EditLine wrapper
>
>  > 简单来说，就是 5.7 版本。
>
>



**艿艿：下面，我们拿一台阿里云的 ECS 服务器，进行测试。**

- 型号 ：ecs.c5.xlarge

  > 艿艿：和我一样抠门（穷）的胖友，可以买竞价类型服务器，使用完后，做成镜像。等下次需要使用的时候，恢复一下。HOHO 。

- 系统 ：CentOS 7.6 64位

- CPU ：4 核

- 内存 ：8 GB

- 磁盘 ：40 GB ESSD 云盘

  > 艿艿：“土豪”一点的胖友，可以买更大的磁盘容量，因为越大的容量，越高的 IOPS 。

- MySQL ：Ver 14.14 Distrib 5.7.26, for Linux (x86_64) using EditLine wrapper

  > 简单来说，就是 5.7 版本。

### 4.2 安装工具

我们以 Linux 为例。如果没有 Linux 环境的胖友，可以使用 [VirtualBox](https://www.virtualbox.org/) 安装一个 Linux 虚拟机环境。

```shell
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
sudo yum -y install sysbench
```

本文，我们使用的 sysbench 是 `1.0.17-2.el7` 版本。

```shell
-bash-4.2# sysbench --version
sysbench 1.0.17
```

下面，我们如下三个步骤，进行测试：

1. 准备数据 prepare
2. 执行测试 run
3. 清理数据 clean



### 4.3 准备数据

> 需要注意，sysbench 1.0.17 版本，和我们在网上看到的 sysbench 0.5 的版本，命令上有一些差异。

```shell
sysbench oltp_common.lua --time=3600 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=buzhidao --mysql-db=sbtest --table-size=10000000 --tables=64 --threads=32 --events=999999999 --report-interval prepare
```

让我们一起看看每个参数的意思：

> 如下参数的介绍，我们主要参考了这两篇文章，想要详细了解更多参数的胖友，可以来看看：
>
> - [《Sysbench 1.0.15 安装及使用》](https://blog.51cto.com/bilibili/2173243)
> - [《基准测试工具 Sysbench》](https://www.jianshu.com/p/4a37a6a452d9)

- `oltp_common.lua` ：执行的测试脚本。因为我们使用 yum 进行安装，所以胖友需要 `cd /usr/share/sysbench/` 目录下，看到 sysbench 自带的 lua 测试脚本。

- `--time` ：最大的总执行时间，以秒为单位，默认为 10 秒。

- `--events` ：最大允许的事件个数，默认为 0 个。

  > 应该和 `--time` 互相形成最大的执行时间与次数。

- MySQL 相关参数

  - `--mysql-host` ：MySQL server host 。
  - `--mysql-port` ：MySQL server port 。
  - `--mysql-user` ：MySQL server 账号。
  - `--mysql-password` ：MySQL server 密码。
  - `--mysql-db` ：MySQL Server 数据库名。

- `--table-size` ：表记录条数。

- `--tables` ：表名。

- `--threads` ：要使用的线程数，默认 1 个。

- `--report-interval` ：以秒为单位定期报告具有指定间隔的中间统计信息，默认为 0 ，表示禁用中间报告。

  > 艿艿：这个一定要记得设置下，例如说设置个 10s ，不然一脸懵逼。

- `prepare` ：执行准备数据。

因为阿里云提供的需要生成的数据较多，所以最后艿艿将命令修改成如下：

```shell
sysbench oltp_common.lua --time=300 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=MyNewPass4! --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=32 --events=999999999   prepare
```

- 主要调整了 `--table-size=1000000` 和 `--tables=10` 参数。

执行命令后，会自动生成数据库的表、和数据。如下：

```shell
[root@iZuf6hci646px19gg3hpuwZ sysbench]# sysbench oltp_common.lua --time=300 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=MyNewPass4! --mysql-db=sbtest --table-size=1000000 --tables=10--threads=32 --events=999999999   prepare
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Creating table 'sbtest1'...
Inserting 1000000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 1000000 records into 'sbtest2'
Creating a secondary index on 'sbtest2'...
Creating table 'sbtest3'...
Inserting 1000000 records into 'sbtest3'
Creating a secondary index on 'sbtest3'...
Creating table 'sbtest4'...
Inserting 1000000 records into 'sbtest4'
Creating a secondary index on 'sbtest4'...
Creating table 'sbtest5'...
Inserting 1000000 records into 'sbtest5'
Creating a secondary index on 'sbtest5'...
Creating table 'sbtest6'...
Inserting 1000000 records into 'sbtest6'
Creating a secondary index on 'sbtest6'...
Creating table 'sbtest7'...
Inserting 1000000 records into 'sbtest7'
Creating a secondary index on 'sbtest7'...
Creating table 'sbtest8'...
Inserting 1000000 records into 'sbtest8'
Creating a secondary index on 'sbtest8'...
Creating table 'sbtest9'...
Inserting 1000000 records into 'sbtest9'
Creating a secondary index on 'sbtest9'...
Creating table 'sbtest10'...
Inserting 1000000 records into 'sbtest10'
Creating a secondary index on 'sbtest10'...
```

- 耐心等待，喝口大可乐压压惊。

### 4.4 执行测试

```shell
sysbench oltp_read_write.lua --time=300 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=MyNewPass4! --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=16 --events=999999999  --report-interval=10  run
```

- `oltp_read_write.lua` ：执行的测试脚本。此时，我们在 `/usr/share/sysbench/` 下，寻找我们想要测试的场景。

  > `oltp_read_write.lua` ，表示混合读写，在一个事务中，默认比例是：`select:update_key:update_non_key:delete:insert = 14:1:1:1:1` 。这也是为什么，我们测试出来的 TPS 和 QPS 的比例，大概在 1:18~20 左右。相当于说，一个事务中，有 18 个读写操作。

- `run` ：执行测试。

执行后，效果如下：

```shell
[root@iZuf6hci646px19gg3hpuwZ sysbench]# sysbench oltp_read_write.lua --time=300 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=MyNewPass4! --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=16 --events=999999999 --rate=0 --histogram=on  --report-interval=10  run
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 16
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 16 tps: 784.14 qps: 15705.59 (r/w/o: 10997.12/3138.58/1569.89) lat (ms,95%): 58.92 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 16 tps: 842.11 qps: 16845.24 (r/w/o: 11791.27/3369.75/1684.22) lat (ms,95%): 52.89 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 16 tps: 856.99 qps: 17124.49 (r/w/o: 11984.72/3425.78/1713.99) lat (ms,95%): 51.02 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 16 tps: 835.42 qps: 16714.15 (r/w/o: 11702.34/3340.97/1670.83) lat (ms,95%): 54.83 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 16 tps: 851.39 qps: 17022.37 (r/w/o: 11913.74/3405.85/1702.78) lat (ms,95%): 52.89 err/s: 0.00 reconn/s: 0.00
... 省略
```

如下，是艿艿跑出来的两组测试结果：

- 1、32 线程 + 每个表 100w 数据：

  ```shell
  SQL statistics:
      queries performed:
          read:                            3950058
          write:                           1128588
          other:                           564294
          total:                           5642940
      transactions:                        282147 (940.40 per sec.)
      queries:                             5642940 (18807.93 per sec.)
      ignored errors:                      0      (0.00 per sec.)
      reconnects:                          0      (0.00 per sec.)
  
  General statistics:
      total time:                          300.0281s
      total number of events:              282147
  
  Latency (ms):
           min:                                    2.59
           avg:                                   34.02
           max:                                  678.96
           95th percentile:                       86.00
           sum:                              9599340.27
  
  Threads fairness:
      events (avg/stddev):           8817.0938/45.51
      execution time (avg/stddev):   299.9794/0.01
  ```

  - **940 TPS + 18807 QPS + 34ms 延迟**

- 2、16 线程 + 每个表 100w 数据：

  ```shell
  SQL statistics:
      queries performed:
          read:                            2336754
          write:                           667644
          other:                           333822
          total:                           3338220
      transactions:                        166911 (556.31 per sec.)
      queries:                             3338220 (11126.11 per sec.)
      ignored errors:                      0      (0.00 per sec.)
      reconnects:                          0      (0.00 per sec.)
  
  General statistics:
      total time:                          300.0331s
      total number of events:              166911
  
  Latency (ms):
           min:                                    2.54
           avg:                                   28.76
           max:                                  206.30
           95th percentile:                       70.55
           sum:                              4799705.19
  
  Threads fairness:
      events (avg/stddev):           10431.9375/49.29
      execution time (avg/stddev):   299.9816/0.00
  ```

  - 556 TPS + 11126 QPS + 28ms 延迟

### 4.5 清理数据

```shell
sysbench oltp_read_write.lua --time=300 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=MyNewPass4! --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=16 --events=999999999  --report-interval=10  cleanup
```

- `cleanup` ：执行清理数据。

开始打扫战场，嘻嘻。效果如下：

```shell
[root@iZuf6hci646px19gg3hpuwZ sysbench]# sysbench oltp_read_write.lua --time=300 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=MyNewPass4! --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=16 --events=999999999 --rate=0 --histogram=on  --report-interval=10  runC^C
[root@iZuf6hci646px19gg3hpuwZ sysbench]# sysbench oltp_read_write.lua --time=300 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=MyNewPass4! --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=16 --events=999999999  --report-interval=10  cleanup
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Dropping table 'sbtest1'...
Dropping table 'sbtest2'...
Dropping table 'sbtest3'...
Dropping table 'sbtest4'...
Dropping table 'sbtest5'...
Dropping table 'sbtest6'...
Dropping table 'sbtest7'...
Dropping table 'sbtest8'...
Dropping table 'sbtest9'...
Dropping table 'sbtest10'...
```



### 4.6 推荐文章

- [《sysbench 在美团点评中的应用》](https://cloud.tencent.com/developer/article/1058214) 强烈推荐。

  ```
  艿艿：该文章提供了几个参数，胖友可以试试：
  
  --warmup_time ：预热时间，预防冷数据对测试结果的影响。
  这个参数加下也是有必要的，因为线上的数据，实际是一直在跑的，不会处于冷数据的状态。
  -rate ：指定数量多少事件(事务)平均每秒钟应该执行的所有线程。0(默认)意味着无限的速率，即事件尽快执行。
  不是很理解这个参数，不过确实增加了这个参数，QPS 和 TPS 都有一定的提升。
  -histogram ：输出测试过程中系统响应时间的分布。
  增加该参数，执行结果会多一个柱状图结果。
  percentile ：在延迟统计数据中计算的百分点 (1-100)，使用特殊值 0 来禁用百分比计算，默认为 95 。
  因为执行过程中，难免会发生锁等情况，导致有一些执行结果会有比较大的延迟，通过抛弃它们，让结果更加精准。
  ```

- [《使用 sysbench 对 mysql 压力测试》](https://segmentfault.com/a/1190000004866961) 更加详细。

- [《基准测试工具 Sysbench》](https://www.jianshu.com/p/4a37a6a452d9)

  >受限于本文仅仅对 MySQL 进行基准测试，所以并没有骚聊 sysbench 对 CPU、磁盘 IO、内存等等的测试，感兴趣的胖友，可以看看。

- [《【非官方】TiDb@腾讯云使用及性能测试》](https://zhuanlan.zhihu.com/p/30572262)

- [《知己知彼 – 对 Aurora 进行压力测试》](https://aws.amazon.com/cn/blogs/china/aurora-test/)



## 5. mysqlslap

>FROM [《MySQL压力测试工具 mysqlslap》](https://www.oschina.net/p/mysqlslap)
>
>mysqlslap 是一个 MySQL 官方提供的压力测试工具。

比较大的优势，在于 mysqlslap 是 MySQL 官方所提供，并且提供多种引擎的性能测试。

> 艿艿：更加喜好 sysbench ，所以不会 mysqlslap 会写的相对简单一些。

### 5.1 测试过程

相比 sysbench 来说，mysqlslap 的测试过程还是比较简洁的，一个命令，即可完成整个过程。如下：

```shell
mysqlslap --concurrency=16,32 --iterations=3 --number-int-cols=1 --number-char-cols=2 --auto-generate-sql --auto-generate-sql-add-autoincrement --engine=innodb --number-of-queries=10000 --create-schema=sbtest2 -uroot -pMyNewPass4!
```

>如下参数的介绍，我们主要参考了这两文章，想要详细了解更多参数的胖友，可以来看看：
>
>- [《MySQL 高性能压力测试》](https://blog.51cto.com/chenhao6/1314418)

- `--concurrency` ：并发量，也就是模拟多少个客户端同时执行命令。可指定多个值，以逗号或者 `–delimiter` 参数指定的值做为分隔符
- `--iterations` ：测试执行的迭代次数。
- `--number-int-cols` ：自动生成的测试表中包含多少个数字类型的列，默认 1 。此处设置为 1 的原因是，因为我们上面 sysbench 我们生成了一个 int 类型的字段。
- `--number-char-cols` ：自动生成的测试表中包含多少个字符类型的列，默认 1 。此处设置为 2 的原因是，因为我们上面 sysbench 我们生成了一个 char 类型的字段。
- `--auto-generate-sql` ：自动生成测试表和数据。这个命令，带来的效果，就类似 sysbench 命令的 prepare 指令。
  - `--auto-generate-sql-add-autoincrement` ：增加 auto_increment 一列。
  - 如果想看，生成的具体脚本，可以用 `–only-print` 指令，只打印测试语句而不实际执行。
- `--engine` ：创建测试表所使用的存储引擎，可指定多个。
- `--number-of-queries` ：总的测试查询次数(并发客户数×每客户查询次数)。
- `--create-schema` ：测试的 schema ，MySQL中 schema 也就是 database 数据库名。
- `-uroot -pMyNewPass4!` ：设置 MySQL 账号和密码。

执行命令后，效果如下图：

```shell
[root@iZuf6hci646px19gg3hpuwZ sysbench]# mysqlslap --concurrency=16,32 --iterations=3 --number-int-cols=1 --number-char-cols=2 --auto-generate-sql --auto-generate-sql-add-autoincrement --engine=innodb --number-of-queries=10000 --create-schema=sbtest2 -uroot -pMyNewPass4!
mysqlslap: [Warning] Using a password on the command line interface can be insecure.
Benchmark
	Running for engine innodb
	Average number of seconds to run all queries: 0.489 seconds
	Minimum number of seconds to run all queries: 0.486 seconds
	Maximum number of seconds to run all queries: 0.496 seconds
	Number of clients running queries: 16
	Average number of queries per client: 625

Benchmark
	Running for engine innodb
	Average number of seconds to run all queries: 0.379 seconds
	Minimum number of seconds to run all queries: 0.377 seconds
	Maximum number of seconds to run all queries: 0.382 seconds
	Number of clients running queries: 32
	Average number of queries per client: 312
```

- 第一个，使用 16 个线程（客户端），平均延迟在 0.489 秒。
- 第二个，使用 32 个线程（客户端），平均延迟在 0.379 秒。

相比来说，mysqlslap 不提供 QPS/TPS 的统计，需要写脚本从 MySQL 统计，或者搭配其它监控工具（例如说，Prometheus MySQL Exporter）。

### 5.2 推荐文章

因为本文确实对 mysqlslap 写的简略，所以可以看看如下几篇文章：

- [《MySQL 性能测试经验》](https://cloud.tencent.com/developer/article/1004894)
- [《MySQL 高性能压力测试》](https://blog.51cto.com/chenhao6/1314418)
- [《mysqlslap 使用总结》](https://my.oschina.net/moooofly/blog/152547)
- [《MySQL 性能测试&压力测试 - mysqlslap》](http://www.xiaot123.com/post/mysqlslap)



## 666. 彩蛋

因为本文并未讲 MySQL 性能优化相关的内容，所以需要胖友自己去寻找一些靠谱的资料，继续进行学习。当然，艿艿还是会不断整理一些，写的不错的 MySQL 性能优化相关的内容：

- [《MySQL 性能调优与测试》](https://segmentfault.com/a/1190000011687570)

最后，不得不感叹，SSD 硬盘，对 MySQL 等存储服务的巨大收益，特别是使用普通机械磁盘，还是 7500 转的，那个性能渣渣。

另外，在给 MySQL 性能基准测试，可以搭配 Prometheus 等监控系统，通过监控大盘，看看具体的[主机性能](http://www.iocoder.cn/images/Performance-Testing/2019-01-01/01.jpg)情况，[MySQL 性能](http://www.iocoder.cn/images/Performance-Testing/2019-01-01/02.png)情况。

> 艿艿：上述两个链接，点进去就是监控大盘。