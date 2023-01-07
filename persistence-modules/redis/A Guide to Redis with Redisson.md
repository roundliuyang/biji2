# A Guide to Redis with Redisson



## **1. Overview**



**Redisson is a Redis client for Java**. In this article, we'll explore some of its features, and demonstrate how it could facilitate building distributed business applications.

**Redisson constitutes an in-memory data grid** that offers distributed Java objects and services backed by Redis**.** Its distributed in-memory data model allows sharing of domain objects and services across applications and servers.

In this article we'll see hot to setup Redisson, understand how it operates, and explore some of Redisson's objects and services.



## **2. Maven Dependencies** 



Let's get started by importing *Redisson* to our project by adding the section below to our *pom.xml:*

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.13.1</version>
</dependency>
```

The latest version of this dependency can be found [here](https://search.maven.org/classic/#search|ga|1|g%3A"org.redisson" AND a%3A"redisson").





## **3. Configuration** 

Before we get started, we must ensure we have the latest version of Redis setup and running. If you don't have Redis and you use Linux or Macintosh, you can follow the information [here](http://redis.io/topics/quickstart) to get it setup. If you're a Windows user, you can setup Redis using this unofficial [port](https://github.com/MSOpenTech/redis).

We need to configure Redisson to connect to Redis. Redisson supports connections to the following Redis configurations:

- Single node
- Master with slave nodes
- Sentinel nodes
- Clustered nodes
- Replicated nodes

**Redisson supports Amazon Web Services (AWS) ElastiCache Cluster and Azure Redis Cache** for Clustered and Replicated Nodes.

Let's connect to a single node instance of Redis. This instance is running locally on the default port, 6379:

```
RedissonClient client = Redisson.create();
```

You can pass different configurations to the *Redisson* object's *create* method. This could be configurations to have it connect to a different port, or maybe, to connect to a Redis cluster. This **configuration could be in Java code or loaded from an external configuration file**.





### **3.1. Java Configuration**

Let's configure Redisson in Java code:

```java
Config config = new Config();
config.useSingleServer()
  .setAddress("redis://127.0.0.1:6379");

RedissonClient client = Redisson.create(config);
```

We **specify Redisson configurations in an instance of a Config object** and then pass it to the *create* method. Above, we specified to Redisson that we want to connect to a single node instance of Redis. To do this we used the *Config* object's *useSingleServer* method. This returns a reference to a *SingleServerConfig* object.

The *SingleServerConfig* object has settings that Redisson uses to connect to a single node instance of Redis. Here, we use its *setAddress* method to configure the *address* setting. This sets the address of the node we're connecting to. Some other settings include *retryAttempts*, *connectionTimeout* and *clientName*. These settings are configured using their corresponding setter methods.

We can configure Redisson for different Redis configurations in a similar way using the *Config* object's following methods:

- *useSingleServer* – for single node instance. Get single node settings [here](https://github.com/redisson/redisson/wiki/2.-Configuration#261-single-instance-settings)
- *useMasterSlaveServers* – for master with slave nodes. Get master-slave node settings [here](https://github.com/redisson/redisson/wiki/2.-Configuration#281-master-slave-settings)
- *useSentinelServers* – for sentinel nodes. Get sentinel node settings [here](https://github.com/redisson/redisson/wiki/2.-Configuration#271-sentinel-settings)
- *useClusterServers* – for clustered nodes. Get clustered node settings [here](https://github.com/redisson/redisson/wiki/2.-Configuration#241-cluster-settings)
- *useReplicatedServers* – for replicated nodes. Get replicated node settings [here](https://github.com/redisson/redisson/wiki/2.-Configuration#251-replicated-settings)



### **3.2. File Configuration**

**Redisson can load configurations from external JSON or YAML** files:

```java
Config config = Config.fromJSON(new File("singleNodeConfig.json"));  
RedissonClient client = Redisson.create(config);
```

The *Config* object's *fromJSON* method can load configurations from a string, file, input stream or URL.

Here is the sample configuration in the *singleNodeConfig.json* file:

```json
{
    "singleServerConfig": {
        "idleConnectionTimeout": 10000,
        "connectTimeout": 10000,
        "timeout": 3000,
        "retryAttempts": 3,
        "retryInterval": 1500,
        "password": null,
        "subscriptionsPerConnection": 5,
        "clientName": null,
        "address": "redis://127.0.0.1:6379",
        "subscriptionConnectionMinimumIdleSize": 1,
        "subscriptionConnectionPoolSize": 50,
        "connectionMinimumIdleSize": 10,
        "connectionPoolSize": 64,
        "database": 0,
        "dnsMonitoringInterval": 5000
    },
    "threads": 0,
    "nettyThreads": 0,
    "codec": null
}
```

Here is a corresponding YAML configuration file:

```yaml
singleServerConfig:
    idleConnectionTimeout: 10000
    connectTimeout: 10000
    timeout: 3000
    retryAttempts: 3
    retryInterval: 1500
    password: null
    subscriptionsPerConnection: 5
    clientName: null
    address: "redis://127.0.0.1:6379"
    subscriptionConnectionMinimumIdleSize: 1
    subscriptionConnectionPoolSize: 50
    connectionMinimumIdleSize: 10
    connectionPoolSize: 64
    database: 0
    dnsMonitoringInterval: 5000
threads: 0
nettyThreads: 0
codec: !<org.redisson.codec.JsonJacksonCodec> {}

```

We can configure other Redis configurations from a file in a similar manner using settings peculiar to that configuration. For your reference, here are their JSON and YAML file formats:

- Single node – [format](https://github.com/redisson/redisson/wiki/2.-Configuration#262-single-instance-json-and-yaml-config-format)
- Master with slave nodes – [format](https://github.com/redisson/redisson/wiki/2.-Configuration#282-master-slave-json-and-yaml-config-format)
- Sentinel nodes – [format](https://github.com/redisson/redisson/wiki/2.-Configuration#272-sentinel-json-and-yaml-config-format)
- Clustered nodes – [format](https://github.com/redisson/redisson/wiki/2.-Configuration#242-cluster-json-and-yaml-config-format)
- Replicated nodes – [format](https://github.com/redisson/redisson/wiki/2.-Configuration#252-replicated-json-and-yaml-config-format)

To save a Java configuration to JSON or YAML format, we can use the *toJSON* or *toYAML* methods of the *Config* object:

```hava
Config config = new Config();
// ... we configure multiple settings here in Java
String jsonFormat = config.toJSON();
String yamlFormat = config.toYAML();
```

Now that we know how to configure Redisson, let's look at how Redisson executes operations.



## **4. Operation**



**Redisson supports synchronous, asynchronous and reactive interfaces**. Operations over these **interfaces are thread-safe**.

All entities (objects, collections, locks and services) generated by a *RedissonClient* have synchronous and asynchronous methods. **Synchronous methods bear asynchronous variants**. These methods normally bear the same method name of their synchronous variants appended with “Async”. Let's look at a synchronous method of the *RAtomicLong* object:

```java
RedissonClient client = Redisson.create();
RAtomicLong myLong = client.getAtomicLong('myLong');
```

The asynchronous variant of the synchronous *compareAndSet* method would be:

```java
RFuture<Boolean> isSet = myLong.compareAndSetAsync(6, 27);
```

The asynchronous variant of the method returns an *RFuture* object. We can set listeners on this object to get back the result when it becomes available:

```java
isSet.handle((result, exception) -> {
    // handle the result or exception here.
});
```

To generate reactive objects, we would need to use the *RedissonReactiveClient*:

```java
RedissonReactiveClient client = Redisson.createReactive();
RAtomicLongReactive myLong = client.getAtomicLong("myLong");

Publisher<Boolean> isSetPublisher = myLong.compareAndSet(5, 28);
```

This method returns reactive objects based on the [Reactive Streams](http://www.reactive-streams.org/) Standard for Java 9.

Let's explore some of the distributed objects provided by Redisson.



## **5. Objects**

An individual instance of a **Redisson object is serialized and stored in any of the available Redis nodes backing Redisson**. These objects could be distributed in a cluster across multiple nodes and can be accessed by a single application or multiple applications/servers.

These distributed objects follow specifications from the *java.util.concurrent.atomic package.* **They support lock-free, thread-safe and atomic operations on objects stored in Redis**. Data consistency between applications/servers is ensured as values are not updated while another application is reading the object.

Redisson objects are bound to Redis keys. We can manage these keys through the *RKeys* interface. And then, we access our Redisson objects using these keys.

There are several options we may use to get the Redis keys.

We can simple get all the keys:

```java
RKeys keys = client.getKeys();
```

Alternatively, we can extract only the names:

```java
Iterable<String> allKeys = keys.getKeys();
```

And finally, we're able to get the keys conforming to a pattern:

````java
Iterable<String> keysByPattern = keys.getKeysByPattern('key*')
````

The RKeys interface also allows deleting keys, deleting keys by pattern and other useful key-based operations that we could use to manage our keys and objects.

Distributed objects provided by Redisson include:

- **ObjectHolder**
- ***BinaryStreamHolder***
- ***GeospatialHolder***
- ***BitSet***
- ***AtomicLong***
- ***AtomicDouble***
- ***Topic***
- ***BloomFilter***
- ***HyperLogLog***

Let's take a look at three of these objects: *ObjectHolder, AtomicLong,* and *Topic.*



### **5.1. Object Holder**

Represented by the *RBucket* class, this object can hold any type of object. This object has a maximum size of 512MB:

```java
RBucket<Ledger> bucket = client.getBucket("ledger");
bucket.set(new Ledger());
Ledger ledger = bucket.get();
```

The *RBucket* object can perform atomic operations such as *compareAndSet and* *getAndSet* on objects it holds.



### **5.2. AtomicLong**

Represented by the *RAtomicLong* class, this object closely resembles the *java.util.concurrent.atomic.AtomicLong* class and represents a *long* value that can be updated atomically:

```java
RAtomicLong atomicLong = client.getAtomicLong("myAtomicLong");
atomicLong.set(5);
atomicLong.incrementAndGet();
```



### 5.3. Topic**

The *Topic* object supports the Redis' “publish and subscribe” mechanism. To listen for published messages:

```java
RTopic subscribeTopic = client.getTopic("baeldung");
subscribeTopic.addListener(CustomMessage.class,
  (channel, customMessage) -> future.complete(customMessage.getMessage()));
```

Above, the *Topic* is registered to listen to messages from the “baeldung” channel. We then add a listener to the topic to handle incoming messages from that channel. We can add multiple listeners to a channel.

Let's publish messages to the “baeldung” channel:

```java
RTopic publishTopic = client.getTopic("baeldung");
long clientsReceivedMessage
  = publishTopic.publish(new CustomMessage("This is a message"));
```

This could be published from another application or server. The *CustomMessage* object will be received by the listener and processed as defined in the *onMessage* method.

We can learn more about other Redisson objects [here](https://github.com/redisson/redisson/wiki/6.-distributed-objects).



## **6. Collections**

We handle Redisson collections in the same fashion we handle objects.



Distributed collections provided by Redisson include:

- ***Map***
- ***Multimap***
- ***Set***
- ***SortedSet***
- ***ScoredSortedSet***
- **LexSortedSet**
- ***List***
- ***Queue***
- ***Deque***
- ***BlockingQueue***
- ***BoundedBlockingQueue***
- ***BlockingDeque***
- ***BlockingFairQueue***
- ***DelayedQueue***
- ***PriorityQueue***
- ***PriorityDeque***

Let's take a look at three of these collections: *Map, Set,* and *List.*



### **6.1. Map**

Redisson based maps implement the *java.util.concurrent.ConcurrentMap* and *java.util.Map* interfaces. Redisson has four map implementations. These are *RMap*, *RMapCache*, *RLocalCachedMap* and *RClusteredMap*.

Let's create a map with Redisson:

```java
RMap<String, Ledger> map = client.getMap("ledger");
Ledger newLedger = map.put("123", new Ledger());map
```

*RMapCache* supports map entry eviction. *RLocalCachedMap* allows local caching of map entries*. RClusteredMap* allows data from a single map to be split across Redis cluster master nodes.

We can learn more about Redisson maps [here](https://github.com/redisson/redisson/wiki/7.-distributed-collections#71-map).



### **6.2. Set**

Redisson based *Set* implements the *java.util.Set* interface.

Redisson has three *Set* implementations, *RSet*, *RSetCache*, and *RClusteredSet* with similar functionality as their map counterparts.

Let's create a *Set* with Redisson:

```java
RSet<Ledger> ledgerSet = client.getSet("ledgerSet");
ledgerSet.add(new Ledger());
```

We can learn more about Redisson sets [here](https://github.com/redisson/redisson/wiki/7.-distributed-collections#71-map).



### **6.3. List**

Redisson-based *Lists* implement the *java.util.List* interface.

Let's create a *List* with Redisson:

````java
RList<Ledger> ledgerList = client.getList("ledgerList");
ledgerList.add(new Ledger());
````

We can learn more about other Redisson collections [here](https://github.com/redisson/redisson/wiki/7.-distributed-collections).



## **7. Locks and Synchronizers** 

Redisson's **distributed locks allow for thread synchronization** across applications/servers. Redisson's list of locks and synchronizers include:

- ***Lock***
- ***FairLock***
- ***MultiLock***
- ***ReadWriteLock***
- ***Semaphore***
- ***PermitExpirableSemaphore***
- ***CountDownLatch***

Let's take a look at *Lock* and *MultiLock.*



### **7.1. Lock**

Redisson's *Lock* implements *java.util.concurrent.locks.Lock* interface.

Let's implement a lock, represented by the *RLock* class:

```java
RLock lock = client.getLock("lock");
lock.lock();
// perform some long operations...
lock.unlock();
```



### **7.2. MultiLock**

Redisson's *RedissonMultiLock* groups multiple *RLock* objects and treats them as a single lock:

```java
RLock lock1 = clientInstance1.getLock("lock1");
RLock lock2 = clientInstance2.getLock("lock2");
RLock lock3 = clientInstance3.getLock("lock3");

RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
lock.lock();
// perform long running operation...
lock.unlock();
```

We can learn more about other locks [here](https://github.com/redisson/redisson/wiki/8.-distributed-locks-and-synchronizers).



## **8. Services**



Redisson exposes 4 types of distributed services. These are: **Remote Service**, **Live Object Service**, **Executor Service** and **Scheduled Executor Service**. Let's look at the Remote Service and Live Object Service.



### **8.1. Remote Service** 

This service provides **Java remote method invocation facilitated by Redis**. A Redisson remote service consists of a server-side (worker instance) and client-side implementation. The server-side implementation executes a remote method invoked by the client. Calls from a remote service can be synchronous or asynchronous.

The server-side registers an interface for remote invocation:

```java
RRemoteService remoteService = client.getRemoteService();
LedgerServiceImpl ledgerServiceImpl = new LedgerServiceImpl();

remoteService.register(LedgerServiceInterface.class, ledgerServiceImpl);
```

The client-side calls a method of the registered remote interface:

```java
RRemoteService remoteService = client.getRemoteService();
LedgerServiceInterface ledgerService
  = remoteService.get(LedgerServiceInterface.class);

List<String> entries = ledgerService.getEntries(10);
```

We can learn more about remote services [here](https://github.com/redisson/redisson/wiki/9.-distributed-services#91-remote-service).



### **8.2. Live Object Service**

Redisson Live Objects extend the concept of standard Java objects that could only be accessed from a single JVM to **enhanced Java objects that could be shared between different JVMs in different machines**. This is accomplished by mapping an object's fields to a Redis hash. This mapping is made through a runtime-constructed proxy class. Field getters and setters are mapped to Redis hget/hset commands.

Redisson Live Objects support atomic field access as a result of Redis' single-threaded nature.

Creating a Live Object is simple:

```java
@REntity
public class LedgerLiveObject {
    @RId
    private String name;

    // getters and setters...
}
```

We annotate our class with *@REntity* and a unique or identifying field with *@RId*. Once we have done this, we can use our Live Object in our application:

```java
RLiveObjectService service = client.getLiveObjectService();

LedgerLiveObject ledger = new LedgerLiveObject();
ledger.setName("ledger1");

ledger = service.persist(ledger);
```

We create our Live Object like standard Java objects using the *new* keyword. We then use an instance of *RLiveObjectService* to save the object to Redis using its *persist* method.

If the object has previously been persisted to Redis, we can retrieve the object:

```java
LedgerLiveObject returnLedger
  = service.get(LedgerLiveObject.class, "ledger1");
```

We use the *RLiveObjectService* to get our Live Object using the field annotated with *@RId*.

[Here](https://github.com/redisson/redisson/wiki/9.-distributed-services#92-live-object-service) we can find more about Redisson Live Objects, and other Redisson services are described [here.](https://github.com/redisson/redisson/wiki/9.-distributed-services)



## **9. Pipelining**

Redisson supports pipelining. **Multiple operations can be batched as a single atomic operation**. This is facilitated by the *RBatch* class. Multiple commands are aggregated against an *RBatch* object instance before they are executed:

```java
RBatch batch = client.createBatch();
batch.getMap("ledgerMap").fastPutAsync("1", "2");
batch.getMap("ledgerMap").putAsync("2", "5");

BatchResult<?> batchResult = batch.execute();
```



## **10. Scripting**

Redisson supports LUA scripting. **We can execute LUA scripts against Redis**:

```java
client.getBucket("foo").set("bar");
String result = client.getScript().eval(Mode.READ_ONLY,
  "return redis.call('get', 'foo')", RScript.ReturnType.VALUE);
```



## **11. Low-Level Client** 

It's possible that we might want to perform Redis operations not yet supported by Redisson. **Redisson provides a low-level client that allows execution of native Redis commands**:

```java
RedisClientConfig redisClientConfig = new RedisClientConfig();
redisClientConfig.setAddress("localhost", 6379);

RedisClient client = RedisClient.create(redisClientConfig);

RedisConnection conn = client.connect();
conn.sync(StringCodec.INSTANCE, RedisCommands.SET, "test", 0);

conn.closeAsync();
client.shutdown();
```

The low-level client also supports asynchronous operations.



## **12. Conclusion**

This article showcased Redisson and some of the features that make it ideal for developing distributed applications. We explored its distributed objects, collections, locks and services. We also explored some of its other features such as pipelining, scripting and its low-level client.

**Redisson also provides integration with other frameworks** such as the JCache API, Spring Cache, Hibernate Cache and Spring Sessions. We can learn more about its integration with other frameworks [here](https://github.com/redisson/redisson/wiki/14.-Integration-with-frameworks).

You can find code samples in the [GitHub project](https://github.com/eugenp/tutorials/tree/master/persistence-modules/redis).



