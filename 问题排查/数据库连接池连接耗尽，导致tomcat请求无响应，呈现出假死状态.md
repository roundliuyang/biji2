# 数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态



## 前言

> 最近，测试部门的同事找到我，说他们测试时，没一会就发现服务接口请求一直无响应，Tomcat跟死掉了一样，也没有返回任何的错误响应，说让我赶紧排查下；听完后，我瞬间激灵了下，妹的，最近老是出问题，领导都要给我开批评大会了。哈哈，开玩笑的，像我这么英俊的人，领导怎么会忍心批评我呢，哼，我把这个问题马上解决掉，都不会让领导知道的！
>
> 
>
> 简单说下程序部署情况：tomcat + oracle
>
> 排查时，可以使用命令进行排查，也可以使用可视化监控工具；例如使用使用JDK自带的 jvisualvm.exe 监控工具。





## 命令排查过程：



### 1、请求服务无响应，首先看看tomcat是否是真的挂掉了：

命令： ps -ef | grep tomcat

通过上面的命令查看tomcat运行着；执行结果如下：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/42f8b9063528447ab453097de30ae40e_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

**注意：** 如果此服务器中运行着多个tomcat，需要查看下图中画框的地方运行的tomcat地址是否正确；

通过命令查看发现，tomcat正常运行着，那么这就是处于假死状态，下面接着排查。





### 2、查看http请求是否到达了tomcat：

通过查看 tomcat 的 logs 目录下的 localhost_access_log  日志文件中 请求记录；

命令： tail -100f  localhost_access_log

通过上面的命令查看实时的日志，执行完上面的查看日志的命令后，然后再次请求下程序，在日志中并没有发现请求记录，说明tomcat处于一种假死状态，下面接着排查。



### 3、查看tomcat的JVM的GC情况：

查看GC情况，是否由于频繁的GC，长时间的GC，导致程序出现长时间的卡顿，最终导致请求来不及处理，进入队列中进行等待，调用方长时间得不到响应，造成tomcat假死状态；

命令：jstat  -gc  pid   time  count

例如： **jstat  -gc  71129   1000  5 **    监控 71129 这个进程JVM的GC情况，每隔1000ms 输出一次，共输出5次；

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/bcddc74b2c824c89a7a6a5ab46154cb8_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

命令执行结果参数解析：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/ad0eff38ba60428b9f02823807a2c69f_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

通过上面命令查看GC情况，发现垃圾回收也不频繁，并且进行GC的时间也不长，应该不是GC的原因。



### 4、查看tomcat的JVM的堆情况：

查看堆内存的情况，是否存在堆内存溢出 导致tomcat假死，无法为新请求分配堆内存资源；

命令 ： jmap -heap pid

例子： jmap -heap 71129 **71129是正在运行tomcat的进程号** ；

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/66d90439cff740c687f3dc0024c9606d_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

通过命令执行结果得知，堆内存中可使用内存还很大，不会出现内存溢出的问题，所以也不是堆内存过小导致的tomcat假死。



### 5、查看tomcat的 JVM线程情况：

①、使用 jstack 命令导出当前JVM的线程dump快照，然后看看dump中线程都在干什么？

命令：jstack   pid   >>   jvmThreadDump.log

例子：jstack  71129  >>   jvmThreadDump.log

> 生成 71129 进程的 JVM的线程快照，并将快照内容重定向到 jvmThreadDump.log 文件中；
>
> 注意：生成的  jvmThreadDump.log 在你当前执行命令的目录下。

②、接着使用命令 more  查看 jvmThreadDump.log  内容；

命令 ： more  jvmThreadDump.log

如果的dump文件太大的话，需要使用more 命令一点点看；执行完more 命令的话，再按 enter 回车键 一点点展示文件内容；

③、通过查看线程快照文件，发现很多线程的状态是  **WAITING** 等待状态；

并且使用命令查看线程状态为  **WAITING**  的线程占总线程的比例：

**注意：tomcatDump.log  为生成的线程快照文件名称，记得改为自己设置的名称** ；

```shell
count=`cat tomcatDump.log | grep java.lang.Thread.State | wc -l`; wait=`cat tomcatDump.log | grep WAITING | wc -l`;  a=`echo | awk "{print $wait/$count*100}"`; echo "$a%"
```

执行命令，得到结果 ： **91.9786% ** ，发现九成多的线程处于等待状态；

至此，找到了tomcat假死的原因，但是还需进一步确定 **什么原因导致的大量线程一直等待？**



在一个普通的Spring Boot 项目中，线程处于等待状态的比例会受到多种因素的影响，包括但不限于以下几点：

1. **业务逻辑复杂度**：如果项目中存在大量需要等待外部资源响应或者依赖其他服务的业务逻辑，那么线程处于等待状态的比例可能会较高。
2. **并发访问量**：项目所承载的并发请求量越高，那么线程等待外部资源返回的情况也会相应增加。
3. **IO操作**：如果项目中存在大量的IO操作（如数据库查询、文件读写、网络请求等），那么线程可能会因为等待IO操作完成而进入等待状态。
4. **线程池配置**：线程池的大小和配置也会影响线程的等待情况。如果线程池过小，可能会导致线程被阻塞在等待队列中。
5. **外部服务响应时间**：如果项目依赖的外部服务响应较慢，那么线程等待的时间也会相应增加。

由于以上因素的复杂性和多样性，无法提供一个准确的百分比来描述线程处于等待状态的比例。不同项目的情况会有所不同。通常来说，在一个典型的高并发的Web应用中，一部分线程会处于等待状态，但具体比例需要通过监控和分析来确定。可以通过使用监控工具（如Spring Boot Actuator、Prometheus等）来实时监控线程的状态和各种指标，以便进行优化和调整。

>通过查看调用的服务接口代码得知，此接口业务逻辑中自己没设置任何的锁，所以应该不是自己写的代码的问题，但是此接口中涉及到了很多 JDBC数据库操作，那是不是数据库连接池中的连接不够用了呢？因为数据库连接属于竞争资源，如果连接池中的连接已经耗尽了，那么接下来的进行 JDBC的线程就需要进行wait 等待连接。



### 6、查看与数据库建立的TCP连接情况：

在上面发现，大量线程处于等待状态，而通过分析得知，可能是由于数据库连接池中的连接耗尽导致的，所以可以通过命令查看下，部署服务代码的服务器与数据库所在服务器建立的TCP连接数是否已经达到了配置的数据库连接池的最大连接数；

命令：netstat  -pan | grep 1521 | wc -l

因为本文中使用的数据库是Oracle，所以 grep  搜索匹配的端口号是 1521；

如果是mysql数据库则将端口号改为3306 即可， netstat  -pan | grep  3306 | wc -l   ；

如果设置了自定义的数据库端口号，则改为自定义的端口号即可；

通过命令查询到 已经使用的数据库的连接数为  **6**  个，那接着看下设置的数据库连接池最大连接数；

数据源配置如下：

```xml
<Resource name="jdbc/testdemo"
      type="javax.sql.DataSource"
      factory="com.alibaba.druid.pool.DruidDataSourceFactory"
      url="jdbc:oracle:thin:@192.168.3.125:1521:ora11g"
      driverClassName="oracle.jdbc.driver.OracleDriver"
      username="root"
      password="root"
      auth="Container"
	  initialSize="2"
      maxActive="6"
      minIdle="3"
      maxWait="30000"
      timeBetweenEvictionRunsMillis="30000"
      minEvictableIdleTimeMillis="600000"
	  maxEvictableIdleTimeMillis="900000"
	  poolPreparedStatements="true"
	  maxOpenPreparedStatements="20"
	  validationQuery="select 1 from dual"
	  testOnBorrow="false"
	  testOnReturn="false"
	  testWhileIdle="true"
	  filters="wall,stat,log4j2"
      />
```

通过查看数据源发现，连接池配置的最大连接数是  **maxActive="6"**  ；发现目前程序中使用的连接数已达到最大值，那么后面再进行 JDBC 操作的线程将进入  **等待状态** ，等待获取连接；

> 至此，tomcat假死的排查过程就结束了，并且原因也找到了，就是数据库连接池中的连接耗尽了；所以，在后面测试中，需要在数据源中将最大连接数设置的大一些，并且也再进一步查看下代码，看看是否存在数据库连接使用完后没有进行关闭的问题。
>
> 除了数据库连接池连接耗尽会导致tomcat假死外，还有一些其它的情况也会导致发生，例如： redis 连接池连接耗尽，或者是redis连接使用完不释放，最终导致redis连接耗尽。

除了使用上面的命令进行问题排查外，也可以直接使用可视化监控工具进行排查，更加简便、直观。



## 可视化监控工具排查

>使用 JDK 自带的 jvisualvm.exe 工具进行 **JMX远程** 可视化监控tomcat；
>
>jvisualvm.exe 位于 $JAVA_HOME/bin 目录下；



### 1、使用JMX实现远程监控步骤：

>下面使用 JMX实现远程监控的内容参考自：[jvisualvm远程监控tomcat](https://link.juejin.cn/?target=https%3A%2F%2Fwww.cnblogs.com%2Fleocook%2Fp%2Fjvisualvmandtomcat.html)

①、在 Tomcat 的 bin 目录下的 startup.sh 文件中的 **倒数第一行上** 加上如下内容：

```
export CATALINA_OPTS="$CATALINA_OPTS
-Dcom.sun.management.jmxremote
-Djava.rmi.server.hostname=192.168.1.130
-Dcom.sun.management.jmxremote.port=7003
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false"
```

**注意：如果直接复制上面的内容，然后放到 startup.sh 文件中的话，可能由于存在很多空格导致tomcat启动失败；请复制下面的内容，然后进行修改：**

```shell
export CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote -Djava.rmi.server.hostname=192.168.1.130 -Dcom.sun.management.jmxremote.port=7003 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
```

上面内容参数解析：

```sql
-Dcom.sun.management.jmxremote 启用JMX远程监控
-Djava.rmi.server.hostname=192.168.1.130  这是连接你的tomcat所在的服务器地址
-Dcom.sun.management.jmxremote.port=7003  jmx连接端口
-Dcom.sun.management.jmxremote.ssl=false  是否ssl加密
-Dcom.sun.management.jmxremote.authenticate=false  远程连接需要密码认证
```

在 startup.sh 文件中添加上上面的内容后，需要将tomcat重启下才会生效；

②、将 jvisualvm.exe 打开，界面如下：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/590d7dd955ef4543b1b137e990e952fb_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

③、在远程上右击，添加主机，输入服务器的ip：就是在 startup.sh 文件中添加内容中的hostname

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/37cd985fa3244ecdb405ff1044ad5ad9_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

④、在远程主机上右击，添加 **JMX连接** ，手动在ip地址后面加上设置的jmx连接端口7003，然后点击确定即可：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/4372aa3463ce44e4a686465957245c5c_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

⑤、通过上面的步骤，就已经完成了远程监控连接了，然后自己双击就能进行监控界面了：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/f153a0d1a044461a80f58cdbb1afc919_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)



### 2、查看监视内容：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/1a3daf7d110a4462a02eaa617e22164e_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

通过查看监视画面得知，CPU、GC、堆Heap的情况都没有问题，那接着查看下线程的情况：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/1a3daf7d110a4462a02eaa617e22164e_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0-1710569941338.awebp)

点击上面图片中的 **线程Dump** 按钮，生成线程的快照，快照文件内容部分如下：

![img](数据库连接池连接耗尽，导致tomcat请求无响应，呈现出假死状态.assets/e9abdfe9de234ed58c5890e9a4ad64d9_tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.awebp)

通过查看快照文件内容发现，很多线程的状态的都是 **WAITING** 等待状态；

> 接下来的分析排查过程就如上面的 **命令排查过程** 一样了。



## 总结：

上面的两种排查方式，本人比较推荐还是使用第一种 **命令排查** ，因为很多的情况是不会让你修改配置文件进行远程监控的，即便使用监控工具看起来更加直观、简便；所以，平时需要记一些常用的排查命令，以备不时之需。