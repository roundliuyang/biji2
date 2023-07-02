# Intro to AspectJ





## **1. Introduction**



This article is a **quick and practical introduction to AspectJ.**

First, we'll show how to enable aspect-oriented programming, and then we'll focus on the difference between compile-time, post-compile, and load-time weaving.

Let's start with a short introduction of aspect-oriented programming (AOP) and AspectJ's basics.

## **2. Overview**



AOP is a programming paradigm that aims to increase modularity by allowing the separation of cross-cutting concerns. It does so by adding additional behavior to existing code without modification of the code itself. Instead, we declare separately which code is to modify.

[AspectJ](https://eclipse.org/aspectj/) implements both concerns and the weaving of crosscutting concerns using extensions of Java programming language.



## **3. Maven Dependencies**



AspectJ offers different libraries depending on its usage. We can find Maven dependencies under group [org.aspectj](https://search.maven.org/classic/#search|ga|1|g%3A"org.aspectj") in the Maven Central repository.

In this article, we focus on dependencies needed to create aspects and Weaver using the compile-time, post-compile, and load-time Weavers.

### **3.1. AspectJ Runtime**



When running an AspectJ program, the classpath should contain the classes and aspects along with the AspectJ runtime library *aspectjrt.jar**:*

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjrt</artifactId>
    <version>1.8.9</version>
</dependency>
```

This dependency is available over on [Maven Central](https://search.maven.org/classic/#search|ga|1|g%3A"org.aspectj" AND a%3A"aspectjrt").

### **3.2. AspectJWeaver**



Besides the AspectJ runtime dependency, we will also need to include the *aspectjweaver.jar* to introduce advice to Java class at load time:



```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.8.9</version>
</dependency>
```

The dependency is also available on [Maven Central](https://search.maven.org/classic/#artifactdetails|org.aspectj|aspectjweaver|1.8.9|jar).

## **4. Aspect Creation**



AspectJ provides an implementation of AOP and has **three core concepts:**

- Join Point
- Pointcut
- Advice

We'll demonstrate these concepts by creating a simple program to validate a user account balance.

First, let's create an *Account* class with a given balance and a method to withdraw:

```java
public class Account {
    int balance = 20;

    public boolean withdraw(int amount) {
        if (balance < amount) {
            return false;
        } 
        balance = balance - amount;
        return true;
    }
}
```

We'll create an *AccountAspect.aj* file to log account information and to validate the account balance (note that AspectJ files end with a “*.aj*” file extension):

```java
public aspect AccountAspect {
    final int MIN_BALANCE = 10;

    pointcut callWithDraw(int amount, Account acc) : 
     call(boolean Account.withdraw(int)) && args(amount) && target(acc);

    before(int amount, Account acc) : callWithDraw(amount, acc) {
    }

    boolean around(int amount, Account acc) : 
      callWithDraw(amount, acc) {
        if (acc.balance < amount) {
            return false;
        }
        return proceed(amount, acc);
    }

    after(int amount, Account balance) : callWithDraw(amount, balance) {
    }
}
```

As we can see, we've added a *pointcut* to the withdraw method and created three *advises* that refer to the defined *pointcut*.

In order to understand the following, we introduce the following definitions:

- **Aspect**: A modularization of a concern that cuts across multiple objects. Each aspect focuses on a specific crosscutting functionality
- **Join point**: A point during the execution of a script, such as the execution of a method or property access
- **Advice**: Action taken by an aspect at a particular join point
- **Pointcut**: A regular expression that matches join points. An advice is associated with a pointcut expression and runs at any join point that matches the pointcut

For more details on these concepts and their specific semantics, we may want to check out the following [link](https://eclipse.org/aspectj/doc/next/progguide/semantics-pointcuts.html).

Next, we need to weave the aspects into our code. The sections below address three different types of weaving: compile-time weaving, post-compile weaving, and load-time weaving in AspectJ.



## **5. Compile-Time Weaving**

The simplest approach of weaving is compile-time weaving. When we have both the source code of the aspect and the code that we are using aspects in, the AspectJ compiler will compile from source and produce a woven class files as output. Afterward, upon execution of your code, the weaving process output class is loaded into JVM as a normal Java class.

We can download the [AspectJ Development Tools](https://eclipse.org/aspectj/downloads.php#ides) since it includes a bundled AspectJ compiler. One of AJDT's most important features is a tool for the visualization of crosscutting concerns, which is helpful for debugging a pointcut specification. We may visualize the combined effect even before the code is deployed.

We use Mojo's AspectJ Maven Plugin to weave AspectJ aspects into our classes using the AspectJ compiler.

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.14.0</version>
    <configuration>
        <complianceLevel>1.8</complianceLevel>
        <source>1.8</source>
        <target>1.8</target>
        <showWeaveInfo>true</showWeaveInfo>
        <verbose>true</verbose>
        <Xlint>ignore</Xlint>
        <encoding>UTF-8 </encoding>
    </configuration>
    <executions>
        <execution>
            <goals>
                <!-- use this goal to weave all your main classes -->
                <goal>compile</goal>
                <!-- use this goal to weave all your test classes -->
                <goal>test-compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

For more details on option reference of the AspectJ compiler, we may want to check out the following [link](http://www.mojohaus.org/aspectj-maven-plugin/ajc_reference/standard_opts.html).

Let's add some test cases for our Account class:

```java
public class AccountTest {
    private Account account;

    @Before
    public void before() {
        account = new Account();
    }

    @Test
    public void given20AndMin10_whenWithdraw5_thenSuccess() {
        assertTrue(account.withdraw(5));
    }

    @Test
    public void given20AndMin10_whenWithdraw100_thenFail() {
        assertFalse(account.withdraw(100));
    }
}
```

When we run the test cases, the below text that is shown in the console means that we successfully weaved the source code:

```xml
[INFO] Join point 'method-call
(boolean com.baeldung.aspectj.Account.withdraw(int))' in Type
'com.baeldung.aspectj.test.AccountTest' (AccountTest.java:20)
advised by around advice from 'com.baeldung.aspectj.AccountAspect'
(AccountAspect.class:18(from AccountAspect.aj))

[INFO] Join point 'method-call
(boolean com.baeldung.aspectj.Account.withdraw(int))' in Type 
'com.baeldung.aspectj.test.AccountTest' (AccountTest.java:20) 
advised by before advice from 'com.baeldung.aspectj.AccountAspect' 
(AccountAspect.class:13(from AccountAspect.aj))

[INFO] Join point 'method-call
(boolean com.baeldung.aspectj.Account.withdraw(int))' in Type 
'com.baeldung.aspectj.test.AccountTest' (AccountTest.java:20) 
advised by after advice from 'com.baeldung.aspectj.AccountAspect'
(AccountAspect.class:26(from AccountAspect.aj))

2016-11-15 22:53:51 [main] INFO  com.baeldung.aspectj.AccountAspect 
-  Balance before withdrawal: 20
2016-11-15 22:53:51 [main] INFO  com.baeldung.aspectj.AccountAspect 
-  Withdraw ammout: 5
2016-11-15 22:53:51 [main] INFO  com.baeldung.aspectj.AccountAspect 
- Balance after withdrawal : 15
2016-11-15 22:53:51 [main] INFO  com.baeldung.aspectj.AccountAspect 
-  Balance before withdrawal: 20
2016-11-15 22:53:51 [main] INFO  com.baeldung.aspectj.AccountAspect 
-  Withdraw ammout: 100
2016-11-15 22:53:51 [main] INFO  com.baeldung.aspectj.AccountAspect 
- Withdrawal Rejected!
2016-11-15 22:53:51 [main] INFO  com.baeldung.aspectj.AccountAspect 
- Balance after withdrawal : 20
```







## **6. Post-Compile Weaving**



Post-compile weaving (also sometimes called binary weaving) is used to weave existing class files and JAR files. As with compile-time weaving, the aspects used for weaving may be in source or binary form, and may themselves be woven by aspects.

In order to do this with Mojo's AspectJ Maven Plugin we need do setup all the JAR files we would like to weave in the plugin configuration:

```xml
<configuration>
    <weaveDependencies>
        <weaveDependency>  
            <groupId>org.agroup</groupId>
            <artifactId>to-weave</artifactId>
        </weaveDependency>
        <weaveDependency>
            <groupId>org.anothergroup</groupId>
            <artifactId>gen</artifactId>
        </weaveDependency>
    </weaveDependencies>
</configuration>Copy
```

The JAR files containing the classes to weave must be listed as `<dependencies/>` in the Maven project and listed as `<weaveDependencies/>` in the `<configuration>` of the AspectJ Maven Plugin.

## **7. Load-Time Weaving** 



Load-time weaving is simply binary weaving deferred until the point that a class loader loads a class file and defines the class to the JVM.

To support this, one or more “weaving class loaders” are required. These are either provided explicitly by the run-time environment or enabled using a “weaving agent”.

### **7.1. Enabling Load-Time Weaving**



AspectJ load-time weaving can be enabled using AspectJ agent that can get involved in the class loading process and weave any types before they are defined in the VM. We specify the *javaagent* option to the JVM *-javaagent:pathto/aspectjweaver.jar* or using Maven plugin to configure the *javaagent* :

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.22.2</version>
    <configuration>
        <argLine>
            -javaagent:"${settings.localRepository}"/org/aspectj/
            aspectjweaver/${aspectj.version}/
            aspectjweaver-${aspectj.version}.jar
        </argLine>
        <useSystemClassLoader>true</useSystemClassLoader>
        <forkMode>always</forkMode>
    </configuration>
</plugin>
```

### **7.2. Configuration Weaver**



AspectJ's load-time weaving agent is configured by the use of *aop.xml* files. It looks for one or more *aop.xml* files on the classpath in the *META-INF* directory and aggregates the contents to determine the weaver configuration.

An *aop.xml* file contains two key sections:

- **Aspects**: defines one or more aspects to the weaver and controls which aspects are to be used in the weaving process. The *aspects* element may optionally contain one or more *include* and *exclude* elements (by default, all defined aspects are used for weaving)
- **Weaver**: defines weaver options to the weaver and specifies the set of types that should be woven. If no include elements are specified then all types visible to the weaver will be woven

Let's configure an aspect to the weaver:

```xml
<aspectj>
    <aspects>
        <aspect name="com.baeldung.aspectj.AccountAspect"/>
        <weaver options="-verbose -showWeaveInfo">
            <include within="com.baeldung.aspectj.*"/>
        </weaver>
    </aspects>
</aspectj>
```

As we can see, we have configured an aspect that points to the *AccountAspect*, and only the source code in the *com.baeldung.aspectj* package will be woven by AspectJ.

## **8. Annotating Aspects**



In addition to the familiar AspectJ code-based style of aspect declaration, AspectJ 5 also supports an annotation-based style of aspect declaration. We informally call the set of annotations that support this development style the “*@AspectJ*” annotations.

Let's create an annotation:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Secured {
    public boolean isLocked() default false; 
}
```

We use the *@Secured* annotation to enable or disable a method:

```java
public class SecuredMethod {

    @Secured(isLocked = true)
    public void lockedMethod() {
    }

    @Secured(isLocked = false)
    public void unlockedMethod() {
    }
}
```

Next, we add an aspect using AspectJ annotation-style, and check the permission based on the attribute of the @Secured annotation:



```java
@Aspect
public class SecuredMethodAspect {
    @Pointcut("@annotation(secured)")
    public void callAt(Secured secured) {
    }

    @Around("callAt(secured)")
    public Object around(ProceedingJoinPoint pjp, 
      Secured secured) throws Throwable {
        return secured.isLocked() ? null : pjp.proceed();
    }
}
```

For more detail on AspectJ annotation-style, we can check out the following [link](https://eclipse.org/aspectj/doc/released/adk15notebook/annotations-aspectmembers.html).

Next, we weave our class and aspect using load-time weaver and put *aop.xml* under *META-INF* folder:

```xml
<aspectj>
    <aspects>
        <aspect name="com.baeldung.aspectj.SecuredMethodAspect"/>
        <weaver options="-verbose -showWeaveInfo">
            <include within="com.baeldung.aspectj.*"/>
        </weaver>
    </aspects>
</aspectj>
```

Finally, we add unit test and check the result:

```java
@Test
public void testMethod() throws Exception {
	SecuredMethod service = new SecuredMethod();
	service.unlockedMethod();
	service.lockedMethod();
}
```

When we run the test cases, we may check the console output to verify that we successfully weaved our aspect and class in the source code:

```java
[INFO] Join point 'method-call
(void com.baeldung.aspectj.SecuredMethod.unlockedMethod())'
in Type 'com.baeldung.aspectj.test.SecuredMethodTest'
(SecuredMethodTest.java:11)
advised by around advice from 'com.baeldung.aspectj.SecuredMethodAspect'
(SecuredMethodAspect.class(from SecuredMethodAspect.java))

2016-11-15 22:53:51 [main] INFO com.baeldung.aspectj.SecuredMethod 
- unlockedMethod
2016-11-15 22:53:51 [main] INFO c.b.aspectj.SecuredMethodAspect - 
public void com.baeldung.aspectj.SecuredMethod.lockedMethod() is locked
```



## 9. Conclusion**

In this article, we covered introductory concepts about AspectJ. For details, you can take a look at the [AspectJ home page.](https://eclipse.org/aspectj/)

You can find the source code for this article [over on GitHub](https://github.com/eugenp/tutorials/tree/master/spring-aop)

