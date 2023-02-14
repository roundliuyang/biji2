# Java SPI(Service Provider Interface)简介



## SPI 简介

SPI 全称为 (Service Provider Interface) ,是JDK内置的一种服务提供发现机制。

一个服务(Service)通常指的是已知的接口或者抽象类，服务提供方就是对这个接口或者抽象类的实现，然后按照SPI 标准存放到资源路径META-INF/services目录下，文件的命名为该服务接口的全限定名。如有一个服务接口：

```java
package com.ricky.codelab.spi;

public interface DemoService {

    public String sayHi(String msg);
}
```

其服务实现类为：

```java
package com.ricky.codelab.spi.impl;

import com.ricky.codelab.spi.DemoService;

public class DemoServiceImpl implements DemoService {

    @Override
    public String sayHi(String msg) {

        return "Hello, "+msg;
    }

}
```

那此时需要在META-INF/services中创建一个名为com.ricky.codelab.spi.DemoService的文件，其中的内容就为该实现类的全限定名：com.ricky.codelab.spi.impl.DemoServiceImpl。
如果该Service有多个服务实现，则每一行写一个服务实现（#后面的内容为注释），并且该文件只能够是以UTF-8编码。

然后，我们可以通过ServiceLoader.load(Class class); 来动态加载Service的实现类了。

许多开发框架都使用了Java的SPI机制，如java.sql.Driver的SPI实现（mysql驱动、oracle驱动等）、common-logging的日志接口实现、dubbo的扩展实现等等。

## SPI机制的约定

在META-INF/services/目录中创建以Service接口全限定名命名的文件，该文件内容为Service接口具体实现类的全限定名，文件编码必须为UTF-8。
使用ServiceLoader.load(Class class); 动态加载Service接口的实现类。
如SPI的实现类为jar，则需要将其放在当前程序的classpath下。
Service的具体实现类必须有一个不带参数的构造方法。



## 示例

### 1、项目结构

![这里写图片描述](Java SPI(Service Provider Interface)简介.assets/20160718101919862)



### 2、Service接口定义

```java
package com.ricky.codelab.spi;

public interface DemoService {

    public String sayHi(String msg);
}
```



### 3、Service接口实现类

本示例中DemoService有两个实现类，分别为：EnglishDemoServiceImpl和ChineseDemoServiceImpl，代码如下：
**EnglishDemoServiceImpl.java**

```java
package com.ricky.codelab.spi.impl;

import com.ricky.codelab.spi.DemoService;

public class EnglishDemoServiceImpl implements DemoService {

    @Override
    public String sayHi(String msg) {

        return "Hello, "+msg;
    }

}
```

**ChineseDemoServiceImpl.java**

```java
package com.ricky.codelab.spi.impl;

import com.ricky.codelab.spi.DemoService;

public class ChineseDemoServiceImpl implements DemoService {

    @Override
    public String sayHi(String msg) {

        return "你好, "+msg;
    }

}
```

### 4、META-INF/services/配置

在src/main/resources 下创建META-INF/services/目录，并新建com.ricky.codelab.spi.DemoService文件，内容如下：

```
#English implementation
com.ricky.codelab.spi.impl.EnglishDemoServiceImpl

#Chinese implementation
com.ricky.codelab.spi.impl.ChineseDemoServiceImpl
```

### 5、加载Service实现类

```java
import java.util.Iterator;
import java.util.ServiceLoader;
import com.ricky.codelab.spi.DemoService;

ServiceLoader<DemoService> serviceLoader = ServiceLoader.load(DemoService.class);
Iterator<DemoService> it = serviceLoader.iterator();
while (it!=null && it.hasNext()) {
     DemoService demoService = it.next();
 System.out.println("class:"+demoService.getClass().getName()+"***"+demoService.sayHi("World"));
}
```

运行结果：

```
class:com.ricky.codelab.spi.impl.DemoServiceImpl***Hello, World
class:com.ricky.codelab.spi.impl.ChineseDemoServiceImpl***你好, World
```









