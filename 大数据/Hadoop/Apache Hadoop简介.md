# Apache Hadoop简介

## 1. 简介

随着世界通过社交媒体平台、消息应用程序和音频/视频流服务比以往任何时候都更加紧密地联系在一起，人类每天都会产生惊人的数据量。

因此，`统计数据`显示，所有数据中有 90% 是在过去两到三年内生成的。

由于数据在现代世界中的作用变得更具战略性，并且大多数数据都是非结构化格式，因此我们需要一个能够处理如此庞大的数据集的框架——其规模为 PB、ZB 或更大，通常称为`大数据`。

在本教程中，我们将探索`Apache Hadoop`，这是一种被广泛认可的处理大数据的技术，它具有可靠性、可扩展性和高效的分布式计算能力。



## 2.什么是 Apache Hadoop

Apache Hadoop 是一个开源框架，旨在从单个服务器扩展到多台机器，每台机器提供本地计算和存储，促进分布式计算环境中大规模数据集的存储和处理。

该框架利用集群计算的强大功能——在集群中的多个服务器上以小块形式处理数据。

此外，Hadoop 与许多其他工具协同工作的灵活性使其成为现代大数据平台的基础——为企业提供了一种可靠且可扩展的方式，可以从不断增长的数据中获取洞察力。



## 3. Hadoop的核心组件

Hadoop的四个核心组件构成了其系统的基础，实现了分布式数据存储和处理。



### 3.1. Hadoop Common

它是一组所有 Hadoop 模块普遍可用的必备 Java 库，可使其更好地发挥作用。



### 3.2. Hadoop分布式文件系统（HDFS）

HDFS 是一种分布式文件系统，原生支持大型数据集，可提供高可用性和容错能力的数据吞吐量。

简单来说，**它是 Hadoop 的存储组件，可以在多台机器上存储大量数据**，并且能够在标准硬件上运行，从而具有成本效益。



### 3.3. Hadoop YARN

YARN 是 Yet Another Resource Negotiator 的缩写，它提供了一个用于调度作业的框架，并为分布式系统管理系统资源。

简而言之，它是 Hadoop 的资源管理组件，用于管理处理存储在 HDFS 中的数据所使用的资源。



### 3.4. Hadoop MapReduce

一种[简单的编程模型](https://www.baeldung.com/cs/mapreduce-algorithm)，通过将非结构化数据转换为键值对（映射）来并行处理数据，然后将其拆分到节点之间并将结果组合到最终输出中（减少）。

因此，用外行人的话来说，它是 Hadoop 的大脑，**在两个阶段提供主要的处理引擎功能——mapping and reducing.**



## 4. Setup

GNU/Linux平台 [支持 Hadoop](https://www.baeldung.com/linux/gnu-project)。因此，让我们在 Linux 操作系统上设置我们的 Hadoop 集群。



### 4.1. 先决条件

首先，我们需要为最新的 Apache Hadoop安装 Java 8/11。另外，我们可以按照此处的Java 版本建议进行操作。

接下来，我们需要安装 SSH 并确保sshd服务正在运行以使用 Hadoop 脚本。



### 4.2. 下载并安装 Hadoop

我们可以按照详细指南[在 Linux 中安装和配置 Hadoop](https://www.baeldung.com/linux/hadoop-install-configure)。

安装完成后，让我们运行以下命令来验证已安装的 Hadoop 的版本：

```powershell
hadoop version
```

这是上述命令的输出：

```powershell
Hadoop 3.4.0 Source code repository git@github.com:apache/hadoop.git -r bd8b77f398f626bb7791783192ee7a5dfaeec760 
Compiled by root on 2024-03-04T06:29Z 
Compiled on platform linux-aarch_64 
Compiled with protoc 3.21.12 
From source with checksum f7fe694a3613358b38812ae9c31114e 
This command was run using /usr/local/hadoop/common/hadoop-common-3.4.0.jar
```

值得注意的是，我们可以在三种受支持的操作模式下运行它——独立、伪分布式和完全分布式。然而，默认情况下，Hadoop 配置为以独立（本地非分布式）模式运行，本质上运行单个 Java 进程。



## 5.基本操作

一旦我们的 Hadoop 集群启动并运行，我们就可以执行许多操作。



### 5.1. HDFS 操作

**让我们来看看使用 HDFS 的命令行界面管理文件和目录的一些便捷操作**。

例如，我们可以将文件上传到 HDFS：

```powershell
hdfs dfs -put /local_file_path /hdfs_path
```

类似地，我们可以下载文件：

```powershell
hdfs dfs -get /hdfs_file_path /local_path
```

让我们列出 HDFS 目录中的所有文件：

```powershell
hdfs dfs -ls /hdfs_directory_path
```

以下是我们如何读取 HDFS 位置的文件内容：

```powershell
hdfs dfs -cat /hdfs_file_path
```

此外，此命令还会检查 HDFS 磁盘使用情况：

```powershell
hdfs dfs -du -h /hdfs_path
```

此外，还有其他有用的命令，如*-mkdir*用于创建目录、-rm*用于*删除文件或目录以及*-mv*用于移动以重命名可通过 HDFS 获得的文件。





### 5.2. 运行 MapReduce 作业

*Hadoop 发行版在hadoop-mapreduce-examples-3.4.0* jar 文件下包含一些简单的介绍性示例，用于探索 MapReduce 。

例如，让我们看一下*WordCount*，这是一个简单的应用程序，它扫描给定的输入文件并提取每个单词出现的次数作为输出。

首先，我们将在*输入*目录中创建一些文本文件 - *textfile1.txt*和*textfile2.txt*，其中包含一些内容：

```powershell
echo "Introduction to Apache Hadoop" > textfile01.txt
echo "Running MapReduce Job" > textfile01.txt
```

然后，让我们运行 MapReduce 作业并创建输出文件：

```powershell
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.0.jar wordcount input output
```

输出日志展示了其他任务中的映射和归约操作：

```powershell
2024-09-22 12:54:39,592 INFO impl.MetricsSystemImpl: Scheduled Metric snapshot period at 10 second(s).
2024-09-22 12:54:39,592 INFO impl.MetricsSystemImpl: JobTracker metrics system started
2024-09-22 12:54:39,722 INFO input.FileInputFormat: Total input files to process : 2
2024-09-22 12:54:39,752 INFO mapreduce.JobSubmitter: number of splits:2
2024-09-22 12:54:39,835 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_local1009338515_0001
2024-09-22 12:54:39,835 INFO mapreduce.JobSubmitter: Executing with tokens: []
2024-09-22 12:54:39,917 INFO mapreduce.Job: The url to track the job: http://localhost:8080/
2024-09-22 12:54:39,918 INFO mapreduce.Job: Running job: job_local1009338515_0001
2024-09-22 12:54:39,959 INFO mapred.MapTask: Processing split: file:/Users/anshulbansal/work/github_examples/hadoop/textfile01.txt:0+30
2024-09-22 12:54:39,984 INFO mapred.MapTask: (EQUATOR) 0 kvi 26214396(104857584)
2024-09-22 12:54:39,984 INFO mapred.MapTask: mapreduce.task.io.sort.mb: 100
2024-09-22 12:54:39,984 INFO mapred.MapTask: soft limit at 83886080
2024-09-22 12:54:39,984 INFO mapred.MapTask: bufstart = 0; bufvoid = 104857600
2024-09-22 12:54:39,984 INFO mapred.MapTask: kvstart = 26214396; length = 6553600
2024-09-22 12:54:39,985 INFO mapred.MapTask: Map output collector class = org.apache.hadoop.mapred.MapTask$MapOutputBuffer
2024-09-22 12:54:39,998 INFO mapred.LocalJobRunner: 
2024-09-22 12:54:39,999 INFO mapred.MapTask: Starting flush of map output
2024-09-22 12:54:39,999 INFO mapred.MapTask: Spilling map output
2024-09-22 12:54:39,999 INFO mapred.MapTask: bufstart = 0; bufend = 46; bufvoid = 104857600
2024-09-22 12:54:39,999 INFO mapred.MapTask: kvstart = 26214396(104857584); kvend = 26214384(104857536); length = 13/6553600
2024-09-22 12:54:40,112 INFO mapred.LocalJobRunner: Finishing task: attempt_local1009338515_0001_r_000000_0
2024-09-22 12:54:40,112 INFO mapred.LocalJobRunner: reduce task executor complete.
2024-09-22 12:54:40,926 INFO mapreduce.Job: Job job_local1009338515_0001 running in uber mode : false
2024-09-22 12:54:40,928 INFO mapreduce.Job:  map 100% reduce 100%
2024-09-22 12:54:40,929 INFO mapreduce.Job: Job job_local1009338515_0001 completed successfully
2024-09-22 12:54:40,936 INFO mapreduce.Job: Counters: 30
	File System Counters
		FILE: Number of bytes read=846793
		FILE: Number of bytes written=3029614
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
	Map-Reduce Framework
		Map input records=2
		Map output records=7
		Map output bytes=80
		Map output materialized bytes=106
		Input split bytes=264
		Combine input records=7
		Combine output records=7
		Reduce input groups=7
		Reduce shuffle bytes=106
		Reduce input records=7
		Reduce output records=7
		Spilled Records=14
		Shuffled Maps =2
		Failed Shuffles=0
		Merged Map outputs=2
		GC time elapsed (ms)=4
		Total committed heap usage (bytes)=663748608
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=52
	File Output Format Counters 
		Bytes Written=78
```

一旦 MapReduce 作业结束，将在*输出*目录中创建*part-r-00000*文本文件，其中包含单词及其出现次数：*
*

```powershell
hadoop dfs -cat /output/part-r-00000
```

以下是该文件的内容：

```powershell
Apache	1
Hadoop	1
Introduction	1
Job	1
MapReduce	1
Running	1
to	1
```

类似地，我们可以查看其他示例，如用于执行大规模数据排序的TerraSort、用于生成对基准测试和测试有用的随机文本数据的RandomTextWriter以及用于在hadoop-mapreduce-examples-3.4.0 jar 文件中提供的输入文件中搜索匹配字符串的Grep。



### 5.3. 管理 YARN 上的服务

现在，**我们来看一下管理 Hadoop 服务的一些操作**。例如，下面是检查节点状态的命令：

```powershell
yarn node -list
```

类似地，我们可以列出所有正在运行的应用程序：

```powershell
yarn application -list
```

以下是我们部署服务的方法：

```powershell
yarn deploy service service_definition.json
```

同样，我们可以启动一个已经注册的服务：

```powershell
yarn app -launch service_name
```

然后，我们可以分别启动、停止和销毁服务：

```powershell
yarn app -start service_name
yarn app -stop service_name
yarn app -destroy service_name
```

此外，还有一些有用的 YARN 管理命令，如用于检查守护进程日志的 daemonlog 、用于启动节点管理器的 nodemanager 、用于启动 Web 代理服务器的proxyserver和用于启动时间线服务器的timelineserver 。



## 6. Hadoop生态系统

**Hadoop 存储和处理大型数据集，需要支持工具来执行诸如提取、分析和数据提取等任务**。让我们列出 Hadoop 生态系统中的一些重要工具：

- Apache Ambari：基于 Web 的集群管理工具，可简化 Hadoop 集群的部署、管理和监控
- [Apache HBase](https://www.baeldung.com/hbase)：面向实时应用程序的分布式、列式数据库
- Apache Hive：一个数据仓库基础设施，允许类似 SQL 的查询来管理和分析大型数据集
- Apache Pig：一种高级脚本语言，可轻松实现数据分析任务
- [Apache Spark](https://www.baeldung.com/apache-spark)：用于批处理、流式传输和机器学习的集群计算框架
- [Apache ZooKeeper](https://www.baeldung.com/java-zookeeper)：为大规模分布式系统提供可靠、高性能、分布式协调服务的服务