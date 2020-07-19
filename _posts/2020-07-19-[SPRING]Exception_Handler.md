---
layout: post
title: Exception_Handler
data: 2020-07-19 22:02:00
categories: spring
permalink: /spring/exception-handler
tags: spring-boot kotlin exception-handler
author: kimjongmo

---

# Controller Exception Handler

컨트롤러에서 에러를 다루는 방법은 여러가지가 있다.

- try-catch 문을 이용해서 에러가 났을 때 처리하기
- 컨트롤러 내부에서 ExceptionHandler를 작성하여 공통적으로 이용하기
- 일정 범위의 패키지내에서 공통적으로 에러를 처리하기

이 정도가 생각이 난다.



## Try - Catch

가장 먼저 try-catch를 이용해서 에러를 처리하는 방법은 추천되지 않는 방법이다. 내가 생각하는 이유는 두 가지 정도가 있는데 

1. 로직에 try-catch를 삽입하게 되면 코드가 더러워진다. 메인로직에만 집중할 수 있도록 코드가 작성되어 있으면 좋겠다. 

2. 리턴타입에 영향이 갈수있기 떄문이다. 간단하게 예를 들어보면 아래와 같은 엔드포인트가 있다고 가정해보자

```kotlin
// item 
data class Item (
	var id: Long, 
	var name: String
)

@RestController
class ItermApiController {
	@GetMapping("/item/{id}")
  fun getItem(@PathVariable id: Int): Item {
      return try {
        itemRepository.findById(id)
      } catch(e: NotFoundException) {
					
      }
  }
}
```

에러를 처리하면서 동시에 어떠한 메시지를 담아주고 싶은데 엔드포인트의 리턴 타입이 Item이기에 Item에 에러 메시지를 담아줄 수있도록 필드를 추가하거나, 에러를 따로 처리할 수 있도록 Item을 감싸고 있는 특별한 클래스가 필요하다.

```kotlin
// 필드 추가하거나
data class Item (
	var id: Long,
  var name: String,
  var errorMessage: String
)

// 데이터를 감싸려는 특별한 클래스
class Common<T> (
  var message: String,
  var status: String,
  var data: T
)
```

그나마 후자의 방법이 좀 더 좋아보이기는 하지만 여전히 try- catch문을 사용해야한다는 점 이와 비슷하게 다른 곳에서도 모두 작성해주어야 한다는 점이 있다.

## @ExceptionHandler 

`exceptionHandler`는` try-catch`의 단점들을 보완할 수 있는 방법 중 하나이다. 컨트롤러 내부에서 공통적으로 처리할 예외들을 한곳에 정의를 내릴 수 있고, 메인 로직에 사용할 필요없이 메서드로 분리하여 관리한다. 이렇게 하면 API를 이용하는 곳에서는 응답의 `status`를 보고 정상처리가 되지 않았다면 `message`를 가져올 수 있을 것이다.

```kotlin
// 에러 내용을 담기 위한 공통 클래스
data class ErrorResponse (
	var message: String = "에러"
)

@RestController
class ItermApiController {
  
  @ExceptionHandler(NotFoundException::class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  fun notFoundExceptionHandler(e: NotFoundException) :ErrorResponse {
    	return ErrorResponse(e.message)
  }
	@GetMapping("/item/{id}")
  fun getItem(@PathVariable id: Int): Item {
      return itemRepository.findById(id)
  }
}
```

하지만 여기서도 하나 고민이 생긴다. NotFoundException 같은 에러는 ItemController 에만 적용할 것이 아니라 다양한 곳에서도 발생할 수 있을텐데 그렇다면 모든 컨트롤러에 exception handler를 작성해주어야 하는 것일까?

## @ControllerAdvice

Controller 간의 공통된 예외를 처리할 때에는 @ControllerAdvice를 사용할 수 있다.  

```kotlin
@RestControllerAdvice
class GlobalAdvice {
  @ExceptionHandler(NotFoundException::class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  fun notFoundExceptionHandler(e: NotFoundException) :ErrorResponse {
    	return ErrorResponse(e.message)
  }
}
```

위와 같이 글로벌한 advice를 만들면 각각의 컨트롤러에서 사용할 공통적인 예외 처리를 할 수 있다. 또한 @ControllerAdvice의 프로퍼티들을 적절하게 사용한다면 특정 패키지 범위, 클래스에 대해서만 예외 처리를 하도록 설정할 수 있고, 각각의 Advice 사이에서 우선순위를 표현할 수도 있다.


> [ControllerAdvice의 option](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/ControllerAdvice.html)
> [RestControllerAdvice vs ControllerAdvice](https://stackoverflow.com/questions/43124391/restcontrolleradvice-vs-controlleradvice) 

