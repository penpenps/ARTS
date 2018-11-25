# Basic Annotations in Mockito
This is an article to introduct the usage of `@Mock`, `@Spy`, `@Captor` and `@InjectMocks` basic annotations in Mockito framework, a unit test framework in Java. If you never heard Mockito before, it's recommended to take a look at its offical website: [https://site.mockito.org/](https://site.mockito.org/).

## What's Mockito?
It's a popular framework for providing mocked variables and methods, flexibly stubbing methods and the return, throw expected exceptions, etc. From my understanding, as Java applications or programs often embeded with multiple factories, interfaces and class implements and there always are complex reference between classes, it is reasonable to decouple the callstack. Mockito allow us to focus on testing the feature we work on and the result from calling other classes or packages can be mocked as expected without really call it. 

For example, if we want to test the `add` method of `ArrayList` but don't want to create a `ArrayList` object in reality because there will be initialization about this class and this part is not included in our unit test. With Mockito we can write a UT to test that feature like that:
```java
@Test
public void testArrayAddElement(){
    List mockList = Mockito.mock(ArrayList.class);

    mockList.add("one");
    Mockito.verify(mockList).add("one");
    assertEquals(0, mockList.size());
}
```

This snip is to test `add` method and verify it is called successfully without any exception. We observe the last line, `mockList` length is `0` that is `Mockit.mock` key function, it mocked a list and execute the `add` feature but not actually new a list and insert the element. It only test the `add` function interactivly and verify its behavior is correct.

This is a typical Mockito usage and there are more complicated scenarioes.

## `@Mock`
Based on the above example, what if we want to use `mockList` in serveral test cases? Create it in each case? Absolutely, it's unnecessary. Annotation `@Mock` can help use like that:
```java
@Mock
private List mockList;

@Test
public void testWithMockAnnotation(){
    mockList.add("one");
    Mockito.verify(mockList).add("one");
    assertEquals(0, mockList.size());
}
```
It is equal to the above example with annotation instead of initialize the mock list.

## `@Spy`
`@Mock` won't initialize a real list for us and just verify `add` behavior so what if we want a list with elements and actually exists in memory?

At this case, we can use `@Spy`, let's expand the example:
```java
@Spy
private List mockList = new ArrayList<String>();

@Test
public void testWithSpyAnnotation(){
    mockList.add("one");
    mockList.add("two");
    Mockito.verify(mockList).add("one");
    Mockito.verify(mockList).add("two");
    assertEquals(2, mockList.size());

    Mockito.when(mockList.size()).thenReturn(100);
    assertEquals(100, mockList.size());
}
```
What worth to notice is that when use `@Spy`, it is expected to initialize the annotated object because it will be executed as a real list, this is different from `@Mock`. Without initialize, it will throw a `MockitoException` like that:
```
org.mockito.exceptions.base.MockitoException: Unable to initialize @Spy annotated field 'mockList'.
Type 'List' is an interface and it cannot be spied on.
```
Look at the first assert statement, it is expected there are 2 elements in `mockList` but it is also can be stubbed with an expected callback just like the following two lines to set the return as 100.

## `@Captor`
Sometimes, except for verify the behavior of the test method, we also want to verify more specific about the execution, like whether the method accept expected arguments. To feed this requirement, we can use `@Captor` to capture the argument passed into the verified method, in our go through example, it will be used like:
```java
@Spy
private List mockList = new ArrayList<String>();

@Captor
ArgumentCaptor argCaptor;
@Test
public void testCaptorAnnotation(){
    mockList.add("one");
    Mockito.verify(mockList).add(argCaptor.capture());
    assertEquals("one", argCaptor.getValue());
}
```

## `@InjectMocks`
If we want to inject a mocked object into a class and thest other methods in this class, we can use `InjectMocks` to do that. Assume we have a customized class `MyDictionary` and its definition like:
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
        // do something here
        System.out.println("Processing the request to lookup word: " + word);
        return wordMap.get(word);
    }
}
```
Now, we want to test `getMeaning` method and verify the logic inside of it but don't to handle initialization about `wordMap` so we use a mocked `wordMap`:
```java
@Mock
private Map wordMap;

@InjectMocks
private MyDictionary myDictionary;

@Test
public void testInjectMocksAnnotation(){
    Mockito.when(wordMap.get("aWord")).thenReturn("aMeaning");

    assertEquals("aMeaning", myDictionary.getMeaning("aWord"));
}
```
`@InjectMocks` injects `wordMap` into `myDictionary` and the operations on `wordMap` in `myDictionary` will use the injected `wordMap`. Notice that if there are multiple `Map` objects with `@Mock` in the UT, Mockit will choose the one has the same name as inside variable. In our example, if the declaration in UT like following:
```java
@Mock
private Map wordMap;

@Mock
private Map anotherMap;

@InjectMocks
private MyDictionary myDictionary;
```
It will automatically use `wordMap` instead of `anotherMap`. However, if there is only one `Map` with `@Mock` anotation, whatever the name of this variable, `@InjectMocks` will use it as its `wordMap`.

Moveover, `@InjectMocks` works both for `@Mock` and `@Spy`.

## Note
In order to enable those annotations in Mockit framework, we should init the annotations before all the tests:
```
@BeforeClass(alwaysRun=true)
public void beforeTest(){
    MockitoAnnotations.initMocks(this);
}
```
Otherwise, it will throw exception like:
```
org.mockito.exceptions.misusing.MissingMethodInvocationException: 
when() requires an argument which has to be 'a method call on a mock'.
For example:
    when(mock.getArticles()).thenReturn(articles);

Also, this error might show up because:
1. you stub either of: final/private/equals()/hashCode() methods.
   Those methods *cannot* be stubbed/verified.
   Mocking methods declared on non-public parent classes is not supported.
2. inside when() you don't call method on mock but on some other object.
```
