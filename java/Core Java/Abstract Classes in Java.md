# Abstract Classes in Java



## **1. Overview**

There are many cases when implementing a contract where we want to postpone some parts of the implementation to be completed later. We can easily accomplish this in Java through abstract classes.

**In this tutorial, we'll learn the basics of abstract classes in Java, and in what cases they can be helpful**.



## **2. Key Concepts for Abstract Classes**

Before diving into when to use an abstract class, **let's look at their most relevant characteristics**:

- **We define an abstract class with the abstract modifier preceding the class keyword**
- An abstract class can be subclassed, but it can't be instantiated
- If a class defines one or more *abstract* methods, then the class itself must be declared *abstract*
- **An abstract class can declare both abstract and concrete methods**
- A subclass derived from an abstract class must either implement all the base class's abstract methods or be abstract itself

To better understand these concepts, we'll create a simple example.

Let's have our base abstract class define the abstract API of a board game:

```java
public abstract class BoardGame {

    //... field declarations, constructors

    public abstract void play();

    //... concrete methods
}
```

Then, we can create a subclass that implements the *play* method:

```java
public class Checkers extends BoardGame {

    public void play() {
        //... implementation
    }
}
```



## 3. When to Use Abstract Classes

Now, **let's analyze a few typical scenarios where we should prefer abstract classes over interfaces** and concrete classes:

- We want to encapsulate some common functionality in one place (code reuse) that multiple, related subclasses will share
- We need to partially define an API that our subclasses can easily extend and refine
- The subclasses need to inherit one or more common methods or fields with protected access modifiers

Let's keep in mind that all these scenarios are good examples of full, inheritance-based adherence to the [Open/Closed principle](https://en.wikipedia.org/wiki/Open–closed_principle).

Moreover, since the use of abstract classes implicitly deals with base types and subtypes, we're also taking advantage of [Polymorphism](https://www.baeldung.com/java-polymorphism).

Note that code reuse is a very compelling reason to use abstract classes, as long as the “is-a” relationship within the class hierarchy is preserved.



**And Java 8 adds another wrinkle with default methods, which can sometimes take the place of needing to create an abstract class altogether.**



## **4. A Sample Hierarchy of File Readers** 

To understand more clearly the functionality that abstract classes bring to the table, let's look at another example.



### **4.1. Defining a Base Abstract Class**

So, if we wanted to have several types of file readers, we might create an abstract class that encapsulates what's common to file reading:

```java
public abstract class BaseFileReader {
    
    protected Path filePath;
    
    protected BaseFileReader(Path filePath) {
        this.filePath = filePath;
    }
    
    public Path getFilePath() {
        return filePath;
    }
    
    public List<String> readFile() throws IOException {
        return Files.lines(filePath)
          .map(this::mapFileLine).collect(Collectors.toList());
    }
    
    protected abstract String mapFileLine(String line);
}
```

Note that we've made *filePath* protected so that the subclasses can access it if needed. More importantly, **we've left something undone: how to actually parse a line of text** from the file's contents.

Our plan is simple: while our concrete classes don't each have a special way to store the file path or walk through the file, they will each have a special way to transform each line.



At first sight, *BaseFileReader* may seem unnecessary. However, it's the foundation of a clean, easily extendable design. From it, **we can easily implement different versions of a file reader that can focus on their unique business logic**.



### **4.2. Defining Subclasses**

A natural implementation is probably one that converts a file's contents to lowercase:

```java
public class LowercaseFileReader extends BaseFileReader {

    public LowercaseFileReader(Path filePath) {
        super(filePath);
    }

    @Override
    public String mapFileLine(String line) {
        return line.toLowerCase();
    }   
}
```

Or another might be one that converts a file's contents to uppercase:

```java
public class UppercaseFileReader extends BaseFileReader {

    public UppercaseFileReader(Path filePath) {
        super(filePath);
    }

    @Override
    public String mapFileLine(String line) {
        return line.toUpperCase();
    }
}
```

As we can see from this simple example, **each subclass can focus on its unique behavior** without needing to specify other aspects of file reading.



### **4.3. Using a Subclass**

Finally, using a class that inherits from an abstract one is no different than any other concrete class:

```java
@Test
public void givenLowercaseFileReaderInstance_whenCalledreadFile_thenCorrect() throws Exception {
    URL location = getClass().getClassLoader().getResource("files/test.txt")
    Path path = Paths.get(location.toURI());
    BaseFileReader lowercaseFileReader = new LowercaseFileReader(path);
        
    assertThat(lowercaseFileReader.readFile()).isInstanceOf(List.class);
}
```

For simplicity's sake, the target file is located under the *src/main/resources/files* folder. Hence, we used an application class loader for getting the path of the example file. Feel free to check out [our tutorial on class loaders in Java](https://www.baeldung.com/java-classloaders).



## **5. Conclusion**

In this quick article, **we learned the basics of abstract classes in Java, and when to use them for achieving abstraction and encapsulating common implementation in one single place**.

As usual, all the code samples shown in this tutorial are available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-lang-oop-inheritance).