# Introduction to the Java NIO Selector



## **1. Overview**



In this article, we'll explore the introductory parts of Java NIO's *Selector* component.

A selector provides a mechanism for monitoring one or more NIO channels and recognizing when one or more become available for data transfer.

This way, **a single thread can be used for managing multiple channels**, and thus multiple network connections.

## **2. Why Use a Selector?**



With a selector, we can use one thread instead of several to manage multiple channels. **Context-switching between threads is expensive for the operating system**, and additionally, **each thread takes up memory.**

Therefore, the fewer threads we use, the better. However, it's important to remember that **modern operating systems and CPU's keep getting better at multitasking**, so the overheads of multi-threading keep diminishing over time.



Here, we'll be dealing with how we can handle multiple channels with a single thread using a selector.

Note also that selectors don't just help you read data; they can also listen for incoming network connections and write data across slow channels.

## **3. Setup**

To use the selector, we do not need any special set up. All the classes we need are in the core *java.nio* package and we just have to import what we need.

After that, we can register multiple channels with a selector object. When I/O activity happens on any of the channels, the selector notifies us. This is how we can read from a large number of data sources on a single thread.

Any channel we register with a selector must be a sub-class of *SelectableChannel*. These are a special type of channels that can be put in non-blocking mode.



## **4. Creating a Selector**

A selector may be created by invoking the static *open* method of the *Selector* class, which will use the system's default selector provider to create a new selector:

```java
Selector selector = Selector.open();
```



## **5. Registering Selectable Channels**

In order for a selector to monitor any channels, we must register these channels with the selector. We do this by invoking the *register* method of the selectable channel.

But before a channel is registered with a selector, it must be in non-blocking mode:

```java
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

This means that we cannot use *FileChannel*s with a selector since they cannot be switched into non-blocking mode the way we do with socket channels.

The first parameter is the *Selector* object we created earlier, the second parameter defines an interest set**,** meaning what events we are interested in listening for in the monitored channel, via the selector.



There are four different events we can listen for, each is represented by a constant in the *SelectionKey* class:

- *Connect* **–** when a client attempts to connect to the server. Represented by *SelectionKey.OP_CONNECT*
- *Accept* **–** when the server accepts a connection from a client. Represented by *SelectionKey.OP_ACCEPT*
- *Read* **–** when the server is ready to read from the channel. Represented by *SelectionKey.OP_READ*
- *Write* **–** when the server is ready to write to the channel. Represented by *SelectionKey.OP_WRITE*

The returned object *SelectionKey* represents the selectable channel's registration with the selector. We'll look at it further in the following section.



## **6. The SelectionKey Object**

As we saw in the previous section, when we register a channel with a selector, we get a *SelectionKey* object. This object holds data representing the registration of the channel.

It contains some important properties which we must understand well to be able to use the selector on the channel. We'll look at these properties in the following subsections.



### **6.1. The Interest Set**

An interest set defines the set of events that we want the selector to watch out for on this channel. It is an integer value; we can get this information in the following way.

First, we have the interest set returned by the *SelectionKey*‘s *interestOps* method. Then we have the event constant in *SelectionKey* we looked at earlier.

When we AND these two values, we get a boolean value that tells us whether the event is being watched for or not:

```java
int interestSet = selectionKey.interestOps();

boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE;
```



### **6.2. The Ready Set**

The ready set defines the set of events that the channel is ready for. It is an integer value as well; we can get this information in the following way.



We've got the ready set returned by *SelectionKey*‘s *readyOps* method. When we AND this value with the events constants as we did in the case of interest set, we get a boolean representing whether the channel is ready for a particular value or not.

Another alternative and shorter way to do this is to use *SelectionKey'*s convenience methods for this same purpose:

```java
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWriteable();Copy
```



### **6.3. The Channel**

Accessing the channel being watched from the *SelectionKey* object is very simple. We just call the *channel* method:

```java
Channel channel = key.channel();
```



### 6.4. The Selector**

Just like getting a channel, it's very easy to obtain the *Selector* object from the *SelectionKey* object:

```java
Selector selector = key.selector();
```

### **6.5. Attaching Objects**



We can attach an object to a *SelectionKey.* Sometimes we may want to give a channel a custom ID or attach any kind of Java object we may want to keep track of.

Attaching objects is a handy way of doing it. Here is how you attach and get objects from a *SelectionKey*:

```java
key.attach(Object);

Object object = key.attachment();
```

Alternatively, we can choose to attach an object during channel registration. We add it as a third parameter to channel's *register* method, like so:

```java
SelectionKey key = channel.register(
selector, SelectionKey.OP_ACCEPT, object);
```

## **7. Channel Key Selection**



So far, we have looked at how to create a selector, register channels to it and inspect the properties of the *SelectionKey* object which represents a channel's registration to a selector.



This is only half of the process, now we have to perform a continuous process of selecting the ready set which we looked at earlier. We do selection using selector's *select* method, like so:

```java
int channels = selector.select();
```

This method blocks until at least one channel is ready for an operation. The integer returned represents the number of keys whose channels are ready for an operation.

Next, we usually retrieve the set of selected keys for processing:

```java
Set<SelectionKey> selectedKeys = selector.selectedKeys();
```

The set we have obtained is of *SelectionKey* objects, each key represents a registered channel which is ready for an operation.

After this, we usually iterate over this set and for each key, we obtain the channel and perform any of the operations that appear in our interest set on it.

During the lifetime of a channel, it may be selected several times as its key appears in the ready set for different events. This is why we must have a continuous loop to capture and process channel events as and when they occur.



## **8. Complete Example**

To cement the knowledge we have gained in the previous sections, we're going to build a complete client-server example.

For ease of testing out our code, we'll build an echo server and an echo client. In this kind of setup, the client connects to the server and starts sending messages to it. The server echoes back messages sent by each client.

When the server encounters a specific message, such as *end*, it interprets it as the end of the communication and closes the connection with the client.



### **8.1. The Server**

Here is our code for *EchoServer.java*:

```java
public class EchoServer {

    private static final String POISON_PILL = "POISON_PILL";

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel serverSocket = ServerSocketChannel.open();
        serverSocket.bind(new InetSocketAddress("localhost", 5454));
        serverSocket.configureBlocking(false);
        serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        ByteBuffer buffer = ByteBuffer.allocate(256);

        while (true) {
            selector.select();
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iter = selectedKeys.iterator();
            while (iter.hasNext()) {

                SelectionKey key = iter.next();

                if (key.isAcceptable()) {
                    register(selector, serverSocket);
                }

                if (key.isReadable()) {
                    answerWithEcho(buffer, key);
                }
                iter.remove();
            }
        }
    }

    private static void answerWithEcho(ByteBuffer buffer, SelectionKey key)
      throws IOException {
 
        SocketChannel client = (SocketChannel) key.channel();
        int r = client.read(buffer);
        if (r == -1 || new String(buffer.array()).trim().equals(POISON_PILL)) {
            client.close();
            System.out.println("Not accepting client messages anymore");
        }
        else {
            buffer.flip();
            client.write(buffer);
            buffer.clear();
        }
    }

    private static void register(Selector selector, ServerSocketChannel serverSocket)
      throws IOException {
 
        SocketChannel client = serverSocket.accept();
        client.configureBlocking(false);
        client.register(selector, SelectionKey.OP_READ);
    }

    public static Process start() throws IOException, InterruptedException {
        String javaHome = System.getProperty("java.home");
        String javaBin = javaHome + File.separator + "bin" + File.separator + "java";
        String classpath = System.getProperty("java.class.path");
        String className = EchoServer.class.getCanonicalName();

        ProcessBuilder builder = new ProcessBuilder(javaBin, "-cp", classpath, className);

        return builder.start();
    }
} 
```

This is what is happening; we create a *Selector* object by calling the static *open* method. We then create a channel also by calling its static *open* method, specifically a *ServerSocketChannel* instance.

This is because **ServerSocketChannel is selectable and good for a stream-oriented listening socket**.

We then bind it to a port of our choice. Remember we said earlier that before registering a selectable channel to a selector, we must first set it to non-blocking mode. So next we do this and then register the channel to the selector.

We don't need the *SelectionKey* instance of this channel at this stage, so we will not remember it.

Java NIO uses a buffer-oriented model other than a stream-oriented model. So socket communication usually takes place by writing to and reading from a buffer.

We, therefore, create a new *ByteBuffer* which the server will be writing to and reading from. We initialize it to 256 bytes, it's just an arbitrary value, depending on how much data we plan to transfer to and fro.



Finally, we perform the selection process. We select the ready channels, retrieve their selection keys, iterate over the keys, and perform the operations for which each channel is ready.

We do this in an infinite loop since servers usually need to keep running whether there is an activity or not.

The only operation a *ServerSocketChannel* can handle is an *ACCEPT* operation. When we accept the connection from a client, we obtain a *SocketChannel* object on which we can do read and writes. We set it to non-blocking mode and register it for a READ operation to the selector.

During one of the subsequent selections, this new channel will become read-ready. We retrieve it and read it contents into the buffer. True to it's as an echo server, we must write this content back to the client.

**When we desire to write to a buffer from which we have been reading, we must call the flip() method**.

We finally set the buffer to write mode by calling the *flip* method and simply write to it.

The *start()* method is defined so that the echo server can be started as a separate process during unit testing.



### **8.2. The Client**

Here is our code for *EchoClient.java*:

```java
public class EchoClient {
    private static SocketChannel client;
    private static ByteBuffer buffer;
    private static EchoClient instance;

    public static EchoClient start() {
        if (instance == null)
            instance = new EchoClient();

        return instance;
    }

    public static void stop() throws IOException {
        client.close();
        buffer = null;
    }

    private EchoClient() {
        try {
            client = SocketChannel.open(new InetSocketAddress("localhost", 5454));
            buffer = ByteBuffer.allocate(256);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public String sendMessage(String msg) {
        buffer = ByteBuffer.wrap(msg.getBytes());
        String response = null;
        try {
            client.write(buffer);
            buffer.clear();
            client.read(buffer);
            response = new String(buffer.array()).trim();
            System.out.println("response=" + response);
            buffer.clear();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return response;

    }
} 
```

The client is simpler than the server.

We use a singleton pattern to instantiate it inside the *start* static method. We call the private constructor from this method.

In the private constructor, we open a connection on the same port on which the server channel was bound and still on the same host.

We then create a buffer to which we can write and from which we can read.

Finally, we have a *sendMessage* method which reads wraps any string we pass to it into a byte buffer which is transmitted over the channel to the server.

We then read from the client channel to get the message sent by the server. We return this as the echo of our message.



### **8.3. Testing**

Inside a class called *EchoTest.java*, we are going to create a test case that starts the server, sends messages to the server, and only passes when the same messages are received back from the server. As a final step, the test case stops the server before completion.

We can now run the test:



```java
public class EchoTest {

    Process server;
    EchoClient client;

    @Before
    public void setup() throws IOException, InterruptedException {
        server = EchoServer.start();
        client = EchoClient.start();
    }

    @Test
    public void givenServerClient_whenServerEchosMessage_thenCorrect() {
        String resp1 = client.sendMessage("hello");
        String resp2 = client.sendMessage("world");
        assertEquals("hello", resp1);
        assertEquals("world", resp2);
    }

    @After
    public void teardown() throws IOException {
        server.destroy();
        EchoClient.stop();
    }
} 
```



## 9. *Selector.wakeup()*

As we saw earlier, calling *selector.select()* blocks the current thread until one of the watched channels becomes operation-ready. We can override this by calling *selector.wakeup()* from another thread.

The result is that **the blocking thread returns immediately rather than continuing to wait, whether a channel has become ready or not**.

We can demonstrate this using a *CountDownLatch* and tracking the code execution steps:

```java
@Test
public void whenWakeUpCalledOnSelector_thenBlockedThreadReturns() {
    Pipe pipe = Pipe.open();
    Selector selector = Selector.open();
    SelectableChannel channel = pipe.source();
    channel.configureBlocking(false);
    channel.register(selector, OP_READ);

    List<String> invocationStepsTracker = Collections.synchronizedList(new ArrayList<>());

    CountDownLatch latch = new CountDownLatch(1);

    new Thread(() -> {
        invocationStepsTracker.add(">> Count down");
        latch.countDown();
        try {
            invocationStepsTracker.add(">> Start select");
            selector.select();
            invocationStepsTracker.add(">> End select");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }).start();

    invocationStepsTracker.add(">> Start await");
    latch.await();
    invocationStepsTracker.add(">> End await");

    invocationStepsTracker.add(">> Wakeup thread");
    selector.wakeup();
    //clean up
    channel.close();

    assertThat(invocationStepsTracker)
      .containsExactly(
        ">> Start await",
        ">> Count down",
        ">> Start select",
        ">> End await",
        ">> Wakeup thread",
        ">> End select"
    );
} 
```

In this example, we use Java NIO's *Pipe* class to open a channel for testing purposes. We track code execution steps in a thread-safe list. By analyzing these steps, we can see how *selector.wakeup()* releases the thread blocked by *selector.select()*.

## **10. Conclusion**

In this article, we have covered the basic usage of the Java NIO Selector component.

The complete source code and all code snippets for this article are available in my [GitHub project](https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-nio-2).

