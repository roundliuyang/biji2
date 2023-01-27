# Mockito 入门@Mock、@Spy、@Captor 和@InjectMocks



## **1. Overview**



In this tutorial, we'll cover the following **annotations of the Mockito library:** *@Mock*, *@Spy*, *@Captor*, and *@InjectMocks*.

For more Mockito goodness, [have a look at the series here](https://www.baeldung.com/tag/mockito/).



## **2. Enable Mockito Annotations**



Before we go further, let's explore different ways to enable the use of annotations with Mockito tests.

### 2.1. *MockitoJUnitRunner*

The first option we have is to **annotate the JUnit test with a MockitoJUnitRunner**:

```java
@RunWith(MockitoJUnitRunner.class)
public class MockitoAnnotationTest {
    ...
}
```

### 2.2. *MockitoAnnotations.openMocks()*

Alternatively, we can **enable Mockito annotations programmatically** by invoking *MockitoAnnotations.openMocks()*:

```java
@Before
public void init() {
    MockitoAnnotations.openMocks(this);
}
```



### 2.3. *MockitoJUnit.rule()*

Lastly, **we can use a MockitoJUnit.rule()**:

```java
public class MockitoInitWithMockitoJUnitRuleUnitTest {

    @Rule
    public MockitoRule initRule = MockitoJUnit.rule();

    ...
}
```

In this case, we must remember to make our rule *public*.





## **3. @Mock Annotation**

The most widely used annotation in Mockito is *@Mock*. We can use *@Mock* to create and inject mocked instances without having to call *Mockito.mock* manually.

In the following example, we'll create a mocked *ArrayList* manually without using the *@Mock* annotation:

```java
@Test
public void whenNotUseMockAnnotation_thenCorrect() {
    List mockList = Mockito.mock(ArrayList.class);
    
    mockList.add("one");
    Mockito.verify(mockList).add("one");
    assertEquals(0, mockList.size());

    Mockito.when(mockList.size()).thenReturn(100);
    assertEquals(100, mockList.size());
}
```

Now we'll do the same, but we'll inject the mock using the *@Mock* annotation:

```java
@Mock
List<String> mockedList;

@Test
public void whenUseMockAnnotation_thenMockIsInjected() {
    mockedList.add("one");
    Mockito.verify(mockedList).add("one");
    assertEquals(0, mockedList.size());

    Mockito.when(mockedList.size()).thenReturn(100);
    assertEquals(100, mockedList.size());
}
```

Note how in both examples, we're interacting with the mock and verifying some of these interactions, just to make sure that the mock is behaving correctly.



## **4. @Spy Annotation**

Now let's see how to use the *@Spy* annotation to spy on an existing instance.

In the following example, we create a spy of a *List* without using the *@Spy* annotation:

```java
@Test
public void whenNotUseSpyAnnotation_thenCorrect() {
    List<String> spyList = Mockito.spy(new ArrayList<String>());
    
    spyList.add("one");
    spyList.add("two");

    Mockito.verify(spyList).add("one");
    Mockito.verify(spyList).add("two");

    assertEquals(2, spyList.size());

    Mockito.doReturn(100).when(spyList).size();
    assertEquals(100, spyList.size());
}
```

Now we'll do the same thing, spy on the list, but we'll use the *@Spy* annotation:

```java
@Spy
List<String> spiedList = new ArrayList<String>();

@Test
public void whenUseSpyAnnotation_thenSpyIsInjectedCorrectly() {
    spiedList.add("one");
    spiedList.add("two");

    Mockito.verify(spiedList).add("one");
    Mockito.verify(spiedList).add("two");

    assertEquals(2, spiedList.size());

    Mockito.doReturn(100).when(spiedList).size();
    assertEquals(100, spiedList.size());
}
```

Note how, as before, we're interacting with the spy here to make sure that it behaves correctly. In this example we:

- Used the **real** method *spiedList.add()* to add elements to the *spiedList*.
- **Stubbed** the method *spiedList.size()* to return *100* instead of *2* using *Mockito.doReturn()*.



## **5. @Captor Annotation**

Next let's see how to use the *@Captor* annotation to create an *ArgumentCaptor* instance.

In the following example, we'll create an *ArgumentCaptor* without using the *@Captor* annotation:

```java
@Test
public void whenNotUseCaptorAnnotation_thenCorrect() {
    List mockList = Mockito.mock(List.class);
    ArgumentCaptor<String> arg = ArgumentCaptor.forClass(String.class);

    mockList.add("one");
    Mockito.verify(mockList).add(arg.capture());

    assertEquals("one", arg.getValue());
}
```

Now let's **make use of @Captor** for the same purpose, to create an *ArgumentCaptor* instance:

```java
@Mock
List mockedList;

@Captor 
ArgumentCaptor argCaptor;

@Test
public void whenUseCaptorAnnotation_thenTheSame() {
    mockedList.add("one");
    Mockito.verify(mockedList).add(argCaptor.capture());

    assertEquals("one", argCaptor.getValue());
}
```

Notice how the test becomes simpler and more readable when we take out the configuration logic.



## **6. @InjectMocks Annotation**

Now let's discuss how to use the *@InjectMocks* annotation to inject mock fields into the tested object automatically.

In the following example, we'll use *@InjectMocks* to inject the mock *wordMap* into the *MyDictionary* *dic*:

```java
@Mock
Map<String, String> wordMap;

@InjectMocks
MyDictionary dic = new MyDictionary();

@Test
public void whenUseInjectMocksAnnotation_thenCorrect() {
    Mockito.when(wordMap.get("aWord")).thenReturn("aMeaning");

    assertEquals("aMeaning", dic.getMeaning("aWord"));
}
```

Here is the class *MyDictionary*:

```java
public class MyDictionary {
    Map<String, String> wordMap;

    public MyDictionary() {
        wordMap = new HashMap<String, String>();
    }
    public void add(final String word, final String meaning) {
        wordMap.put(word, meaning);
    }
    public String getMeaning(final String word) {
        return wordMap.get(word);
    }
}

```



## **7. Injecting a Mock Into a Spy**

Similar to the above test, we might want to inject a mock into a spy:

```java
@Mock
Map<String, String> wordMap;

@Spy
MyDictionary spyDic = new MyDictionary();
```

**However, Mockito doesn't support injecting mocks into spies,** and the following test results in an exception:

```java
@Test 
public void whenUseInjectMocksAnnotation_thenCorrect() { 
    Mockito.when(wordMap.get("aWord")).thenReturn("aMeaning"); 

    assertEquals("aMeaning", spyDic.getMeaning("aWord")); 
}
```

If we want to use a mock with a spy, we can manually inject the mock through a constructor:

```java
MyDictionary(Map<String, String> wordMap) {
    this.wordMap = wordMap;
}
```

Instead of using the annotation, we can now create the spy manually:

```java
@Mock
Map<String, String> wordMap; 

MyDictionary spyDic;

@Before
public void init() {
    MockitoAnnotations.openMocks(this);
    spyDic = Mockito.spy(new MyDictionary(wordMap));
}

```

The test will now pass.



## **8. Running Into NPE While Using Annotation** 

Often we may **run into NullPointerException** when we try to actually use the instance annotated with *@Mock* or *@Spy*:

```java
public class MockitoAnnotationsUninitializedUnitTest {

    @Mock
    List<String> mockedList;

    @Test(expected = NullPointerException.class)
    public void whenMockitoAnnotationsUninitialized_thenNPEThrown() {
        Mockito.when(mockedList.size()).thenReturn(1);
    }
}
```

Most of the time, this happens simply because we forget to properly enable Mockito annotations.

So we have to keep in mind that each time we want to use Mockito annotations, we must take the extra step and initialize them as we already explained [earlier](https://www.baeldung.com/mockito-annotations#enable-annotations).



## **9. Notes**

Finally, here are **some notes** about Mockito annotations:

- Mockito's annotations minimize repetitive mock creation code.
- They make tests more readable.
- *@InjectMocks* is necessary for injecting both *@Spy* and *@Mock* instances.



## **10. Conclusion**



In this brief article, we explained the basics of **annotations in the Mockito library**.

The implementation of all of these examples can be found [over on GitHub](https://github.com/eugenp/tutorials/tree/master/testing-modules/mockito-simple). This is a Maven project, so it should be easy to import and run as it is.

Of course, for more Mockito goodness, [have a look at the series here](https://www.baeldung.com/tag/mockito/).



