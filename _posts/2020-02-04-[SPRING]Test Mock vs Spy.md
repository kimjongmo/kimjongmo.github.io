---
layout: post
title: Mock vs Spy
data: 2020-02-04 19:20:00
categories: spring
permalink: /spring/mock-vs-spy
tags: spring mockito
author: kimjongmo
---



# Mock vs Spy

기존의 테스트를 진행했을 때 나는 아래와 같은 방식으로 테스트하였다.

- 테스트하려는 주체(FileUploadService)를 실제로 생성
- 해당 객체가 의존하고 있는 객체(CategoryRepository)는 Mock으로 생성

```java
public class FileUploadService{
    private CategoryRepository categoryRepository;
    
    public FileUploadService(CategoryRepository categoryRepository){
        this.categoryRepository = categoryRepository;
    }
    
    // Some method ..
    public String someMethod(String someMessage){
        if(!check(someMessage))
            return "FAIL";
        if(!request(someMesage).getStatus().equals("SUCCESS"))
            return "FAIL";
        return "SUCCESS";
    }
    
    public boolean check(String someMessage){
        ...
    }
    
    public Header request(String someMessage){
        ...
    }
}
```

```java
public class FileUploadServiceTest{
    private FileUploadService fileUploadService;
    @Mock
    private CategoryRepository categoryRespository;
    
    @Before
    public void setUp(){
        MockitoAnnotations.initMocks(this);
        fileUploadService = new FileUploadService(categoryRepository);
    }
    
    // Some Test..
}
```

 

FileUploadService안에서는 someMethod()와 같이 내부의 다른 메서드를 요청하는 부분 등이 있는데  이 메소드를 실제로 실행 시킬 필요는 없기에 해당 메서드를 stubbing을 하면 된다.

하지만 stubbing을 하기위해서 mockito의 given()이나 when()을 사용하려고 했지만 요구하는 객체가 모의 객체이므로 실제 객체를 모의 객체로 만들필요가 있었다.

Mockito를 이용하여 모의 객체를 만들기 위한 방법들 중에는 mock()을 사용하는 방법과 spy()를 사용하는 방법이 있다.

아래에서 spy와 mock의 차이를 알고 사용을 하자..

## Spy

Mockito.spy()는 모의 객체에서 실제로 메소드 호출 이루어지고 이 행동에 대한 결과가 반영이 된다는 특징이 있다. 그리고 모의 객체에 대한 Stubbing 또한 가능하다.

```java
@Test
public void SPY_SAMPLE_TEST(){
    List<Integer> list = new ArrayList<Integer>();
    List<Integer> mockList = Mockito.spy(list);

    mockList.add(100);
    assertEquals(mockList.size(),1); //PASS

    doReturn(5).when(mockList).size(); //Stubbing
    assertEquals(mockList.size(),5); //PASS

    mockList.add(200);
    assertEquals(mockList.size(),5); //PASS

    verify(mockList,times(2)).add(anyInt());
}
```



## Mock

Mockito.mock()로 만들어진 모의 객체는 spy()와는 다르게 작동한다.

```java
public void MOCK_SAMPLE_TEST(){
    List<Integer> mockList = Mockito.mock(ArrayList.class);

    mockList.add(100);
    assertEquals(mockList.size(),0); //PASS

    doReturn(5).when(mockList).size(); //Stubbing
    assertEquals(mockList.size(),5); //PASS

    mockList.add(200);
    assertEquals(mockList.size(),5); //PASS

    verify(mockList,times(2)).add(anyInt());//PASS
}
```

호출은 정확하게 일어나고 있지만 결과에 대한 반영은 되지 않는 다는 것을 알 수 있다.



