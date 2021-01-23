---
layout: post
title: HttpMessageConverter
data: 2021-01-23 21:04:00
categories: spring
permalink: /spring/http-message-converter
tags: spring  
author: kimjongmo
comments: true
---



## 시뮬레이션



## 프롤로그.

HttpMessageConverter에 대하여 알아보기 전 간단한 예시를 보도록 하자.

아래와 같은 스펙의 API를 만들어달라는 요청을 받게되었다는 가정하에 코드를 만들어봤다.

- API는 POST여야 한다.

- API의 Content-Type, Accept은 text/plain 이다.

- 전달되는 데이터는 문자열이며 쉼표로 각 데이터를 분리한다. 

  ```
  이름,나이
  ```

- 응답으로는 위의 요청데이터를 다시 내뱉어주는걸로 하자.



> 프로젝트는 스프링부트를 이용하여 만들었고 디펜던시는 아래와 같이 설정하였다.
>
> ```groovy
> implementation 'org.springframework.boot:spring-boot-starter-web'
> compileOnly 'org.projectlombok:lombok'
> annotationProcessor 'org.projectlombok:lombok'
> testImplementation 'org.springframework.boot:spring-boot-starter-test'
> ```



아래의 코드가 이번 포스팅에서 사용할 엔드포인트이다.  스프링부트에서는 기본적으로 `application/json` 형식의 컨텐츠 타입만 지원하도록 디폴트되어있기 때문에 아래와 같이 `consumes`, `produces` 를 설정해주었다.

```java
@RestController
@Slf4j
public class TestController {
    // POST + text/plain
    @PostMapping(value = "/test/text", consumes = MediaType.TEXT_PLAIN_VALUE, produces = MediaType.TEXT_PLAIN_VALUE)
    public ResponseEntity<String> textTest(@RequestBody String data){
        Person person = Person.deserialize(data);
        log.info("person = {}",person);
        return ResponseEntity.ok(person.serialize());
    }
    
}


// other package
@Getter
@Setter
@ToString
@AllArgsConstructor
public class Person {
    private String name;
    private Integer age;

    public Person deserialize(String data) {
        String[] split = data.split(",");
        return new Person(split[0], Integer.parseInt(split[1]));
    }

    public String serialize() {
        return name+","+age;
    }
}
```



먼저 외부에서 해당 API를 요청해보도록 한다. (난 Postman 사용함)

```
POST /test/text HTTP/1.1
Host: 
Content-Type: text/plain
Accept: text/plain
Cache-Control: no-cache

mojong,29
```



이렇게 요청을 하면 일단 아래와 같은 응답을 받을 수 있다.

```
mojong,29
```



그런데 코드를 보다보니 파라미터 타입을 굳이 String으로 받아서 이걸 다시 Person 객체로 넘겨서 생성시키는 모습이 굉장히 구리다. 

![박명수_꼴보기싫어](/img/humorous/박명수_꼴보기싫어.jpg)

좀 더 내가 파라미터로 받아오는 이 데이터가 무슨 데이터인지 눈에 들어왔으면 좋겠다는 생각을 했다 마치 아래와 같이 말이다.

```java
public ResponseEntity<String> textTest(@RequestBody Person person)
```

만약 컨텐츠타입이 `application/json` 이었다면 위와 같이 적어도 자동으로 매핑을 시켜줬겠지만 현재 시뮬레이션의 컨텐츠타입은 `text/plain` 이기 때문에 위와 같이 적어준다해서 동작하지 않는다.  스프링에게 `text/plain` 컨텐츠타입 들어온 문자열 데이터를 `Person`으로 매핑을 하려면 어떻게 해주어야 하는지 알려주어야 한다.



## HttpMessageConverter

스프링은 `@RequestBody` 애노테이션이나 `@ResponseBody` 애노테이션이 붙어있으면 이 받은 데이터를 HttpMessageConverter를 이용해서 변환시킨다. 

스프링부트(정확히는 spring-boot-starter-web 프로젝트)는 기본적으로 어플리케이션 실행을 하면서 여러가지 HttpMessageConverter 구현체를 미리 등록해준다. 그 중 하나인 `MappingJacksonHttpMessageConverter`는 위에서 언급한 json <-> 객체 사이의 매핑을 담당해주고 있는데 이 덕분에 우리는 json 형식으로 데이터를 보냈더라도 이를 객체로 매핑시켜서 가져올 수 있다.

```java
// 예시
// POST + application/json
@PostMapping(value = "/test/text")
public ResponseEntity<String> textTest(@RequestBody Person person) {
    log.info("person = {}",person);
    return ResponseEntity.ok(person.toString());
}
```



그렇다면 우리가 프롤로그에서 언급했듯이 `text/plain` 데이터를 `Person` 객체로 매핑하기 위해서는 이러한 `HttpMessageConverter`를 만들어서 스프링에 등록을 해주어야한다.



## PersonHttpMessageConverter 생성해서 등록하기

이제 그러면 스프링에게 그냥 문자열을 어떻게 Person으로 매핑시키는지 알려주기 위해 아래와 같이 HttpMessageConverter를 만들어주도록 하자.

```java
@Component
public class PersonHttpMessageConverter extends AbstractHttpMessageConverter<Person> {

    public PersonHttpMessageConverter() {
        // 디폴트 Charset = UTF-8, 컨텐츠 타입은 text/plain 쌉가능!
        super(Charset.forName("UTF-8"), MediaType.TEXT_PLAIN);
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        // Person 객체 ?? 내가 서포트 하마.
        return clazz == Person.class;
    }
    
    @Override
    protected Person readInternal(Class<? extends Person> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        
        // Body 데이터를 default Charset (UTF-8)로 읽어들인다.
        String body = StreamUtils.copyToString(inputMessage.getBody(), getDefaultCharset());
        
        // 문자열을 -> Person 객체로 변환
        return Person.deserialize(body);
    }

    /**
     * application -> external
     * */
    @Override
    protected void writeInternal(Person person, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        // person 객체를 Http Body에 세팅을 할 때
        StreamUtils.copy(person.serialize(), getDefaultCharset(), outputMessage.getBody());
    }
}
```

 

스프링부트에서는 Converter를 등록하기 위해 특별한 동작을 하지 않아도 된다. `@Component`를 붙여 생성만 시켜주면 자동으로 위의 클래스 타입이 Converter라는 것을 알아채고 다른 `HttpMessageConverter`를 등록할 때 자동으로 등록시켜준다.  

이렇게 Converter 를 등록완료했다면 이제 컨트롤러의 코드를 아래와 같이 바꾸고 다시 외부에서 요청을 해보도록 하자.

```java
// POST + text/plain
@PostMapping(value = "/test/text", consumes = MediaType.TEXT_PLAIN_VALUE, produces = MediaType.TEXT_PLAIN_VALUE)
public ResponseEntity<Person> textTest(@RequestBody Person person){
    log.info("person = {}",person);
    return ResponseEntity.ok(person);
}
```



![유재석_흡족](/img/humorous/유재석_흡족.jpg)



## 마치며

별것 아닌 내용이지만 이런 작은 발상하나로 시작하여 구현을 해냈을 때 참 보람을 느낀다. 개발을 하면서도 익숙한 방식으로만 코딩하지 말고 이러한 작은 발상들을 꾸준하게 해나가며 좋은 코드를 만들어가고 싶다.