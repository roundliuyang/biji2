# Guide to @ConfigurationProperties in Spring Boot



## **1. Introduction**



Spring Boot has many useful features including **externalized configuration and easy access to properties defined in properties files**. An earlier [tutorial](https://www.baeldung.com/properties-with-spring) described various ways in which this could be done.

We are now going to explore the *@ConfigurationProperties* annotation in greater detail.

## **2. Setup**

This tutorial uses a fairly standard setup. We start by adding [*spring-boot-starter-parent*](https://search.maven.org/search?q=a:spring-boot-starter-parent AND g:org.springframework.boot) as the parent in our *pom.xml*:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.0.0</version>
    <relativePath/>
</parent>Copy
```

To be able to validate properties defined in the file, we also need an implementation of JSR-380, and hibernate-validator is one of them and provided by *spring-boot-starter-validation* dependency.

Let's add it to our *pom.xml* as well:

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

The [“Getting Started with Hibernate Validator”](http://hibernate.org/validator/documentation/getting-started/) page has more details.



## **3. Simple Properties**

**The official documentation advises that we isolate configuration properties into separate POJOs.**

So let's start by doing that:

```java
@Configuration
@ConfigurationProperties(prefix = "mail")
public class ConfigProperties {
    
    private String hostName;
    private int port;
    private String from;

    // standard getters and setters
}
```

We use *@Configuration* so that Spring creates a Spring bean in the application context.

**@ConfigurationProperties works best with hierarchical properties that all have the same prefix;** therefore, we add a prefix of *mail*.

The Spring framework uses standard Java bean setters, so we must declare setters for each of the properties.

Note: If we don't use *@Configuration* in the POJO, then we need to add *@EnableConfigurationProperties(ConfigProperties.class)* in the main Spring application class to bind the properties into the POJO:

```java
@SpringBootApplication
@EnableConfigurationProperties(ConfigProperties.class)
public class EnableConfigurationDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(EnableConfigurationDemoApplication.class, args);
    }
}
```

That's it! **Spring will automatically bind any property defined in our property file that has the prefix mail and the same name as one of the fields in the ConfigProperties class**.

Spring uses some relaxed rules for binding properties. As a result, the following variations are all bound to the property *hostName*:

```shell
mail.hostName
mail.hostname
mail.host_name
mail.host-name
mail.HOST_NAME

```

Therefore, we can use the following properties file to set all the fields:

```shell
#Simple properties
mail.hostname=host@mail.com
mail.port=9000
mail.from=mailer@mail.com
```





### 3.1. Spring Boot 2.2

**As of Spring Boot 2.2, Spring finds and registers @ConfigurationProperties classes via classpath scanning**. Scanning of *@ConfigurationProperties* needs to be explicitly opted into by adding the **@ConfigurationPropertiesScan** annotation. Therefore, **we don't have to annotate such classes with @Component** *(and other meta-annotations like @Configuration),* **or even use the @EnableConfigurationProperties:**

```java
@ConfigurationProperties(prefix = "mail") 
@ConfigurationPropertiesScan 
public class ConfigProperties { 

    private String hostName; 
    private int port; 
    private String from; 

    // standard getters and setters 
}

```

The classpath scanner enabled by *@SpringBootApplication* finds the *ConfigProperties* class, even though we didn't annotate this class with *@Component.*

In addition, we can use **the @ConfigurationPropertiesScan annotation to scan custom locations for configuration property classes:**

```java
@SpringBootApplication
@ConfigurationPropertiesScan("com.baeldung.configurationproperties")
public class EnableConfigurationDemoApplication { 

    public static void main(String[] args) {   
        SpringApplication.run(EnableConfigurationDemoApplication.class, args); 
    } 
}
```

This way Spring will look for configuration property classes only in the *com.baeldung.properties* package.





## **4. Nested Properties** 

**We can have nested properties in Lists, Maps, and Classes.**

Let'S create a new *Credentials* class to use for some nested properties:

```java
public class Credentials {
    private String authMethod;
    private String username;
    private String password;

    // standard getters and setters
}
```

We also need to update the *ConfigProperties* class to use a *List,* a *Map*, and the *Credentials* class:

```java
public class ConfigProperties {

    private String hostname;
    private int port;
    private String from;
    private List<String> defaultRecipients;
    private Map<String, String> additionalHeaders;
    private Credentials credentials;
 
    // standard getters and setters
}
```

The following properties file will set all the fields:



```plaintext
#Simple properties
mail.hostname=mailer@mail.com
mail.port=9000
mail.from=mailer@mail.com

#List properties
mail.defaultRecipients[0]=admin@mail.com
mail.defaultRecipients[1]=owner@mail.com

#Map Properties
mail.additionalHeaders.redelivery=true
mail.additionalHeaders.secure=true

#Object properties
mail.credentials.username=john
mail.credentials.password=password
mail.credentials.authMethod=SHA1
```



## **5. Using @ConfigurationProperties on a @Bean Method**

**We can also use the @ConfigurationProperties annotation on @Bean-annotated methods.**

This approach may be particularly useful when we want to bind properties to a third-party component that's outside of our control.

Let's create a simple *Item* class that we'll use in the next example:

```java
public class Item {
    private String name;
    private int size;

    // standard getters and setters
}
```

Now let's see how we can use *@ConfigurationProperties* on a *@Bean* method to bind externalized properties to the *Item* instance:

```java
@Configuration
public class ConfigProperties {

    @Bean
    @ConfigurationProperties(prefix = "item")
    public Item item() {
        return new Item();
    }
}
```

Consequently, any item-prefixed property will be mapped to the *Item* instance managed by the Spring context.





## **6. Property Validation** 

**@ConfigurationProperties provides validation of properties using the JSR-380 format.** This allows all sorts of neat things.

For example, let's make the *hostName* property mandatory:

```java
@NotBlank
private String hostName;
@Length(max = 4, min = 1)
private String authMethod;
```

Then the *port* property from 1025 to 65536:

```java
@Min(1025)
@Max(65536)
private int port;
```

Finally, the *from* property must match an email address format:

```java
@Pattern(regexp = "^[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,6}$")
private String from;
```

This helps us reduce a lot of *if – else* conditions in our code, and makes it look much cleaner and more concise.

**If any of these validations fail, then the main application would fail to start with an IllegalStateException**.

The Hibernate Validation framework uses standard Java bean getters and setters, so it's important that we declare getters and setters for each of the properties.





## **7. Property Conversion**

*@ConfigurationProperties* supports conversion for multiple types of binding the properties to their corresponding beans.

### **7.1. Duration**

We'll start by looking at converting properties into *Duration* objects*.*

Here we have two fields of type *Duration*:



```java
@ConfigurationProperties(prefix = "conversion")
public class PropertyConversion {

    private Duration timeInDefaultUnit;
    private Duration timeInNano;
    ...
}
```

This is our properties file:

```bash
conversion.timeInDefaultUnit=10
conversion.timeInNano=9nsCopy
```

As a result, the field *timeInDefaultUnit* will have a value of 10 milliseconds, and *timeInNano* will have a value of 9 nanoseconds.

**The supported units are ns, us, ms, s, m, h and d for nanoseconds, microseconds, milliseconds, seconds, minutes, hours, and days, respectively.**

The default unit is milliseconds, which means if we don't specify a unit next to the numeric value, Spring will convert the value to milliseconds.

We can also override the default unit using *@DurationUnit:*

```java
@DurationUnit(ChronoUnit.DAYS)
private Duration timeInDays;Copy
```

This is the corresponding property:

```bash
conversion.timeInDays=2
```



### **7.2. DataSize**

**Similarly, Spring Boot @ConfigurationProperties supports DataSize type conversion.**

Let's add three fields of type *DataSize*:

```java
private DataSize sizeInDefaultUnit;

private DataSize sizeInGB;

@DataSizeUnit(DataUnit.TERABYTES)
private DataSize sizeInTB;
```

These are the corresponding properties:

```bash
conversion.sizeInDefaultUnit=300
conversion.sizeInGB=2GB
conversion.sizeInTB=4
```

**In this case, the sizeInDefaultUnit value will be 300 bytes, as the default unit is bytes.**

The supported units are *B, KB, MB, GB*, and *TB.* We can also override the default unit using *@DataSizeUnit.*





### **7.3. Custom Converter**

We can also add our own custom *Converter* to support converting a property to a specific class type.

Let's add a simple class *Employee*:

```java
public class Employee {
    private String name;
    private double salary;
}
```

Then we'll create a custom converter to convert this property:

```bash
conversion.employee=john,2000
```

We will convert it to a file of type *Employee*:

```java
private Employee employee;
```

We will need to implement the *Converter* interface, then **use @ConfigurationPropertiesBinding annotation to register our custom** ***Converter**:*

```java
@Component
@ConfigurationPropertiesBinding
public class EmployeeConverter implements Converter<String, Employee> {

    @Override
    public Employee convert(String from) {
        String[] data = from.split(",");
        return new Employee(data[0], Double.parseDouble(data[1]));
    }
}
```





## 8. Immutable *@ConfigurationProperties* Binding



As of Spring Boot 2.2, **we can use the @ConstructorBinding annotation to bind our configuration properties**.

This essentially means that *@ConfigurationProperties*-annotated classes may now be [immutable](https://www.baeldung.com/java-immutable-object).

But from Spring Boot 3, this annotation is not required:

```java
@ConfigurationProperties(prefix = "mail.credentials")
public class ImmutableCredentials {

    private final String authMethod;
    private final String username;
    private final String password;

    public ImmutableCredentials(String authMethod, String username, String password) {
        this.authMethod = authMethod;
        this.username = username;
        this.password = password;
    }

    public String getAuthMethod() {
        return authMethod;
    }

    public String getUsername() {
        return username;
    }

    public String getPassword() {
        return password;
    }
}
```

As we can see, when using *@ConstructorBinding,* we need to provide the constructor with all the parameters we'd like to bind.

Note that all the fields of *ImmutableCredentials* are final. Also, there are no setter methods.

Furthermore, it's important to emphasize that **to use the constructor binding, we need to explicitly enable our configuration class either with @EnableConfigurationProperties or with @ConfigurationPropertiesScan***.*





## 9. Java 16 *record*s



Java 16 introduced the *record* types as part of [JEP 395](https://openjdk.java.net/jeps/395). Records are classes that act as transparent carriers for immutable data. This makes them perfect candidates for configuration holders and DTOs. As a matter of fact, **we can define Java records as configuration properties in Spring Boot**. For instance, the previous example can be rewritten as:

```java
@ConstructorBinding
@ConfigurationProperties(prefix = "mail.credentials")
public record ImmutableCredentials(String authMethod, String username, String password) {
}
```

Obviously, it's more concise compared to all those noisy getters and setters.

Moreover, as of [Spring Boot 2.6](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6.0-M2-Release-Notes#records-and-configurationproperties), **for single-constructor records, we can drop the @ConstructorBinding annotation**. If our record has multiple constructors, however, *@ConstructorBinding* should still be used to identify the constructor to use for property binding.



## **10. Conclusion**

In this article, we explored the *@ConfigurationProperties* annotation and highlighted some of the useful features it provides, like relaxed binding and Bean Validation.

As usual, the code is available [over on Github](https://github.com/eugenp/tutorials/tree/master/spring-boot-modules/spring-boot-properties).









































