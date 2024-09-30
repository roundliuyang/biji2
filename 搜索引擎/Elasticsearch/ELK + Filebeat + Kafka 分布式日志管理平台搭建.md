# ELK + Filebeat + Kafka 分布式日志管理平台搭建



## 一、先描述一下使用这种框架搭建平台的工作流程

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/0d6eb90f2bc201df17592e7fabed36b3)



## 二、对上面的工作流程进行简单描述

（1）将filebeat部署到需要采集日志的服务器上，filebeat将采集到的日志数据传输到kafka中。

（2）kafka将获取到的日志信息存储起来，并且作为输入(input)传输给logstash。

（3）logstash将kafka中的数据作为输入，并且把kafka中的数据进行过滤等其他操作，然后把操作后得到的数据输入（output）到es(elasticsearch)中。

（4）es(基于lucene搜索引擎）对logstash中的数据进行处理，并且将数据作为输入传送给kibna进行显示。



## 三、部署该平台需要的软件

本次部署使用的软件及版本如下：

(1) elasticsearch-6.1.3.tar.gz

（2）filebeat-6.1.3-linux-x86_64.tar.gz

（3）kibana-6.1.3-linux-x86_64.tar.gz

（4）logstash-6.2.3.tar.gz

（5）kafka_2.12-1.0.0.tgz

以上软件可以到elasticsearch官网下载需要的软件：

https://www.elastic.co/cn/products



## 四、安装以及配置各软件

（1）elasticsearch安装配置

 1.首先解压elasticsearch-6.1.3.tar.gz

 tar -zxvf elasticsearch-6.1.3.tar.gz

 2.修改config/elasticsearch.yml配置文件

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/a94b3ee06b3cca25539e630c3f31accb)

 将配置中的network.host修改为本机的ip地址，配置配置自己的端口号

 3.启动elasticsearch(启动该软件不能使用root用户，需要普通用户，可以新建普通用户，将 目录的权限都赋予给该新用户)

 nohup bin/elasticsearch &

 验证是否启动成功：

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/9a1975d4d918adcddec75586cf294748)

（2）kibana安装配置：

 vim config/kibana.yml

 具体配置如下（根据自己的ip以及端口情况进行配置）

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/cbd14d9cd1fc1263ee627e3e1b100fc1)

 启动kibana： nohup bin/kibana &

 在浏览器中验证是否启动成功:

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/5ca741417beb5cc1ea15e49c01ffd91a)



(3)kafka安装与配置：

请参考 [《芋道 Kafka 极简入门》](http://www.iocoder.cn/Kafka/install/?self)

（4）logstash安装与配置:

在logstash安装软件中新建test.conf配置文件。

配置如下，该配置中没有加过滤器filter

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/5d629cd8db5aefbaf6a8622f83f88886)

上述配置说明如下:

topics后面的test和logstash-tomcat表示从kafka中topic为test与logstash-tomcat的主题中获取数据，此处的配置根据自己的具体情况去配置。

bootstrap_servers表示配置kafka的ip与端口。

output配置中的hosts表示elasticsearch的ip和端口好，index的配置是用于后面在kibana中配置index使用。

启动logstash：nohup bin/logstash -f test.conf &

(5)filebeat安装与配置：

修改filebeat的配置文件filebeat.yml

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/83080bb25880c959496ac04662a07491)

paths后的值表示从/home/elk/log/access.log中获取数据，tags表示日志标签，在后面的kibna中查看数据时可以找到该tag标签，并且可以根据该tag标签查找过滤查找数据。

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/b7da38c2801a5747c946b921cbc615a8)

添加kafka输出的配置，将elasticsearch输出配置注释掉。hosts表示kafka的ip和端口号，topic表示filebeat将数据输出到topic为test的主题下，此处也根据自己情况修改。

启动filebeat： nohup ./filebeat -c filebeat.yml &

(6)在kibana操作：

以上平台都搭建好以后在kibana上创建index索引，该index索引和logstash配置中的output中的index对应。

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/d9f747d627dbe37b78ac2452ae98a32e)

该index名称要和logstash配置中的index正则匹配，否则新增不了。

验证搭建的平台，日志数据是否能正常获取及显示。在filebeat配置文件中paths对应的目录下新增access.log文件，并且添加数据进去，然后在kibana上查看数据是否正常显示。

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/87cafb273c12308f9a7b7f73f3dc8a2d)

点开任何一条数据可以查看详细情况：

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/85233df8a679cd90fa27cd0eda950305)

截图中的tags即表示filebeat中配置文件中的tags配置，可以将tags作为filter过滤条件进行查询。

![img](ELK + Filebeat + Kafka 分布式日志管理平台搭建.assets/79a730b5d8f1ab54062550ba59bd0878)

在分布式系统中，可以将某一类服务器日志归为一个tag，这样在查询日志时可以减小查询的区域。

说明：

1.在部署的过程中可能会遇到各种情况，此时根据日志说明都可以百度处理（如部署的过程中不能分配内存的问题）。

2.如果完成后如果数据显示不了，可以先到根据工作流程到各个节点查询数据是否存储和传输成功。如查询filebeat是否成功把数据传输到了kafka，可以使用kafka中如下命令查询：

bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning

查看日志filebeat中的数据是否正常在kafka中存储。

在filebeat配置的日志中手动添加日志： echo "test" > access.log，可以在kafka中动态的显示出该信息

3.该平台的搭建是比较简便的方式，大家可以更加灵活以及动态的配置该平台。

















