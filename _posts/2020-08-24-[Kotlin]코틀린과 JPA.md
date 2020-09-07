---
layout: post
title: kotlin with jpa
data: 2020-08-24 21:59:00
categories: kotlin
permalink: /kotlin/kotlin-with-jpa
tags: spring-boot kotlin jpa
author: kimjongmo

---



## 서버 사이드 개발 - 코틀린

### 몇가지 특징..

- JVM 위에서 동작
- 자바와 완전한 호환성
- 간결한 코드
- null 체크를 강하게 한다

> 그만알아보자



### 컴파일

`.kt` 파일은 컴파일하게 되면 Java의 Bytecode가 된다. 즉 코틀린으로 작성한 문법들이 다시 자바로 변환하게 된다.

intellij 에서는 `Tools` - `Kotlin` - `Show Kotlin Bytecode` - `Decompile` 을 통해서 해당 코틀린 파일이 어떻게 자바로 변환되는지 볼수있다.

이것을 하다보면 알 수 있는 점중 하나는 코틀린으로 선언한 대부분이 final이 붙는다는 것이다.

예를 들어 아래의 코드는

```kotlin
val name: String
```

자바로 컴파일하면 아래와 같이 표현된다

```java
final String name;
```



## JPA의 지연로딩

jpa를 써본 사람들은 지연 로딩이라는 특징에 대해서 알고 있을 것이다. jpa가 데이터베이스로부터 데이터를 받아와 객체로 매핑을 시켜줄 때 지연로딩으로 만든 객체에 대해서는 임시로 <u>프록시 객체</u>를 넣어두고 실제로 이 <u>프록시 객체가 사용되는 시점에 다시한번 쿼리를 날려 데이터를 세팅</u>해준다는 것이다. 

예를 들어 Book 테이블과 Author 테이블이 1:N 관계를 가지고 있다고 했을 때

```kotlin
@Entity
class Book (
	@Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  var id: Int,
  @OneToMany(mappedBy= "book")
  var authors: List<Author>
)

@Entity
class Author (
	@Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  var id: Int,
  var name: String
  @ManyToOne(fetch = FetchType.LAZY)
  var book: Book
)
```

Jpa를 이용해 Book을 조회하면 author가 곧바로 세팅이 되는것이 아니라 프록시 객체가 세팅이 된다는 것이다.

여기서 주의해야 할 점은 바로 <u>프록시 객체를 세팅한다는 것이다</u>. 프록시 객체는 대체하려는 객체에 대해 상속을 받아야 하는데 예를 들면 위에 코드에서 Author라는 클래스에 대해서 Proxy가 만들어지면 대충 아래와 같은 느낌이 된다. 상속을 받은 프록시 객체는 그리하여 해당 객체를 대신할 수 있게 된다.

```java
class AuthtorProxy extends Author {
  ...
}
```



<u>자 그럼 이제부터 코틀린이 컴파일 할 때 특징과 JPA의 지연로딩이 어떤 상관이 있는지 살펴보자</u>

먼저 위에 코틀린은 별다른 선언을 하지 않는다면 자바로 변환될 때 기본적으로 final 이 붙는다고 하였다. 그렇다면 위의 예시로 만든 Author 클래스는 아래와 같이 final이 붙은 형태로 컴파일이 될것이다.

```java
final class Author {
  ...
}
```

문법이지만.. 클래스에 final이 붙게되면 자바에서는 더이상 상속을 하지 않겠다는 것을 의미한다.

"네??? 그런데 JPA의 지연로딩을 이용하려면 상속을 받아야 하는데요??"

실제로 위와 같이 만들고 테스트를 해보면 예외가 발생하면서 프록시 객체가 세팅되는 것이 아니라 실제 쿼리를 한번더 날려서 객체를 즉각 세팅을 하게 된다.

따라서 만약 내가 코틀린에서 JPA의 지연 로딩을 사용하고 싶다면 아래와 같이 앞에 `open` 키워드를 붙여서 오버라이딩이 가능하게 해주어야 한다.

```kotlin
@Entity
open class Book (
	@Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  open var id: Int,
  @OneToMany(mappedBy= "book")
  open var authors: List<Author>
)

@Entity
open class Author (
	@Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  open var id: Int,
  open var name: String
  @ManyToOne(fetch = FetchType.LAZY)
  open var book: Book
)
```



## JPA는 인자 없는 생성자가 필요합니다 

JPA 인자 문제 있어???

JPA를 사용하기 위해서는 기본적으로 디폴트 생성자가 필요하게 됩니다. 고로 위의 코틀린 코드를 다시 수정하면 아래와 같은 모양이 되어야 합니다.

```kotlin
@Entity
open class Book {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  open var id: Int
  
  @OneToMany(mappedBy= "book")
  open var authors: List<Author>
}
```



## 플러그인 사용하기

이렇게 사이가 안좋아보이는 JPA와 코틀린 이 둘사이를 돈독하게 해주기 위해 아래의 플러그인들이 존재하고 있습니다.

- all-open
- no-args

all-open 플러그인은 특정한 annotation을 지정하면 컴파일시간에 해당 annotation이 선언된 클래스를 모두 open으로 만들어주는 플러그인입니다.

예를 들면 아래는 @Entity가 붙은 클래스는 모두 open으로 만들고, 디폴트 생성자를 만들어 낸다는 것을 의미합니다.

```groovy
apply plugin: "kotlin-noarg"
apply plugin: "kotlin-allopen"

allOpen {
  annotation("javax.persistence.Entity")
}

noArg {
  nnotation("javax.persistence.Entity")
}
```

