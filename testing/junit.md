# junit

## **JUnit 4** 

如果你只想使用 **JUnit 4** 而不涉及 JUnit 5，那么你只需要 **`junit` 依赖**，通常使用的是 `junit:junit`，最新稳定版是 `4.13.2`（截至目前）。



### **Maven 依赖**

在 `pom.xml` 中添加以下内容：

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
</dependency>
```



### **Gradle 依赖**

如果使用 Gradle，可以在 `build.gradle` 中添加：

```
dependencies {
    testImplementation 'junit:junit:4.13.2'
}
```

------



### **JUnit 4 代码示例**

```
import org.junit.Test;
import static org.junit.Assert.assertEquals;

public class JUnit4Test {
    @Test
    public void testAddition() {
        assertEquals(2, 1 + 1);
    }
}
```

------



### **说明**

1. **不需要 `junit-jupiter-engine`**：因为 `junit:junit` 依赖本身包含了运行 JUnit 4 测试的必要组件。

2. **JUnit 4 依赖 `org.junit.Assert`** 进行断言，而不是 `org.junit.jupiter.api.Assertions`（后者是 JUnit 5 的）。

3. JUnit 4 需要 `@RunWith`

    进行某些测试执行方式的扩展，比如 Spring 测试：

   ```
   @RunWith(SpringRunner.class)
   public class MySpringTest {
   }
   ```

如果你只需要 JUnit 4，就只要 `junit:junit` 这个依赖，无需其他配置。



## junit-jupiter

spring-boot 2.7.9 版本依赖 junit-jupiter5.8.2 版本

```xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>5.8.2</version>
  <scope>compile</scope>
</dependency>
```



**点进 junit-jupiter 5.8.2**

```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.junit</groupId>
        <artifactId>junit-bom</artifactId>
        <version>5.8.2</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter-api</artifactId>
      <version>5.8.2</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter-params</artifactId>
      <version>5.8.2</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter-engine</artifactId>
      <version>5.8.2</version>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
```



## junit-jupiter-api 和 junit-jupiter-engine 之间的区别



### JUnit 5.4 之前的版本

**`junit-jupiter-api`**

JUnit Jupiter API 用于编写测试和扩展。

**`junit-jupiter-engine`**

JUnit Jupiter 测试引擎实现，仅在运行时需要。

**`junit-vintage-engine`**

JUnit Vintage 测试引擎实现允许在新 JUnit 平台上运行复古 JUnit 测试，即以 JUnit 3 或 JUnit 4 风格编写的测试。

所以 ...

- 要编写和运行 JUnit 5 测试，你需要同时引入 `junit-jupiter-api` 和 `junit-jupiter-engine`。

  值得注意的是，`junit-jupiter-api` 已作为子依赖包含在 `junit-jupiter-engine` 的 Maven 仓库中。所以你只需要添加 `junit-jupiter-engine` 就可以同时获得这两个依赖。

-  只有在以下情况下才需要 `junit-vintage-engine`：

  - (a) 你正在使用 JUnit 5 运行测试，并且

  - (b) 你的测试用例仍然使用 JUnit 4 的结构、注解、规则等。

    

### JUnit 5.4 及以上版本

在 JUnit 5.4 中，这一点得到了简化，有关更多详细信息，请参阅[此答案。](https://stackoverflow.com/a/55084036/8200937)



#### junit-jupiter aggregator artifact

从 JUnit 5.4+ 开始，如果你打算编写 JUnit 5 测试，Maven 配置变得更加简单。
 只需指定名为 `junit-jupiter` 的聚合依赖即可。

```xml
<!-- https://mvnrepository.com/artifact/org.junit.jupiter/junit-jupiter -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.12.0</version>
    <scope>test</scope>
</dependency>
```

作为一个聚合依赖，该 artifact 会自动引入以下三个依赖，方便使用：

- **junit-jupiter-api**（编译时依赖）
- **junit-jupiter-params**（编译时依赖）
- **junit-jupiter-engine**（运行时依赖）

在你的项目中，你还会得到以下文件：

- **junit-platform-commons-1.4.0.jar**
- **junit-platform-engine-1.4.0.jar**

上述内容是基于全新的 **Jupiter** 模式编写和运行 **JUnit 5** 测试所需的依赖。



#### 遗留测试

如果你的项目中有 JUnit 3 或 JUnit 4 测试，并且希望继续运行这些测试，可以添加另一个依赖项，即 **JUnit Vintage Engine**，即 **junit-vintage-engine**。请参考 IBM 的教程。

```xml
<!-- https://mvnrepository.com/artifact/org.junit.vintage/junit-vintage-engine -->
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <version>5.12.0</version>
    <scope>test</scope>
</dependency>
```



