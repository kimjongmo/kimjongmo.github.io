---
layout: post
title: Mock Layer
data: 2020-10-11 11:13:00
categories: spring
permalink: /spring/mock-layer
tags: spring-boot kotlin 
author: kimjongmo

---



# 3계층 구조

- Presentation Layer : 사용자 중점 계층 (front-end)
- Business Layer : 비즈니스 로직 계층, presentation layer의 요구를 받아 처리하는 곳 (back-end)
- Data Layer : 데이터베이스에 접근하여 데이터를 읽거나 쓰는 작업등의 역할





## Mock Layer란?

`Mock Layer`라는 개념이 존재하는 것은 아니고 제가 임의로 편하게 부르기 위해서 지어봤습니다. 위의 3계층 구조에서 Business Layer에 속하게 됩니다.

테스트 코드를 짜다보면 `mockito` 라는 테스트 프레임워크에 대하여 한번쯤은 들어보셨을 겁니다. 어떤 특정 요청에 대하여 개발자가 지정한 결과를 내려주는 방식인데 이러한 것을 실행 환경에서도 사용하기를 원했습니다. 한마디로 <u>개발 환경에서만 작동했으면 하는 특정 로직을 실행하는 가짜 객체</u>정도라고 생각하면 좋을 것 같습니다. 그 속에서도 아래와 같은 규칙을 지키고 싶었습니다.

1. 프로덕션 환경에서는 작동하지 않으며, 개발 환경에서만 적용됬으면 좋겠다.
2. 기존에 존재하는 코드들에 하드 코딩을 하기 싫었다. ex) 로직 주석 처리 후 return "";
3. Mock Layer 에서 로직이 실패했을 때에는 실제 객체에 다시 요청되게 만들어야 한다.



## Sample Code

아래는 샘플 코드입니다.

[https://github.com/kimjongmo/mock-layer](https://github.com/kimjongmo/mock-layer)



## Profile

스프링 부트에는 profile 이라는 개념이 존재한다. 각각의 코드에 대하여 @Profile 이라는 어노테이션과 함께 이름을 지정 후 프로젝트를 실행시킬 때 어떤 프로파일들을 실행시킬 건지 선언하면 해당하는 클래스만 생성되어 빈이 올라간다.

![profile-dev](/img/mock-layer/profile-dev.png)

![profile-prod](/img/mock-layer/profile-product.png)

![profile-select](/img/mock-layer/profile-select.png)

만약 위 사진과 같이 `dev` profile을 지정한다면 ItemServiceImpl , MockItemServiceImpl이 빈으로 올라가게 되고, `product` 프로파일을 지정한다면 ItemServiceImpl만 빈으로 올라가게 될 것이다.



## Primary Bean

현재 ItemService를 구현하고 있는 ItemServiceImpl, MockItemServiceImpl이 있는데 이럴 경우 itermService에 대하여 주입을 받으려 할 때 (물론 리스트로 받으면 구현체 모두를 가져올 수 있지만..)구체적으로 내가 어떤 빈을 사용할 것인지 명시를 해줘야 한다. 내가 어떤 빈을 사용할 것인지 명시하는 방법은 여러가지가 있다.

- 사용하려는 빈의 이름을 변수로 선언
- @Qualifier를 이용해서 빈을 지정
- 각각의 구현체에 우선 순위를 부여

우리의 상황을 생각해보면 `dev` 일 때에는 `MockItemServiceImpl`이 붙어야 하고, `product`일 때는 `itemServiceImpl`을 주입받아야 한다. 그러므로 특정 빈의 이름을 지정해서 사용할 수는 없다. 그래서 구현체에 우선순위를 부여할 것이다. mockItemServiceImpl과 itemServiceImpl 두 개의 빈이 생성되었을 때에는 mock이 우선순위가 높아야하므로 MockItemServiceImpl에는 @Primary 를 붙여 우선순위를 제일 높게 잡아준다.

![profile-prod](/img/mock-layer/profile-product.png)



## Proxy

MockItemServiceImpl에서 만약 mock 대상이 아니라면 실제 객체인 ItemServiceImpl 에 연결해주기 위해서 프록시와 같은 형태로 만들었다.

![proxy](/img/mock-layer/proxy.png)

mockRepository에서 요청 데이터를 찾을 수 없다면 실제 객체로 재요청하게 된다.

여기서 바로 위에서 말한 빈 명시를 해줘야 하므로 실제 객체는 아래와 같은 방식으로 주입받도록 하자.

![qualifier](/img/mock-layer/qualifier.png)



## Test

profile을 dev로 한번 실행시켜서 요청을 해보고, 다시 profile을 product로 지정해서 재실행 후 http://localhost:8080/item/1 을 요청해보자.



## 마치며

너무 간단한 예제 때문에 굳이 이렇게까지 할 필요가 있나 싶을수도 있지만 dev, product 각각의 환경속에서 내가 원하는 데이터만 mocking 할 수 있고, 각 환경에서 다른 로직을 가질 수 있을 수 있다는 점은 나름 유용하게 사용할 수 있을 것이다. 