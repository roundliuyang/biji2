# ELK(Elasticsearch + Logstash + Kibana) 极简入门





## 1. 概述

在线上问题排查时，通过日志来定位是经常使用的手段之一，甚至是最有效的。

线上服务为了实现高可用往往采用多节点部署，又或者随着项目愈发复杂会考虑微服务架构，导致日志分散在不同的服务器上，导致排查一个问题，需要登录多台服务器，查询在其上的日志，非常繁琐且低效。

所以，此时我们需要一个统一的实时【**日志服务**】，将我们需要的日志全部**收集**在一起，并提供灵活的**查询**功能。一般来说，一个完整的日志服务，需要提供如下 5 个功能：

- 1、收集 ：能够采集多个来源的日志数据。
- 2、传输 ：能够稳定的把日志数据传输到日志服务。
- 3、存储 ：能够存储海量的日志数据。
- 4、查询 ：能够灵活且高效的查询日志数据，并提供一定的分析能力。
- 5、告警 ：能够提供提供告警功能，通知开发和运维等等。

### 1.1 解决方案

目前，市面上有非常多的日志服务的解决方案。例如说：

- 开源

  解决方案

  - [ELK](https://www.elastic.co/what-is/elk-stack)
  - [Apache Chukwa](http://chukwa.apache.org/)
  - [Apache Kafka](https://github.com/apache/kafka)
  - [Cloudera Fluentd](https://github.com/fluent/fluentd)
  - [Syslog](https://en.wikipedia.org/wiki/Syslog)、[Rsyslog](https://www.rsyslog.com/)、[Syslog-ng](https://www.syslog-ng.com/)
  - [Facebook Scribe](https://github.com/facebookarchive/scribe)

- 商业

  解决方案

  - [阿里云 SLS](https://cn.aliyun.com/product/sls)、[腾讯云 CLS](https://cloud.tencent.com/product/cls)、[华为云 LTS](https://www.huaweicloud.com/product/lts.html)
  - [Splunk](https://www.splunk.com/zh-hans_cn)

我们目前线上采用阿里云 SLS 日志服务，主要考虑使用方便，成本合算。阿里云**打钱**~

目前采用最多的日志服务的解决方案，是 ELK 搭建的日志服务。😈 艿艿也问了一圈身边的胖友，验证无误。

### 1.2 ELK

那么 ELK 是什么呢？我们来看看其官方 [Elastic NV](https://en.wikipedia.org/wiki/Elastic_NV) 对其的定义：

> FROM <https://www.elastic.co/cn/what-is/elk-stack>
>
> “ELK” 是三个开源项目的首字母缩写，这三个项目分别是：Elasticsearch、Logstash 和 Kibana。
>
> - Elasticsearch 是一个搜索和分析引擎。
> - Logstash 是服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到诸如 Elasticsearch 等“存储库”中。
> - Kibana 则可以让用户在 Elasticsearch 中使用图形和图表对数据进行可视化。

即整体架构，如下图所示：![ELK 最简架构](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/01.png)

下面，我们就来使用 ELK 搭建一个日志服务。

## 2. Elasticsearch 搭建

> FROM <https://www.elastic.co/cn/products/elasticsearch>
>
> Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。 作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

参考[《Elasticsearch 极简入门》](http://www.iocoder.cn/Elasticsearch/install/?self)的[「1. 单机部署」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)小节，搭建一个 Elasticsearch 单机服务。

不过要**注意**，本文使用的是 Elasticsearch `7.5.1` 版本。

## 3. Logstash 搭建

> FROM <https://www.elastic.co/cn/products/logstash>
>
> Logstash 是开源的服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到您最喜欢的“存储库”中。

### 3.1 下载

打开 [Logstash 下载页面](https://www.elastic.co/cn/downloads/logstash)，选择想要的 Logstash 版本。这里，我们选择 `7.5.1` 最新版本。



```
# 创建目录
$ mkdir -p /Users/yunai/Logstash
$ cd /Users/yunai/Logstash

# 下载
$ wget https://artifacts.elastic.co/downloads/logstash/logstash-7.5.1.zip

# 解压
$ unzip logstash-7.5.1.zip
$ cd logstash-7.5.1

# 查看目录
$ ls- ls
   8 -rwxr-xr-x@  1 yunai  staff    2276 Dec 17 00:54 CONTRIBUTORS
  16 -rwxr-xr-x@  1 yunai  staff    4097 Dec 17 00:54 Gemfile
  48 -rwxr-xr-x@  1 yunai  staff   22792 Dec 17 00:54 Gemfile.lock
  32 -rwxr-xr-x@  1 yunai  staff   13675 Dec 17 00:54 LICENSE.txt
1584 -rwxr-xr-x@  1 yunai  staff  808305 Dec 17 00:54 NOTICE.TXT
   0 drwxr-xr-x  18 yunai  staff     576 Jan  1 08:35 bin # 执行文件
   0 drwxr-xr-x   8 yunai  staff     256 Jan  1 08:35 config # 配置文件
   0 drwxr-xr-x@  2 yunai  staff      64 Dec 17 00:54 data
   0 drwxr-xr-x   6 yunai  staff     192 Jan  1 08:35 lib
   0 drwxr-xr-x   6 yunai  staff     192 Jan  1 08:35 logstash-core
   0 drwxr-xr-x   5 yunai  staff     160 Jan  1 08:35 logstash-core-plugin-api
   0 drwxr-xr-x   5 yunai  staff     160 Jan  1 08:35 modules
   0 drwxr-xr-x   3 yunai  staff      96 Jan  1 08:35 tools
   0 drwxr-xr-x   4 yunai  staff     128 Jan  1 08:36 vendor
   0 drwxr-xr-x  14 yunai  staff     448 Jan  1 08:36 x-pack
```



### 3.2 配置文件

在 `config` 目录下，提供了 Logstash 的配置文件。我们来看看：



```
# 查看 config 目录
$ ls -ls config/
 8 -rwxr-xr-x@ 1 yunai  staff  2019 Dec 17 00:54 jvm.options
16 -rwxr-xr-x@ 1 yunai  staff  7482 Dec 17 00:54 log4j2.properties
 8 -rwxr-xr-x@ 1 yunai  staff   342 Dec 17 00:54 logstash-sample.conf
24 -rwxr-xr-x@ 1 yunai  staff  8372 Dec 17 00:54 logstash.yml
 8 -rwxr-xr-x@ 1 yunai  staff  3146 Dec 17 00:54 pipelines.yml
 8 -rwxr-xr-x@ 1 yunai  staff  1696 Dec 17 00:54 startup.options
```



其中，`logstash-sample.conf` 配置文件，是 Logstash 提供的 Pipeline 配置的示例。我们再来看看：



```
$ cat config/logstash-sample.conf

# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.

input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
}
```



在 Logstash 中，我们通过定义了一个 Logstash 管道（Logstash Pipeline），来读取、过滤、输出数据。一个 Logstash Pipeline 包含**三部分**，如下图所示：![Logstash Pipeline](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/9df53588c96658d66d2d286f8fa272e0)

- 【必选】输入（Input） 数据（包含但不限于日志）往往都是以不同的形式、格式存储在不同的系统中，而 Logstash 支持从多种数据源中收集数据（File、Syslog、MySQL、消息中间件等等）。
- 【可选】过滤器（Filter） ：实时解析和转换数据，识别已命名的字段以构建结构，并将它们转换成通用格式。
- 【必选】输出（Output） ：Elasticsearch 并非存储的唯一选择，Logstash 提供很多输出选择。

在 `logstash-sample.conf` 配置文件中，我们配置了 **Input** 和 **Output**。

- Output 比较好理解，写入数据到[「2. Elasticsearch 搭建」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)存储器中。
- Input 定义了 5044 端口，接收[「4. Beats 搭建」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)的数据。

在采集日志数据时，我们需要在服务器上安装一个 Logstash。不过 Logstash 是基于 JVM 的**重量级**的采集器，对系统的 CPU、内存、IO 等等资源占用非常高，这样可能影响服务器上的其它服务的运行。所以，[Elastic NV](https://en.wikipedia.org/wiki/Elastic_NV) 推出 Beats ，基于 Go 的**轻量级**采集器，对系统的 CPU、内存、IO 等等资源的占用基本可以忽略不计。

因此，本文的示例就变成了 ELFK 。其中，Beats 负责**采集**数据，并通过**网路传输**给 Logstash。即整体架构，如下图所示：![ELFK 最简架构](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/03.png)

更多关于 Beats 的内容，我们会在[「4. Beats 搭建」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)看到。

### 3.3 启动

执行 `nohup bin/logstash -f config/logstash-sample.conf &` 命令，后台启动 Logstash 服务。

可以通过 `nohup.out` 日志，查看启动是否成功。日志内容如下：



```
# 注释：Beats 5044 端口
[2020-01-01T10:12:04,262][INFO ][logstash.inputs.beats    ][main] Beats inputs: Starting input listener {:address=>"0.0.0.0:5044"}
[2020-01-01T10:12:04,271][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
[2020-01-01T10:12:04,349][INFO ][org.logstash.beats.Server][main] Starting server on port: 5044
[2020-01-01T10:12:04,351][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}

# 注释：Logstash 9600 端口
[2020-01-01T10:12:04,571][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```



## 4. Beats 搭建

> FROM <https://www.elastic.co/cn/products/beats>
>
> 轻量型数据采集器 ：Beats 平台集合了多种单一用途数据采集器。它们从成百上千或成千上万台机器和系统向 Logstash 或 Elasticsearch 发送数据。

在 Beats 发布之后，[Elastic NV](https://en.wikipedia.org/wiki/Elastic_NV) 重新定义 ELK ，**升级**成 **Elastic Stack**。如下图：![Elastic Stack](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/04.png)

Beats 是一个全品类采集器的**系列**，包含多个：

- [Filebeat](https://www.elastic.co/cn/products/beats/filebeat) ：轻量型日志采集器。
- [Metricbeat](https://www.elastic.co/cn/products/beats/metricbeat) ：轻量型指标采集器。
- [Packetbeat](https://www.elastic.co/cn/products/beats/packetbeat) ：轻量型网络数据采集器。
- [Winlogbeat](https://www.elastic.co/cn/products/beats/winlogbeat) ：轻量型 Windows 事件日志采集器。
- [Auditbeat](https://www.elastic.co/cn/products/beats/auditbeat) ：轻量型审计日志采集器。
- [Heartbeat](https://www.elastic.co/cn/products/beats/heartbeat) ：面向运行状态监测的轻量型采集器。
- [Functionbeat](https://www.elastic.co/cn/products/beats/functionbeat) ：面向云端数据的无服务器采集器。

本小节，我们使用 **Filebeat**，采集日志文件。

具体采集的日志文件，胖友可以先从 <https://static.iocoder.cn/spring.log> 下载，艿艿预先生成好了一个 Spring Boot 启动时打印的日志文件。

### 4.1 下载

打开 [Filebeat 下载页面](https://www.elastic.co/cn/downloads/beats/filebeat)，选择想要的 Filebeat 版本。这里，我们选择 `7.5.1` 最新版本。



```
# 创建目录
$ mkdir -p /Users/yunai/Filebeat
$ cd /Users/yunai/Filebeat

# 下载
$ wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.5.1-darwin-x86_64.tar.gz

# 解压
$ tar -zxvf filebeat-7.5.1-darwin-x86_64.tar.gz
$ cd filebeat-7.5.1-darwin-x86_64

# 查看目录
$ ls- ls
    32 -rw-r--r--@  1 yunai  staff     13675 Dec 17 05:15 LICENSE.txt
   592 -rw-r--r--@  1 yunai  staff    302830 Dec 17 05:16 NOTICE.txt
     8 -rw-r--r--@  1 yunai  staff       802 Dec 17 05:58 README.md
   624 -rw-r--r--@  1 yunai  staff    317651 Dec 17 05:56 fields.yml
174688 -rwxr-xr-x@  1 yunai  staff  89437104 Dec 17 09:56 filebeat # 执行文件
   168 -rw-r--r--@  1 yunai  staff     83274 Dec 17 05:56 filebeat.reference.yml
    24 -rw-------@  1 yunai  staff      8236 Dec 17 05:56 filebeat.yml # 配置文件
     0 drwxr-xr-x@  3 yunai  staff        96 Dec 17 05:56 kibana
     0 drwxr-xr-x@ 38 yunai  staff      1216 Dec 17 05:56 module
     0 drwxr-xr-x@ 37 yunai  staff      1184 Dec 17 05:56 modules.d
```



### 4.2 配置文件

使用 `vi filebeat.yml` 命令，编辑 Filebeat 配置文件。主要修改如下行：



```
#=========================== Filebeat inputs =============================
filebeat.inputs:
- type: log
  # Change to true to enable this input configuration.
  enabled: true
  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    # - /var/log/*.log
    #- c:\programdata\elasticsearch\logs\*
    - /Users/yunai/logs/spring.log # 配置我们要读取的 Spring Boot 应用的日志

#================================ Outputs =====================================
#-------------------------- Elasticsearch output ------------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
  # hosts: ["localhost:9200"]

#----------------------------- Logstash output --------------------------------
output.logstash:
  # The Logstash hosts
  hosts: ["localhost:5044"]
```



- ```
  filebeat.inputs
  ```

   

  配置项，设置 Filebeat 读取的日志来源。该配置项是

  数组

  类型，可以将 Nginx、MySQL、Spring Boot

   

  每一类

  ，作为数组中的一个元素。

  - 这里，我们配置读取一个 Spring Boot 应用的日志。

- ```
  output.elasticsearch
  ```

   

  配置项，设置 Filebeat 直接写入数据到 Elasticsearch 中。虽然说 Filebeat

   

  ```
  5.0
  ```

   

  版本以来，也提供了

   

  Filter

   

  功能，但是相比 Logstash 提供的 Filter 会弱一些。所以在一般情况下，Filebeat 并不直接写入到 Elasticsearch 中。

  - 这里，我们注释掉该配置项，设置 Filebeat 不写入到 Elasticsearch 中。

- ```
  output.logstash
  ```

   

  配置项，设置 Filebeat 写入数据到 Logstash 中。

  - 这里，我们配置写入到[「3. Logstash 搭建」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)中。

### 4.3 启动

执行 `nohup ./filebeat &` 命令，后台启动 Filebeat 服务。

可以通过 `logs/filebeat` 日志，查看启动是否成功。如果没有 `ERROR` 级别的日志，说明启动成功。下面是艿艿截取的部分日志：



```
# 加载成功一个 input
2020-01-01T16:03:03.686+0800    INFO    crawler/crawler.go:72   Loading Inputs: 1
2020-01-01T16:03:03.687+0800    INFO    log/input.go:152        Configured paths: [/Users/yunai/logs/spring.log]
2020-01-01T16:03:03.687+0800    INFO    input/input.go:114      Starting input of type: log; ID: 7378641906887219931
2020-01-01T16:03:03.687+0800    INFO    crawler/crawler.go:106  Loading and starting Inputs completed. Enabled inputs: 1

# 所有配置加载完成
2020-01-01T16:03:03.687+0800    INFO    cfgfile/reload.go:171   Config reloader started
2020-01-01T16:03:03.687+0800    INFO    cfgfile/reload.go:226   Loading of config files completed.
```



可以通过 `data/registry/filebeat/data.json` 数据，查看每个日文件的采集进度。例如说：



```
[{"source":"/Users/yunai/logs/spring.log","offset":1721,"timestamp":"2020-01-01T16:03:03.687331+08:00","ttl":-1,"type":"log","meta":null,"FileStateOS":{"inode":102842596,"device":16777220}}]
```



- 这里 `offset` 就是采集日志文件的位置。

我们使用 [Elasticsearch Head](https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm) ，查看到日志文件的日志，已经存储到 Elasticsearch 中。如下图所示：![Elasticsearch Head 查看日志数据](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/05.png)

## 5. Kibana 搭建

> FROM <https://www.elastic.co/cn/products/kibana>
>
> 通过 Kibana，您可以对自己的 Elasticsearch 进行可视化。
>
> ![简单直观地构建可视化](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/dbfc86d62222dad6c6cb58bacdf7b0c7)

### 5.1 下载

打开 [Kibana 下载页面](https://www.elastic.co/cn/downloads/kibana)，选择想要的 Kibana 版本。这里，我们选择 `7.5.1` 最新版本。



```
# 创建目录
$ mkdir -p /Users/yunai/Kibana
$ cd /Users/yunai/Kibana

# 下载
$ wget https://artifacts.elastic.co/downloads/kibana/kibana-7.5.1-darwin-x86_64.tar.gz

# 解压
$ tar -zxvf kibana-7.5.1-darwin-x86_64.tar.gz
$ cd kibana-7.5.1-darwin-x86_64

# 查看目录
$ ls- ls
  32 -rw-r--r--@    1 yunai  staff    13675 Dec 17 07:46 LICENSE.txt
2840 -rw-r--r--@    1 yunai  staff  1453580 Dec 17 07:46 NOTICE.txt
   8 -rw-r--r--@    1 yunai  staff     4048 Dec 17 07:46 README.txt
   0 drwxr-xr-x@    5 yunai  staff      160 Jan  1 23:52 bin # 执行文件
   0 drwxr-xr-x@    5 yunai  staff      160 Jan  1 23:52 built_assets
   0 drwxr-xr-x@    3 yunai  staff       96 Jan  1 23:52 config # 配置文件
   0 drwxr-xr-x@    2 yunai  staff       64 Dec 17 07:46 data
   0 drwxr-xr-x@    9 yunai  staff      288 Jan  1 23:52 node
   0 drwxr-xr-x@ 1207 yunai  staff    38624 Jan  1 23:52 node_modules
   0 drwxr-xr-x@    4 yunai  staff      128 Jan  1 23:52 optimize
   8 -rw-r--r--@    1 yunai  staff      738 Dec 17 07:46 package.json
   0 drwxr-xr-x@    2 yunai  staff       64 Dec 17 07:46 plugins
   0 drwxr-xr-x@   11 yunai  staff      352 Dec 17 07:46 src
   0 drwxr-xr-x@   16 yunai  staff      512 Jan  1 23:52 webpackShims
   0 drwxr-xr-x@    9 yunai  staff      288 Jan  1 23:52 x-pack
```



因为 Kibana 软件包比较大，所以下载可能比较久。

### 5.2 配置文件

使用 `vi config/kibana.yml` 命令，编辑 Kibana 配置文件。主要修改如下行：



```
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
kibana.index: ".kibana"
```



- 如上四个配置项，实际配置文件里都已经有了，只需要将 `#` 注释去掉即可。

### 5.3 启动

执行 `nohup bin/kibana &` 命令，后台启动 Kibana 服务。

可以通过 `nohup.out` 日志，查看启动是否成功。

同时，我们可以使用浏览器，访问 <http://127.0.0.1:5601/> 地址，进一步判断 Kibana 是否启动成功。界面如下：![Kibana 界面](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/02.png)

### 5.4 简单使用

下面，我们使用 Kibana 查询刚[「4. Beatas 搭建」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)收集的日志文件的日志。

**第一步**

① 使用浏览器，访问 <http://127.0.0.1:5601/> 地址，选择左边「Management」菜单，进入「Management」界面。如下图所示：![Management 界面](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/06.png)

② 点击「Kibana Index Patterns」按钮，进入「Index patterns」界面。如下图所示：![Index patterns 界面](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/07.png)

③ 点击右上角的「Create index pattern」按钮，进入「Create index pattern —— Define index pattern」界面。如下图所示：![Create Index patterns 界面](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/08.png)

④ 在「Index pattern」输入框，输入 `filebeat-7.5.1-*` 索引，再点击右边的「Next step」按钮，进入「Create index pattern —— Configure settings」界面。如下图所示：![Create Index patterns 界面](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/09.png)

⑤ 在「Time Filter field name」下拉框，选择 `@timestamp` 选项，再点击右边的「Create index pattern」按钮，创建 Kibana Index Patterns **完成**。如下图所示：![ Index Patterns](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/10.png)

**第二步**

① 使用浏览器，访问 <http://127.0.0.1:5601/> 地址，选择左边「Discover」菜单，进入「Discover」界面。如下图所示：![Discover 界面](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/11.png)

② 在「UptimeIndexPattern」下拉框，选择 `filebeat-7.5.1-*` 选项。在「时间范围」下拉框，选择近 15 天。如下图所示：![Discover 界面 2](ELK(Elasticsearch + Logstash + Kibana) 极简入门.assets/12.png)

至此，我们完成了使用 Kibana 对[「4. Beatas 搭建」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)收集的日志文件的日志的查询。美滋滋~

## 6. 消息队列的集成

随着 Beats 收集的每秒数据量越来越大，Logstash 可能无法承载这么大量日志的处理。虽然说，可以增加 Logstash 节点数量，提高每秒数据的处理速度，但是仍需考虑可能 Elasticsearch 无法承载这么大量的日志的写入。此时，我们可以考虑引入消息队列，进行缓存：

- Beats 收集数据，写入数据到消息队列中。
- Logstash 从消息队列中，读取数据，写入 Elasticsearch 中。

因此，整体的架构就变成了：

> [《logstash 和 filebeat 是什么关系？》](https://www.zhihu.com/question/54058964)
>
> [引入消息队列](https://pic2.zhimg.com/80/v2-3b578fddb1221bcdb250fcfdf3b070a2_hd.jpg)

一般情况下，我们会考虑采用 Kafka 作为消息队列。这里，艿艿推荐阅读 [《ELK + Filebeat + Kafka 分布式日志管理平台搭建》](http://www.iocoder.cn/Fight/ELK-Filebeat-Kafka-distributed-log-management-platform/?self)文章，**重点关注 FileBeat 写入 Kafka 的 output 配置，和 Logstash 读取 Kafka 的 input 配置**。

## 7. 告警

针对收集的日志，我们可以实现监控告警。例如说：

- 监控 Nginx 日志，当返回状态码 5xx 超过一定阀值时，发送告警。
- 监控 Spring Boot 应用的日志，当出现错误日志时，发送告警。

能够实现 ELK 日志服务的方式比较多，我们来逐个看看。

**1、ElastAlert**

[ElastAlert](https://github.com/Yelp/elastalert)，基于查询 Elasticsearch 的数据，进行告警。

具体使用，参考[《ELK 实现日志监控告警》](http://www.iocoder.cn/Fight/ELK-implements-log-monitoring-alarm/?self)文章。

**2、Sentinl**

[Sentinl](https://github.com/sirensolutions/sentinl/) 构建在 Kibana 之上的可视化告警工具，内部实现也是基于查询 Elasticsearch 的数据，进行告警。

具体使用，参考[《基于 ELK 实现日志告警》](http://www.iocoder.cn/Fight/Implement-log-alarm-based-on-ELK/?self)文章。

不过现在貌似基本暂停维护了，不太推荐。

**3、Grafana**

[Grafana](https://github.com/grafana/grafana) ，是支持包括对 Elasticsearch 在内的多种数据源的可视化的工具。同时，它提供告警功能。因此，也能实现基于查询 Elasticsearch 的数据，进行告警。

目前暂时没有找到合适的文章，胖友可以参考[《芋道 Prometheus + Grafana + Alertmanager 极简入门》](http://www.iocoder.cn/Prometheus/install/?self)的[「8. Grafana Alerting」](https://www.iocoder.cn/Elasticsearch/ELK-install/?self#)小节，只是数据源从 Prometheus 换成 Elasticsearch。

**4、X-Pack**

[Elastic NV](https://en.wikipedia.org/wiki/Elastic_NV) 提供的 X-Pack 套件，提供了基于查询 Elasticsearch 的数据，进行[告警](https://www.elastic.co/cn/what-is/elasticsearch-alerting/)。

具体使用，参考[《es-xpack-watch-user》](http://www.iocoder.cn/Fight/es-xpack-watch-user/?self)文章。

**5、Logstash + 脚本**

不同于上述基于查询 Elasticsearch 的数据，进行告警的方案。我们可以在 Logstash 获得到异常日志时，执行 Shell 脚本，发送告警。

具体实现，参考[《ELK 日志监控平台告警升级(邮件 + 钉钉)》](http://www.iocoder.cn/Fight/ELK-log-monitoring-platform-alarm-upgrade/?self)

不过个人不是很推荐这个方案，还是让 Logstash 专注做采集的事情，监控告警还是基于 Elasticsearch 上的数据。

## 8. Spring Boot 使用示例

在 [《芋道 Spring Boot 日志服务 ELK 入门》](http://www.iocoder.cn/Spring-Boot/ELK/?self) 中，我们来详细学习如何在 Spring Boot 中，整合并使用 ELK 收集日志。

## 666. 参考

本文更多的是以入门为主，更多配置示例，可以参考 <https://github.com/ameizi/ELK> 项目。其提供了收集 Tomcat、Nginx、MySQL 等等日志收集的示例。

想要进一步深入的胖友，也可以阅读如下资料：

- [《集中式日志系统 ELK 协议栈详解》](http://www.iocoder.cn/Fight/Centralized-logging-system-ELK-stack-details/?self)

  > ELK + Filebeat 架构。

- [《搭建 ELK 实时日志平台并在 Spring Boot 和 Nginx 项目中使用》](http://www.iocoder.cn/Fight/build-elk-and-use-it-for-springboot-and-nginx/?self)

  > ELK + Redis 架构。

- [《ELK 平台介绍》](http://www.iocoder.cn/Fight/ELK-platform-introduction/?self)

  > ELK 架构，并且 Spring Boot 应用使用 [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder) 直接发送给 Logstash。

- [《快速搭建 ELK 日志分析系统》](http://www.iocoder.cn/Fight/Set-up-ELK-log-analysis-system-in-minutes/?self)

  > ELK + Redis 架构，并且提供较多实战示例。

- [《ELK、FILEBEAT 日志分析平台搭建》](http://www.iocoder.cn/Fight/ELK-FILEBEAT-log-analysis-platform/?self)

  > ELK + Filebeat 架构，并提供 Nginx 访问日志和错误日志的示例。