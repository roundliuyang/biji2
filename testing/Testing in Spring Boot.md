# Testing in Spring Boot



## **1. Overview**



In this tutorial, we'll have a look at **writing tests using the framework support in Spring Boot.** We'll cover unit tests that can run in isolation as well as integration tests that will bootstrap Spring context before executing tests.

If you are new to Spring Boot, check out our [intro to Spring Boot](https://www.baeldung.com/spring-boot-start).



## **2. Project Setup**



The application we're going to use in this article is an API that provides some basic operations on an *Employee* Resource. This is a typical tiered architecture — the API call is processed from the *Controller* to *Service* to the *Persistence* layer.



## **3. Maven Dependencies**

Let's first add our testing dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <version>2.5.0</version>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

The [*spring-boot-starter-test*](https://search.maven.org/classic/#search|gav|1|g%3A"org.springframework.boot" AND a%3A"spring-boot-starter-test") is the primary dependency that contains the majority of elements required for our tests.

The [H2 DB](https://search.maven.org/classic/#search|gav|1|g%3A"com.h2database" AND a%3A"h2") is our in-memory database. It eliminates the need for configuring and starting an actual database for test purposes.



### 3.1. JUnit 4

Starting with Spring Boot 2.4, JUnit 5’s vintage engine has been removed from *spring-boot-starter-test*. If we still want to write tests using JUnit 4, we need to add the following Maven dependency:

```xml
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
    <exclusions>
        <exclusion>
            <groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```



## **4. Integration Testing With @SpringBootTest**



As the name suggests, integration tests focus on integrating different layers of the application. That also means no mocking is involved.

**Ideally, we should keep the integration tests separated from the unit tests and should not run along with the unit tests.** We can do this by using a different profile to only run the integration tests. A couple of reasons for doing this could be that the integration tests are time-consuming and might need an actual database to execute.

However in this article, we won't focus on that, and we'll instead make use of the in-memory H2 persistence storage.

The integration tests need to start up a container to execute the test cases. Hence, some additional setup is required for this — all of this is easy in Spring Boot:

```java
@RunWith(SpringRunner.class)
@SpringBootTest(
  webEnvironment = SpringBootTest.WebEnvironment.MOCK,
  classes = Application.class)
@AutoConfigureMockMvc
@TestPropertySource(
  locations = "classpath:application-integrationtest.properties")
public class EmployeeRestControllerIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @Autowired
    private EmployeeRepository repository;

    // write test cases here
}
```

**The @SpringBootTest annotation is useful when we need to bootstrap the entire container.** The annotation works by creating the *ApplicationContext* that will be utilized in our tests.

We can use the *webEnvironment* attribute of *@SpringBootTest* to configure our runtime environment; we're using *WebEnvironment.MOCK* here so that the container will operate in a mock servlet environment.

Next, the *@TestPropertySource* annotation helps configure the locations of properties files specific to our tests. Note that the property file loaded with *@TestPropertySource* will override the existing *application.properties* file.

The *application-integrationtest.properties* contains the details to configure the persistence storage:

>spring.datasource.url = jdbc:h2:mem:test
>spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.H2Dialect

If we want to run our integration tests against MySQL, we can change the above values in the properties file.

The test cases for the integration tests might look similar to the *Controller* layer unit tests:

```java
@Test
public void givenEmployees_whenGetEmployees_thenStatus200()
  throws Exception {

    createTestEmployee("bob");

    mvc.perform(get("/api/employees")
      .contentType(MediaType.APPLICATION_JSON))
      .andExpect(status().isOk())
      .andExpect(content()
      .contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
      .andExpect(jsonPath("$[0].name", is("bob")));
}
```

The difference from the *Controller* layer unit tests is that here nothing is mocked and end-to-end scenarios will be executed.



## 5. Test Configuration With *@TestConfiguration*



As we've seen in the previous section, a test annotated with *@SpringBootTest* will bootstrap the full application context, which means we can *@Autowire* any bean that's picked up by component scanning into our test:

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class EmployeeServiceImplIntegrationTest {

    @Autowired
    private EmployeeService employeeService;

    // class code ...
}

```

However, we might want to avoid bootstrapping the real application context but use a special test configuration. We can achieve this with the *@TestConfiguration* annotation. There are two ways of using the annotation. Either on a static inner class in the same test class where we want to *@Autowire* the bean:

```java
@RunWith(SpringRunner.class)
public class EmployeeServiceImplIntegrationTest {

    @TestConfiguration
    static class EmployeeServiceImplTestContextConfiguration {
        @Bean
        public EmployeeService employeeService() {
            return new EmployeeService() {
                // implement methods
            };
        }
    }

    @Autowired
    private EmployeeService employeeService;
}
```

Alternatively, we can create a separate test configuration class:

```java
@TestConfiguration
public class EmployeeServiceImplTestContextConfiguration {
    
    @Bean
    public EmployeeService employeeService() {
        return new EmployeeService() { 
            // implement methods 
        };
    }
}
```

Configuration classes annotated with *@TestConfiguration* are excluded from component scanning, therefore we need to import it explicitly in every test where we want to *@Autowire* it. We can do that with the *@Import* annotation:

```java
@RunWith(SpringRunner.class)
@Import(EmployeeServiceImplTestContextConfiguration.class)
public class EmployeeServiceImplIntegrationTest {

    @Autowired
    private EmployeeService employeeService;

    // remaining class code
}
```





## **6. Mocking With @MockBean**

Our *Service* layer code is dependent on our *Repository:*

```java
@Service
public class EmployeeServiceImpl implements EmployeeService {

    @Autowired
    private EmployeeRepository employeeRepository;

    @Override
    public Employee getEmployeeByName(String name) {
        return employeeRepository.findByName(name);
    }
}
```

However, to test the *Service* layer, we don't need to know or care about how the persistence layer is implemented. Ideally, we should be able to write and test our *Service* layer code without wiring in our full persistence layer.

To achieve this, **we can use the mocking support provided by Spring Boot Test.**

Let's have a look at the test class skeleton first:

```java
@RunWith(SpringRunner.class)
public class EmployeeServiceImplIntegrationTest {

    @TestConfiguration
    static class EmployeeServiceImplTestContextConfiguration {
 
        @Bean
        public EmployeeService employeeService() {
            return new EmployeeServiceImpl();
        }
    }

    @Autowired
    private EmployeeService employeeService;

    @MockBean
    private EmployeeRepository employeeRepository;

    // write test cases here
}
```

To check the *Service* class, we need to have an instance of the *Service* class created and available as a *@Bean* so that we can *@Autowire* it in our test class. We can achieve this configuration using the *@TestConfiguration* annotation.

Another interesting thing here is the use of *@MockBean*. It [creates a Mock](https://www.baeldung.com/mockito-mock-methods) for the *EmployeeRepository*, which can be used to bypass the call to the actual *EmployeeRepository*:

```java
@Before
public void setUp() {
    Employee alex = new Employee("alex");

    Mockito.when(employeeRepository.findByName(alex.getName()))
      .thenReturn(alex);
}
```

Since the setup is done, the test case will be simpler:

```java
@Test
public void whenValidName_thenEmployeeShouldBeFound() {
    String name = "alex";
    Employee found = employeeService.getEmployeeByName(name);
 
     assertThat(found.getName())
      .isEqualTo(name);
 }
```



## **7. Integration Testing With @DataJpaTest**

We're going to work with an entity named *Employee,* which has an *id* and a *name* as its properties:

```java
@Entity
@Table(name = "person")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Size(min = 3, max = 20)
    private String name;

    // standard getters and setters, constructors
}
```

And here's our repository using Spring Data JPA:

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    public Employee findByName(String name);

}
```

That's it for the persistence layer code. Now let's head toward writing our test class.

First, let's create the skeleton of our test class:

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class EmployeeRepositoryIntegrationTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private EmployeeRepository employeeRepository;

    // write test cases here

}
```

*@RunWith(SpringRunner.class)* provides a bridge between Spring Boot test features and JUnit. Whenever we are using any Spring Boot testing features in our JUnit tests, this annotation will be required.

*@DataJpaTest* provides some standard setup needed for testing the persistence layer:

- configuring H2, an in-memory database
- setting Hibernate, Spring Data, and the *DataSource*
- performing an *@EntityScan*
- turning on SQL logging

To carry out DB operations, we need some records already in our database. To setup this data, we can use *TestEntityManager.*

**The Spring Boot TestEntityManager is an alternative to the standard JPA EntityManager that provides methods commonly used when writing tests.**

*EmployeeRepository* is the component that we are going to test.

Now let's write our first test case:

```java
@Test
public void whenFindByName_thenReturnEmployee() {
    // given
    Employee alex = new Employee("alex");
    entityManager.persist(alex);
    entityManager.flush();

    // when
    Employee found = employeeRepository.findByName(alex.getName());

    // then
    assertThat(found.getName())
      .isEqualTo(alex.getName());
}
```

In the above test, we're using the *TestEntityManager* to insert an *Employee* in the DB and reading it via the find by name API.

The *assertThat(…)* part comes from the [Assertj library](https://www.baeldung.com/introduction-to-assertj), which comes bundled with Spring Boot.





## **8. Unit Testing With @WebMvcTest**

Our *Controller* depends on the *Service* layer; let's only include a single method for simplicity:

```java
@RestController
@RequestMapping("/api")
public class EmployeeRestController {

    @Autowired
    private EmployeeService employeeService;

    @GetMapping("/employees")
    public List<Employee> getAllEmployees() {
        return employeeService.getAllEmployees();
    }
}
```

Since we're only focused on the *Controller* code, it's natural to mock the *Service* layer code for our unit tests:

```java
@RunWith(SpringRunner.class)
@WebMvcTest(EmployeeRestController.class)
public class EmployeeRestControllerIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private EmployeeService service;

    // write test cases here
}
```

**To test the Controllers, we can use @WebMvcTest. It will auto-configure the Spring MVC infrastructure for our unit tests.**

In most cases, @*WebMvcTest* will be limited to bootstrap a single controller. We can also use it along with *@MockBean* to provide mock implementations for any required dependencies.

*@WebMvcTest* also auto-configures *MockMvc*, which offers a powerful way of easy testing MVC controllers without starting a full HTTP server.

Having said that, let's write our test case:

```java
@Test
public void givenEmployees_whenGetEmployees_thenReturnJsonArray()
  throws Exception {
    
    Employee alex = new Employee("alex");

    List<Employee> allEmployees = Arrays.asList(alex);

    given(service.getAllEmployees()).willReturn(allEmployees);

    mvc.perform(get("/api/employees")
      .contentType(MediaType.APPLICATION_JSON))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$", hasSize(1)))
      .andExpect(jsonPath("$[0].name", is(alex.getName())));
}
```

The *get(…)* method call can be replaced by other methods corresponding to HTTP verbs like *put()*, *post()*, etc. Please note that we are also setting the content type in the request.

*MockMvc* is flexible, and we can create any request using it.



## 9. Auto-Configured Tests

One of the amazing features of Spring Boot's auto-configured annotations is that it helps to load parts of the complete application and test-specific layers of the codebase.

In addition to the above-mentioned annotations, here's a list of a few widely used annotations:

- *@WebF**luxTest*: We can use the *@WebFluxTest* annotation to test Spring WebFlux controllers. It's often used along with *@MockBean* to provide mock implementations for required dependencies.

- *@JdbcTest*: *W*e can use the *@JdbcTest* annotation to test JPA applications, but it's for tests that only require a *DataSource.* The annotation configures an in-memory embedded database and a *JdbcTemplate.*

- *@JooqTest*: To test jOOQ-related tests, we can use *@JooqTest* annotation, which configures a DSLContext.

- *@DataMongoTest*: To test MongoDB applications, *@DataMongoTest* is a useful annotation. By default, it configures an in-memory embedded MongoDB if the driver is available through dependencies, configures a *MongoTemplate,* scans for *@Document* classes, and configures Spring Data MongoDB repositories.

- *@DataRedisTest*makes it easier to test Redis applications. It scans for *@RedisHash* classes and configures Spring Data Redis repositories by default.
- *@DataLdapTest* configures an in-memory embedded *LDAP* (if available), configures a *LdapTemplate*, scans for *@Entry* classes, and configures Spring Data *LDAP* repositories by default.

- *@RestClientTest*: We generally use the *@RestClientTest* annotation to test REST clients. It auto-configures different dependencies such as Jackson, GSON, and Jsonb support; configures a *RestTemplateBuilder;* and adds support for *MockRestServiceServer* by default.
- *@JsonTest*: Initializes the Spring application context only with those beans needed to test JSON serialization.

You can read more about these annotations and how to further optimize integrations tests in our article [Optimizing Spring Integration Tests](https://www.baeldung.com/spring-tests).







