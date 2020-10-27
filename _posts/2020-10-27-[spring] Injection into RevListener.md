---
layout: post
title: Injection into RevisionListener
data: 2020-10-27 20:00:00
categories: spring
permalink: /spring/inject-into-rev-listener
tags: spring-boot hibernate-envers kotlin 
author: kimjongmo

---



# RevisionListener에 빈 주입하기

hibernate-envers 를 사용하다보면 RevListener를 작성할 일이 생길것이다. 이 때 버전에 따라 다르지만 5.3 이전의 버전에 대해서는 평범하게 빈을 주입할 수 없고 특별한 방식으로 다루어야 한다.



## Hibernate-Envers

hibernate-envers는 JPA 및 Hibernate, Spring 등등에서 작동가능한 모듈이다. 간단하게는 엔티티 클래스에 대해서 변경 이력을 쉽게 관리할 수 있도록 해준다. 단순 변경 이력만 남긴다고 가정하면 어노테이션 몇번이면 가능하다. 



## 암튼 원인..

각설하고 RevisionListener에 빈 주입이 안된다면 일단 버전을 먼저 확인해보자.  [문서](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#envers-tracking-modified-entities-revchanges)에 나와있기를 5.3 이전 버전에 관해서는 의존 주입이 지원되지 않다가 5.3 이후부터 지원을 한다고 적혀있다. (* 위 하이퍼링크로 들어가면 위로 조금만 올리면 바로 나옵니다.) <u>5.2까지는 하이버네이트 프로젝트에서 RevisionListener 구현체를 직접 만들고 그 만들어진 구현체를 스프링이 꺼내 쓰게 되느라, 자동주입이 안 먹힌다는 것 같다</u>. 그래서 만약 내가 지금 사용하고 있는 버전이 5.3 이하라면 아래의 방법을 따르면 되겠습니다. 아 5.3 이후에도 안된다는 글이 있는데 그렇다면 [여기](https://stackoverflow.com/questions/57902388/how-to-inject-spring-beans-into-the-hibernate-envers-revisionlistener/57910884#57910884)를 한번 보심이..



## 암튼 해결..

위의 밑줄과 같이 하이버네이트 프로젝트에서 만들어진 구현체를 스프링이 가져다가 사용하는 방식이라 그 사이에서 빈이 주입이 안된다는 것 같은데 static 영역에 변수를 하나 만들고 해당 변수에 setter injection을 이용하기로 한다. 

```kotlin
class MyRevListener: RevisionListener {
    @Autowired
    fun setMyBean(myBean: MyBean){
        MyRevListener.myBean = myBean
    }
    
    override fun newRevision(revisionEntity: Any) {
        ...
    }
    
    companion object {
        lateinit var myBean: MyBean
    }
}
```

이렇게 되면 JVM 에 클래스가 올라갈 때 static 영역에 들어가게 되고 런타임에 스프링이 해당 빈을 세팅해줄 수 있을 것이다.



