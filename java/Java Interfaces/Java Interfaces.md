# Java Interfaces



## 1. Overview



In this tutorial, we're going to talk about interfaces in Java. We'll also see how Java uses them to implement polymorphism and multiple inheritances.



## 2. What Are Interfaces in Java?

In Java, an interface is an abstract type that contains a collection of methods and constant variables. It is one of the core concepts in Java and is **used to achieve abstraction, polymorphism and multiple inheritances**.

Let's see a simple example of an interface in Java:

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

We can implement an interface in a Java class by using the *implements* keyword.

Next, let's also create a *Computer* class that implements the *Electronic* interface we just created:

```java
public class Computer implements Electronic {

    @Override
    public int getElectricityUse() {
        return 1000;
    }
}

```





### 2.1. Rules for Creating Interfaces

In an interface, we're allowed to use:

- [constants variables](https://www.baeldung.com/java-final)
- [abstract methods](https://www.baeldung.com/java-abstract-class)
- [static methods](https://www.baeldung.com/java-static-default-methods)
- [default methods](https://www.baeldung.com/java-static-default-methods)

We also should remember that:

- we can't instantiate interfaces directly
- an interface can be empty, with no methods or variables in it
- we can't use the *final* word in the interface definition, as it will result in a compiler error
- all interface declarations should have the *public* or default access modifier; the *abstract* modifier will be added automatically by the compiler
- an interface method can't be *protected* or *final*
- up until Java 9, interface methods could not be *private*; however, Java 9 introduced the possibility to define [private methods in interfaces](https://www.baeldung.com/java-interface-private-methods)
- interface variables are *public*, *static*, and *final* by definition; we're not allowed to change their visibility





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