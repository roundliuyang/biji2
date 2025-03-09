# JUnit 5 指南

## **概述**

[JUnit](http://junit.org/junit5/)是 Java 生态系统中最流行的单元测试框架之一。JUnit 5 版本包含许多令人兴奋的创新，**旨在支持 Java 8 及更高版本中的新功能**，并支持多种不同的测试风格。



##  **Maven 依赖项**

设置 [JUnit 5.x.0非常简单；我们只需要在](https://mvnrepository.com/artifact/org.junit.jupiter/junit-jupiter-engine)*pom.xml*中添加以下依赖项：

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <version>5.11.0-M2</version>
    <scope>test</scope>
</dependency>
```

此外，现在 Eclipse 和 IntelliJ 都直接支持在 JUnit Platform 上运行单元测试。当然，我们也可以使用 Maven 的 Test 目标来运行测试。

另一方面，IntelliJ 默认支持 JUnit 5，因此在 IntelliJ 上运行 JUnit 5 非常简单。我们只需 **右键 → 运行**，或使用快捷键 **Ctrl + Shift + F10**。

需要注意的是，该版本要求 **Java 8** 及以上才能运行。



## 架构

JUnit 5 由三个不同子项目的多个模块组成。



### JUnit 平台（JUnit Platform）

该平台负责在 JVM 上启动测试框架，并定义了 JUnit 与其客户端（如构建工具）之间的稳定且强大的接口。

该平台可以轻松将客户端与 JUnit 集成，以发现和执行测试。

此外，它还定义了 **TestEngine API**，用于开发运行在 JUnit 平台上的测试框架。通过实现自定义 **TestEngine**，我们可以直接将第三方测试库集成到 JUnit 中。



### **JUnit Jupiter**

该模块引入了新的编程和扩展模型，以便在 JUnit 5 中编写测试。与 JUnit 4 相比，新增的注解包括：

- **@TestFactory** – 表示该方法是用于动态测试的测试工厂
- **@DisplayName** – 为测试类或测试方法定义自定义显示名称
- **@Nested** – 表示该类是嵌套的非静态测试类
- **@Tag** – 声明标签，用于筛选测试
- **@ExtendWith** – 注册自定义扩展
- **@BeforeEach** – 该方法将在每个测试方法执行之前运行（前身为 **@Before**）
- **@AfterEach** – 该方法将在每个测试方法执行之后运行（前身为 **@After**）
- **@BeforeAll** – 该方法将在当前类的所有测试方法执行之前运行（前身为 **@BeforeClass**）
- **@AfterAll** – 该方法将在当前类的所有测试方法执行之后运行（前身为 **@AfterClass**）
- **@Disabled** – 禁用测试类或方法（前身为 **@Ignore**）



### **JUnit Vintage**

JUnit Vintage 支持在 JUnit 5 平台上运行基于 JUnit 3 和 JUnit 4 的测试。



## 基本注解

 为了讨论新的注解，我们将本节分为以下几个负责执行的组别：测试之前、测试期间（可选）以及测试之后。



### @BeforeAll  and @BeforeEach

下面是一个在主测试用例执行之前运行的简单代码示例：

```java
@BeforeAll
static void setup() {
    log.info("@BeforeAll - executes once before all test methods in this class");
}

@BeforeEach
void init() {
    log.info("@BeforeEach - executes before each test method in this class");
}
```

需要注意的是，带有 `@BeforeAll` 注解的方法必须是静态的，否则代码将无法编译。



### @DisplayName and @Disabled

现在让我们来看一些新的可选测试方法：

```java
@DisplayName("Single test successful")
@Test
void testSingleSuccessTest() {
    log.info("Success");
}

@Test
@Disabled("Not implemented yet")
void testShowSomething() {
}
```

正如我们所看到的，我们可以使用新的注解更改显示名称或通过注释禁用该方法。



### @AfterEach and @AfterAll

最后，让我们讨论与测试执行后操作相关的方法：

```java
@AfterEach
void tearDown() {
    log.info("@AfterEach - executed after each test method.");
}

@AfterAll
static void done() {
    log.info("@AfterAll - executed after all test methods.");
}
```

Please note that the method with *@AfterAll* also needs to be a static method.



## **Assertions and Assumptions**

JUnit 5 尽力充分利用 Java 8 的新特性，尤其是 lambda 表达式。



###  **Assertions**

断言已经移动到 `org.junit.jupiter.api.Assertions`，并且得到了显著改进。如前所述，我们现在可以在断言中使用 lambda 表达式：

```java
@Test
void lambdaExpressions() {
    List numbers = Arrays.asList(1, 2, 3);
    assertTrue(numbers.stream()
      .mapToInt(Integer::intValue)
      .sum() > 5, () -> "Sum should be greater than 5");
}
```

虽然上面的例子很简单，但使用 lambda 表达式作为断言消息的一个优点是它是延迟评估的。如果消息的构造开销较大，这样可以节省时间和资源。

现在还可以使用 `assertAll()` 来分组断言，它会报告组内任何失败的断言，并抛出 `MultipleFailuresError`：

```java
 @Test
 void groupAssertions() {
     int[] numbers = {0, 1, 2, 3, 4};
     assertAll("numbers",
         () -> assertEquals(numbers[0], 1),
         () -> assertEquals(numbers[3], 3),
         () -> assertEquals(numbers[4], 1)
     );
 }
```

这意味着现在进行更复杂的断言更安全，因为我们能够准确定位到任何失败的具体位置。



### **Assumptions**

假设（Assumptions）用于仅在满足特定条件时运行测试。这通常用于测试所需的外部条件，这些条件与正在测试的内容没有直接关系。

我们可以使用 `assumeTrue()`、`assumeFalse()` 和 `assumingThat()` 来声明假设：

```java
@Test
void trueAssumption() {
    assumeTrue(5 > 1);
    assertEquals(5 + 2, 7);
}

@Test
void falseAssumption() {
    assumeFalse(5 < 1);
    assertEquals(5 + 2, 7);
}

@Test
void assumptionThat() {
    String someString = "Just a string";
    assumingThat(
        someString.equals("Just a string"),
        () -> assertEquals(2 + 2, 4)
    );
}
```

如果假设失败，将抛出 `TestAbortedException` 异常，并且测试将被跳过。

假设也支持使用 lambda 表达式。





## **Exception Testing**

JUnit 5 中有两种异常测试方法，我们可以通过 `assertThrows()` 方法来实现这两种方式：

```java
@Test
void shouldThrowException() {
    Throwable exception = assertThrows(UnsupportedOperationException.class, () -> {
      throw new UnsupportedOperationException("Not supported");
    });
    assertEquals("Not supported", exception.getMessage());
}

@Test
void assertThrowsException() {
    String str = null;
    assertThrows(IllegalArgumentException.class, () -> {
      Integer.valueOf(str);
    });
}
```

第一个示例验证了抛出异常的详细信息，而第二个示例验证了异常的类型。

## **Test Suites**

继续探索JUnit 5的新特性，我们将探讨将多个测试类聚合到一个测试套件中的概念，以便一起运行这些测试。JUnit 5提供了两个注解，@SelectPackages和@SelectClasses，用于创建测试套件。

请注意，在这个早期阶段，大多数IDE并不支持这些特性。

让我们来看一下第一个注解：

```java
@Suite
@SelectPackages("com.baeldung")
@ExcludePackages("com.baeldung.suites")
public class AllUnitTest {}
```

@SelectPackages 用于指定在运行测试套件时要选择的包的名称。在我们的示例中，它将运行所有的测试。第二个注解 @SelectClasses 用于指定在运行测试套件时要选择的类：

```java
@Suite
@SelectClasses({AssertionTest.class, AssumptionTest.class, ExceptionTest.class})
public class AllUnitTest {}
```

例如，上面的类将创建一个包含三个测试类的测试套件。请注意，这些类不必在同一个包中。



## Dynamic Tests

我们要介绍的最后一个主题是JUnit 5的动态测试功能，它允许我们在运行时声明和运行生成的测试用例。与静态测试不同，静态测试在编译时定义了固定数量的测试用例，而动态测试则允许我们在运行时动态地定义测试用例。

动态测试可以通过使用@TestFactory注解的工厂方法来生成。让我们看一下代码：

```java
@TestFactory
Stream<DynamicTest> translateDynamicTestsFromStream() {
    return in.stream()
      .map(word ->
          DynamicTest.dynamicTest("Test translate " + word, () -> {
            int id = in.indexOf(word);
            assertEquals(out.get(id), translate(word));
          })
    );
}
```

这个例子非常简单，容易理解。我们希望使用两个ArrayList（分别命名为in和out）来翻译单词。工厂方法必须返回一个Stream、Collection、Iterable或Iterator。在我们的例子中，我们选择了Java 8的Stream。

请注意，@TestFactory方法不能是private或static的。测试的数量是动态的，取决于ArrayList的大小。