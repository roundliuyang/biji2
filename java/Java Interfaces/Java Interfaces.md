# Java Interfaces



## 1. 概述

在本教程中，我们将讨论 Java 中的接口。我们还将了解 Java 如何使用它们来实现多态性和多重继承。



## 2. Java 中的接口是什么？

在 Java 中，接口是一种抽象类型，它包含一组方法和常量变量。接口是 Java 的核心概念之一，用于实现抽象、多态和多重继承。

我们来看一个 Java 中接口的简单例子：

```java
public interface Electronic {

    // Constant variable
    String LED = "LED";

    // Abstract method
    int getElectricityUse();

    // Static method
    static boolean isEnergyEfficient(String electtronicType) {
        if (electtronicType.equals(LED)) {
            return true;
        }
        return false;
    }

    //Default method
    default void printDescription() {
        System.out.println("Electronic Description");
    }
}

```

我们可以通过使用 `implements` 关键字在 Java 类中实现一个接口。

接下来，让我们创建一个 `Computer` 类来实现我们刚刚创建的 `Electronic` 接口：

```java
public class Computer implements Electronic {

    @Override
    public int getElectricityUse() {
        return 1000;
    }
}

```





### 2.1. 创建接口的规则

在接口中，我们可以使用：

- 常量变量
- 抽象方法
- 静态方法
- 默认方法

我们还应该记住：

- 我们不能直接实例化接口
- 接口可以是空的，即没有任何方法或变量
- 接口定义中不能使用 `final` 关键字，否则会导致编译错误
- 所有接口声明应使用 `public` 或默认访问修饰符；`abstract` 修饰符会由编译器自动添加
- 接口中的方法不能是 `protected` 或 `final`
- 在 Java 9 之前，接口方法不能是 `private`；但从 Java 9 开始，接口中可以定义私有方法
- 接口中的变量默认是 `public`、`static` 和 `final`；我们不能更改它们的可见性



## 3. What Can We Achieve by Using Them?



### 3.1. Behavioral Functionality

We use interfaces to add certain behavioral functionality that can be used by unrelated classes. For instance, *Comparable*, *Comparator*, and *Cloneable* are Java interfaces that can be implemented by unrelated classes. Below is an example of the *Comparator* interface that is used to compare two instances of the *Employee* class:

```java
public class Employee {

    private double salary;

    public double getSalary() {
        return salary;
    }

    public void setSalary(double salary) {
        this.salary = salary;
    }
}

public class EmployeeSalaryComparator implements Comparator<Employee> {

    @Override
    public int compare(Employee employeeA, Employee employeeB) {
        if (employeeA.getSalary() < employeeB.getSalary()) {
            return -1;
        } else if (employeeA.getSalary() > employeeB.getSalary()) { 
            return 1;
        } else {
            return 0;
        }
    }
}

```

For more information, please visit our tutorial on [*Comparator* and *Comparable* in Java.](https://www.baeldung.com/java-comparator-comparable)



### 3.2. Multiple Inheritances

Java classes support singular inheritance. However, by using interfaces, we're also able to implement multiple inheritances.

For instance, in the example below, we notice that the *Car* class implements the *Fly* and *Transform* interfaces. By doing so, it inherits the methods *fly* and *transform*:

```java
public interface Transform {
    void transform();
}

public interface Fly {
    void fly();
}

public class Car implements Fly, Transform {

    @Override
    public void fly() {
        System.out.println("I can Fly!!");
    }

    @Override
    public void transform() {
        System.out.println("I can Transform!!");
    }
}

```



### 3.3. Polymorphism

Let's start with asking the question: what is [polymorphism](https://www.baeldung.com/java-polymorphism)? It's the ability for an object to take different forms during runtime. To be more specific it's the execution of the override method that is related to a specific object type at runtime.

**In Java, we can achieve polymorphism using interfaces.** For example, the *Shape* interface can take different forms — it can be a *Circle* or a *Square.*

Let's start by defining the *Shape* interface:

```java
public interface Shape {
    String name();
}

```

Now let's also create the *Circle* class:

```java
public class Circle implements Shape {

    @Override
    public String name() {
        return "Circle";
    }
}

```

And also the *Square* class:

```java
public class Square implements Shape {

    @Override
    public String name() {
        return "Square";
    }
}

```



Finally, it's time to see polymorphism in action using our *Shape* interface and its implementations. Let's instantiate some *Shape* objects, add them to a List,and, finally, print their names in a loop:

```java
List<Shape> shapes = new ArrayList<>();
Shape circleShape = new Circle();
Shape squareShape = new Square();

shapes.add(circleShape);
shapes.add(squareShape);

for (Shape shape : shapes) {
    System.out.println(shape.name());
}

```



## 4. Default Methods in Interfaces

Traditional interfaces in Java 7 and below don't offer backward compatibility.

What this means is that **if you have legacy code written in Java 7 or earlier, and you decide to add an abstract method to an existing interface, then all the classes that implement that interface must override the new abstract method**. Otherwise, the code will break.

**Java 8 solved this problem by introducing the default method** that is optional and can be implemented at the interface level.





## 5. Interface Inheritance Rules

In order to achieve multiple inheritances thru interfaces, we have to remember a few rules. Let's go over these in detail.



### 5.1. Interface Extending Another Interface

When an interface *extends* another interface, it inherits all of that interface's abstract methods. Let's start by creating two interfaces, *HasColor* and *Shape*:

```java
public interface HasColor {
    String getColor();
}

public interface Box extends HasColor {
    int getHeight()
}

```

In the example above, *Box* inherits from *HasColor* using the keyword *extends.* By doing so, the *Box* interface inherits *getColor*. As a result, the *Box* interface now has two methods: *getColor* and *getHeight*.



### 5.2. Abstract Class Implementing an Interface

When an abstract class implements an interface, it inherits all of its abstract and default methods. Let's consider the *Transform* interface and the *abstract* class *Vehicle* that implements it:

```java
public interface Transform {
    
    void transform();
    default void printSpecs(){
        System.out.println("Transform Specification");
    }
}

public abstract class Vehicle implements Transform {}

```

In this example, the *Vehicle* class inherits two methods: the abstract *transform* method and the default *printSpecs* method.



## 6. Functional Interfaces

Java has had many functional interfaces since its early days, such as *Comparable* (since Java 1.2) and *Runnable* (since Java 1.0).

Java 8 introduced new functional interfaces such as *Predicate*, *Consumer*, and *Function*. To learn more about these, please visit our tutorial on [Functional Interfaces in Java 8](https://www.baeldung.com/java-8-functional-interfaces).



## 7. Conclusion

In this tutorial, we gave an overview of Java interfaces, and we talked about how to use them to achieve polymorphism and multiple inheritances.

As always, the complete code samples are [available over on GitHub.](https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-lang-oop-inheritance)