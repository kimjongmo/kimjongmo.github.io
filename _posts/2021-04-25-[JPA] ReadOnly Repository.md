---
layout: post
title: ReadOnly Repository
data: 2021-04-25 02:27:00
categories: jpa
permalink: /jpa/read-only-repo
tags: jpa
author: kimjongmo
---



#  





# 디비를 붙여야 하는데..

회사에서 작업을 하려다 보니 <u>과거 데이터 조회가 필요한 상황이 생겼다</u>. 우연히도(?) 관련 데이터를 이번에 몽고 데이터베이스로 마이그레이션 하면서 과거 데이터는 사용할 데이터 외에 모두 마이그레이션하지 않았다. 

![](\img\humorous\어디갔어.jpg)

그러면 과거 데이터를 다시 몽고로 다 옮기면 되지 않을까라는 생각도 했었지만 데이터양이 워낙 많기도 하고, 이 데이터를 몽고로 전부 옮기더라도 이번 작업 외에는 사용하지 않을 데이터라 용량만 잡아먹을 것 같았다. 그래서 이번 <u>작업을 진행하는 동안에만 해당 데이터베이스를 연결해놓고 사용하기로 하였다</u>.



# 누군가가 사용할 수 있을라나?..

작업을 위해 데이터베이스를 붙여놓고 보니 걱정됐던 건 "혹시 다른 개발자가 이걸 사용해서 뭔가 작업을 하면 어떻게 하지?" 혹은 "실수로 누군가가 여기로 데이터를 쌓게 된다면?"라는 생각들이 들기 시작했다.



![](\img\humorous\긴장.gif)

 

그냥 구두로 사람들에게 말할까? 싶다가도 또 누군가가 까먹고 이걸 사용할 수도 있으니 차라리 해당 데이터베이스에 작업을 할 수 없도록 READ ONLY 설정을 걸어서 배포하기로 결정했다.

 

# READ ONLY 설정하기

현재 회사에서 사용 중인 스택은 `SPRING BOOT`, `JPA` 이라 보통 Repository를 생성해서 디비에 접근하고 있다. 그리고 <u>모든 Repository에서는 JpaRepository를 상속받고 있다</u>. 이 상황에서 어떻게 리드온리 설정을 할 수 있을까?



### 데이터베이스에 Select 권한만 가지고 있는 유저 생성?

가장 간단한 작업이라고 생각을 했다. 그냥 리드온리 전용으로 유저를 만들어서 사용하면 되지 않을까 생각을 했었는데, 이렇게 작업을 하더라도 애플리케이션 레벨에서 이 연결된 데이터베이스가 리드온리인지 알 수가 없을 거라고 생각을 했기도 했고, 데이터베이스 레이어에서 select를 못하는 것이지 애플리케이션에서 CUD 작업 시도를 막기 어려울 것이라 생각했다.

![](\img\humorous\놉.jpg)

### @Transactional(readOnly = true)  ?

서비스 레이어에서 트랜잭션을 열고 이를 readOnly로 설정하면 어떨까? 근데 repository에 직접 접근하면 되지 않나라는 생각도 들었고, 다른 SAVE, saveAndFlush 등등의 접근을 제어할 방법도 생각나지 않았다.

![](\img\humorous\놉.jpg)

### Save, flush 등을 호출하면 예외를 던지기

위의 방법에서 더 나아가 생각해 본 방법이다. 그러면 repository에서 save, flush 등을 호출하면 IllegalAccessException 같은 걸 던지면 되겠다 생각을 했었는데, 뭔가 호출할 수 있도록 만들어놓고서는 호출하고 나니 예외를 던진다?? 역시 맘에 들지 않았다. ㅠㅠ

![](/img/humorous/놉.jpg)

### 사용할 메서드만 상속받자

다른 방법이 없을까 검색하다 보니 대부분은 모두 이 방법을 사용하는 것 같았다. 지금 문제가 JpaRepository를 상속받게 되면서 repository에서 save나 flush 등의 CUD 작업이 가능하게 된 거니까 간단하게 JpaRepository를 상속 안 받으면 되잖아

![](\img\humorous\개허탈.jpg)

# 사용할 메서드만 상속받기

## ReadOnlyRepository 인터페이스 생성하기

```kotlin
@NoRepositoryBean
public interface ReadOnlyRepository<T, ID> extends Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
}
```

- 두 개의 읽기 전용 메서드만 선언해서 이 ReadOnlyRepository를 만든다.
- Repository  인터페이스는 특별한 기능을 제공하는 인터페이스가 아니라 이 인터페이스가 Repository  용도로 사용될 것 이라는 걸 알리는 용도이다.
- @NoRepositoryBean 어노테이션은 Repository의 메서드를 정의하는 인터페이스라는 정보를 부여하는 것

[참조] https://www.baeldung.com/spring-data-read-only-repository



## 상속받아 작성하기

위에서 만든 read only repo를 확장해서 사용해보도록 하자.

먼저 간단한 엔티티를 만들고 시작하자

```kotlin
@Entity
@Table(name = "book")
class Book (
    @Id
    @GeneratedValue
    var id: Long = 0L,
    var name: String = ""
)
```

그리고 repository를 작성한다

``` kotlin
interface BookReadOnlyRepository: ReadOnlyRepository<Book, Long> {
    
}
```



이제 실제로 BookReadOnlyRepository를 주입받아서 사용을 해보면 알 수 있듯이 조회 외에 다른 작업을 할 수가 없다. 다른말로 다른 개발자들이 이 repository를 이용하면서 잘못 CUD를 호출할 일이 없다는 것이다. 

![](\img\humorous\윤후_끝.jpg)



## 마치며

내가 정말 나쁜 습관을 가지고 있다는 생각이 들었다. 반복적인 작업만 하고 생각을 잘 안하니 이렇게 단순한 대안이 머리에 떠오르지 않았던 것 같다. 이제는 코드 한 줄에도 고민을 담아서 작성해보자. 