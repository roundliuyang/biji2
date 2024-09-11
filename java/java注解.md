Java Annotations 提供有关代码的信息。Java 注解对它们注解的代码没有直接影响。在java注解教程中，我们将研究以下内容；

1. 内置 Java 注解
2. 如何编写自定义注解
3. 注解用法以及如何使用[反射 API](https://www.journaldev.com/1789/java-reflection-example-tutorial)解析注解。

### Java注解

Java 1.5 引入了注解，现在它在 Hibernate、[Jersey](https://www.journaldev.com/498/jersey-java-tutorial)和[Spring](https://www.journaldev.com/2888/spring-tutorial-spring-core-tutorial)等 Java EE 框架中得到了大量使用。

Java Annotation 是关于嵌入在程序本身中的程序的元数据。可以通过注解解析工具解析，也可以通过编译器解析。我们还可以将注解可用性指定为仅编译时或直到运行时。

### java  自定义注解

创建自定义注解类似于编写接口，不同之处在于interface 关键字以**@**符号为前缀。我们可以在注解中声明方法。

让我们看看java自定义注解的例子，然后我们将讨论它的特点和要点。

```java
package com.journaldev.annotations;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Documented
@Target(ElementType.METHOD)
@Inherited
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodInfo{
	String author() default "Pankaj";
	String date();
	int revision() default 1;
	String comments();
}
```

关于java 注解的一些要点是：

1.注解方法不能有参数

2.注解方法返回类型仅限于原始类型、字符串、枚举、注解或这些类型的数组。

3.java Annotation 方法可以有默认值

4.注解可以有元注解，元注解用于提供有关注解的信息。



### Java 中的内置注解

java提供了五个内置注解

1. `@Override`– 当我们想覆盖超类的一个方法时，我们应该使用这个注解来通知编译器我们正在覆盖一个方法。因此，当移除或更改超类方法时，编译器将显示错误消息。了解为什么我们应该在覆盖方法时始终使用[java 覆盖注释](https://www.journaldev.com/817/java-override-method-overriding)。
2. `@Deprecated`– 当我们希望编译器知道某个方法已被弃用时，我们应该使用这个注解。Java 建议在 javadoc 中，我们应该提供有关为什么不推荐使用此方法以及可以使用的替代方法的信息。
3. `@SuppressWarnings`– 这只是告诉编译器忽略它们产生的特定警告，例如在[java generics 中](https://www.journaldev.com/1663/java-generics-example-method-class-interface)使用原始类型。它的保留策略是 SOURCE，它会被编译器丢弃。
4. `@FunctionalInterface`– 这个注解是在[Java 8](https://www.journaldev.com/2389/java-8-features-with-examples)中引入的，以表明该接口旨在成为一个[功能接口](https://www.journaldev.com/2763/java-8-functional-interfaces)。
5. `@SafeVarargs` – 程序员断言带注释的方法或构造函数的主体不会对其 varargs 参数执行潜在的不安全操作。

### java 注解示例

让我们看一个java例子，展示了java中内置注解的使用以及我们上面的例子中创建的自定义注解的使用。

```java
package com.journaldev.annotations;

import java.io.FileNotFoundException;
import java.util.ArrayList;
import java.util.List;

public class AnnotationExample {

	public static void main(String[] args) {
	}

	@Override
	@MethodInfo(author = "Pankaj", comments = "Main method", date = "Nov 17 2012", revision = 1)
	public String toString() {
		return "Overriden toString method";
	}

	@Deprecated
	@MethodInfo(comments = "deprecated method", date = "Nov 17 2012")
	public static void oldMethod() {
		System.out.println("old method, don't use it.");
	}

	@SuppressWarnings({ "unchecked", "deprecation" })
	@MethodInfo(author = "Pankaj", comments = "Main method", date = "Nov 17 2012", revision = 10)
	public static void genericsTest() throws FileNotFoundException {
		List l = new ArrayList();
		l.add("abc");
		oldMethod();
	}

}
```

我相信上面的 java 注解示例是不言自明的，并显示了注释在不同情况下的使用。



### java 注解解析

我们将使用反射来解析类中的 Java 注解。请注意 Annotation Retention Policy 应该是*RUNTIME*否则它的信息在运行时将不可用，我们将无法从中获取任何数据。

```java
package com.journaldev.annotations;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

public class AnnotationParsing {

	public static void main(String[] args) {
		try {
			for (Method method : AnnotationParsing.class.getClassLoader()
					.loadClass(("com.journaldev.annotations.AnnotationExample")).getMethods()) {
				// checks if MethodInfo annotation is present for the method
				if (method.isAnnotationPresent(com.journaldev.annotations.MethodInfo.class)) {
					try {
						// iterates all the annotations available in the method
						for (Annotation anno : method.getDeclaredAnnotations()) {
							System.out.println("Annotation in Method '" + method + "' : " + anno);
						}
						MethodInfo methodAnno = method.getAnnotation(MethodInfo.class);
						if (methodAnno.revision() == 1) {
							System.out.println("Method with revision no 1 = " + method);
						}

					} catch (Throwable ex) {
						ex.printStackTrace();
					}
				}
			}
		} catch (SecurityException | ClassNotFoundException e) {
			e.printStackTrace();
		}
	}

}
```

上述程序的输出为：

```java
Annotation in Method 'public java.lang.String com.journaldev.annotations.AnnotationExample.toString()' : @com.journaldev.annotations.MethodInfo(author=Pankaj, revision=1, comments=Main method, date=Nov 17 2012)
Method with revision no 1 = public java.lang.String com.journaldev.annotations.AnnotationExample.toString()
Annotation in Method 'public static void com.journaldev.annotations.AnnotationExample.oldMethod()' : @java.lang.Deprecated()
Annotation in Method 'public static void com.journaldev.annotations.AnnotationExample.oldMethod()' : @com.journaldev.annotations.MethodInfo(author=Pankaj, revision=1, comments=deprecated method, date=Nov 17 2012)
Method with revision no 1 = public static void com.journaldev.annotations.AnnotationExample.oldMethod()
Annotation in Method 'public static void com.journaldev.annotations.AnnotationExample.genericsTest() throws java.io.FileNotFoundException' : @com.journaldev.annotations.MethodInfo(author=Pankaj, revision=10, comments=Main method, date=Nov 17 2012)
```

反射 API 非常强大，在 Java、J2EE 框架（如 Spring、[Hibernate](https://www.journaldev.com/3793/hibernate-tutorial)、JUnit）中广泛使用，请查看**Java 中的反射**。