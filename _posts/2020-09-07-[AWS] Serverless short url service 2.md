---
layout: post
title: Serverless-Short-URL-SERVICE-2
data: 2020-09-15 22:07:00
categories: aws
permalink: /aws/serverless-short-url-service-2
tags: python s3 cloud-front lambda
author: kimjongmo

---



# AWS를 활용한 서버리스 단축 URL 만들기 2편

## 전체 아키텍처

이 각각의 서비스는 아래와 같이 구성이 됩니다.

![구성도](/img/short-url/short_url_architecture.PNG)



### 구현 방향 

1. S3 생성 및 정적 웹 사이트 호스팅 전환
2. 람다 생성, short url 라이브러리를 이용하여 주소 단축, s3 저장
3. API Gateway 와 람다 연결 및 REST 보안 설정
4. CloudFront 및 DNS 설정



---------------------------------------------------------------------------------------

***지난 포스팅에서는 1, 2번을 구현했었고, 이번편은 남은 부분들을 이어서 해나갈 것입니다.***

----------------------------------------------------------



## API Gateway 와 람다 연결 및 REST 보안 설정

이전편에서는 람다의 테스트를 통해서 내부적으로 람다를 실행시켰었는데 이제 이 람다를 외부에서도 연결하고 호출하기 위해 API Gateway 서비스와 연동을 할 것입니다.

### API  생성

API Gateway 서비스에 접속을 하고 메인페이지에서 아래의 `REST API`를 선택해줍니다.

![api-gw-1](\img\short-url\api-gw-1.png)



생성하기를 눌렀으면 아래 사진과 같이 구성시켜줍니다.

![api-gw-2](\img\short-url\api-gw-2.png)



### 리소스 구성 및 메서드 추가

다음으로는 우리가 이전에 만들었던 람다를 연결시켜주는 작업을 할 것입니다. 

REST API를 만들고 싶으니 리소스를 먼저 정의해줍니다. 다음과 같이 `작업` - `리소스 생성`을 클릭합니다. 리소스 이름은 short-url 로 하고 리소스 생성을 눌러 생성해줍니다.

![api-gw-5](\img\short-url\api-gw-5.png)



리소스가 정의되었으면 행위에 대해서 API 를 정의해봅니다. short-url 을 생성한다는 의미로 HttpMethod는 POST로 만들겠습니다.

 `작업` -`메서드 생성`- `POST` - 우측 체크 버튼 클릭

![api-gw-3](\img\short-url\api-gw-3.png)

 

우측 체크 버튼을 누르면 아래와 같은 화면이 나올텐데 이 부분에서 이제 우리는 람다함수를 설정하도록 하겠습니다. 람다 리전 및 함수 이름은 이전에 만들었던 람다 함수의 내용을 써주시면 됩니다. `저장`을 누르시면 람다 함수를 호출하기 위해 권한을 부여한다는 알람이 뜨는데 `확인`을 눌러줍니다.

![api-gw-6](\img\short-url\api-gw-6.png)



### 스테이지에 배포

람다함수와 API 연결 설정하였으나 여기서 끝이 아닙니다. 우리가 한 작업은 말그대로 설정일 뿐 실제로 이  API를 배포를 해야지 접근가능합니다. 아래와 같이 `작업`-`API 배포`를 눌러줍니다.

![api-gw-7](\img\short-url\api-gw-7.png)



배포 스테이지는 `[새 스테이지]`를 누르고 스테이지 이름은 알아서 하는데 저는 beta 라고 하겠습니다. 배포를 하고 좌측에 `스테이지` 를 누르면 위에서 정한 이름의 스테이지가 보입니다.  아래 사진과 같이 POST 를 누르고 오른쪽에 나오는  URL 호출을 복사합니다.

![api-gw-8](\img\short-url\api-gw-8.png)



### 외부에서 요청하기

해당 URL을 요청해볼텐데 저는 이때 postman 툴을 이용해서 요청해보도록 하겠습니다.  

[설치](https://chrome.google.com/webstore/detail/postman/fhbjgbiflinjbdggehcddcbncdddomop?hl=ko)

설치를 마치고 Post man 을 실행시키면 아래와 같은 화면이 보입니다.

 ![postman-1](\img\short-url\postman-1.png)



POST 요청 그리고 short-url 을 만드는데 필요한 long_url, number 등을 설정합니다.

![postman-2](\img\short-url\postman-2.png)



`send` 를 눌러 요청을 보내봅니다.  아래로 내리면 응답이 보입니다. 제대로 요청이 되었고 결과가 처리되었네요

![postman-3](\img\short-url\postman-3.png)



### 보안 작업

이 부분은 해도 되고 안해도 되는 부분이지만 만약 이렇게만 API Gateway  설정을 마치게 되면 누구든지 내 URL을 이용할 수 있을 겁니다. 따라서 이 URL에 어느정도 보안 설정을 해놓고 싶습니다. 카카오나 외부의 다른 API 를 이용해보신분들은 익숙할텐데  키를 하나 발급할 것입니다.

### 키 발급

이전화면으로 돌아가 하단의 `API 키` 눌러줍니다. 그리고는 `작업` - ` API 키 생성`을 눌러줍니다. 내가 사용할 키니 이름은 `admin` 이라고 하고 저장을 눌러줍니다.

![api-gw-9](\img\short-url\api-gw-9.png)



키가 생성이 되면 아래의 화면이 나올텐데 `API 키` 의 `표시` 부분을 누르면 키를 볼 수 있습니다

![api-gw-10](\img\short-url\api-gw-10.png)

다음으로 좌측 메뉴에서 `사용량 계획`을 들어가줍니다. `생성`을 누르고 다음과 같이 세팅합니다. 어드민용이기 때문에 따로 할당량은 조절하지 않겠습니다. 

![api-gw-13](\img\short-url\api-gw-13.png)



`다음`을 누르고 `API 스테이지 추가`를 눌러 우리가 위에서 만들었던 스테이지에 이 계획을 적용해줍니다.

![api-gw-14](\img\short-url\api-gw-14.png)



다음으로 위에서 만든  API  키와 사용량 계획을 연결합니다.

![api-gw-15](\img\short-url\api-gw-15.png)

### 메서드 설정 변경

API Key 를 발급을 받았으니 이를 메서드에 연결해주도록 하겠습니다. 화면과 같이 이전의 `리소스 화면`의 `POST` 메서드를 클릭하고 우측에 있는 그림 중에서 `메서드 요청`부분을 클릭하고 아래의 사진처럼 `API 키가 필요함` 부분을 true로 수정하겠습니다.

![api-gw-12](\img\short-url\api-gw-12.png)

![api-gw-11](\img\short-url\api-gw-11.png)

이전과 동일하게 배포를 하도록 합니다. 이전에는 [새 스테이지]였지만 이번에는 아까 만든 스테이지에 재배포하도록 합니다.



### 외부에서 요청하기

아까 위에처럼 동일하게 만들고 똑같이 요청을 눌러봅시다.

아까와는 달리 이번에는 결과가 안나오고 아래와 같은 에러메시지가 결과로 나오게 될겁니다.

![postman-4](\img\short-url\postman-4.png)



제대로 적용이 되었다는 뜻입니다. 이제 요청을 할 때 API-key를 세팅해서 요청하도록 수정하여야 합니다.

아래처럼 `x-api-key`헤더를 추가하고 값으로 방금전 받은 `api-key` 값을 삽입해줍니다.

![postman-5](\img\short-url\postman-5.png)

이제 다시 요청을 눌러봅니다. 제대로 작동합니다.

> number 값 변경해줘야 되는거 잊지 마세요!!! 



## CloudFront - S3 연결

CloudFront 설정은 실무에서는 굉장히 복잡하지만 포스팅의 목적은 한번 해보는 것이기 때문에 정말 대충 연결하고 끝낼 것 같습니다. 



CloudFront 서비스로 들어갑니다. 메인페이지 한가운데 보이는 `Create Distribution`  를 누르게 되면 `Web` , `RTMP` 가 보이는데  Web을 선택하도록 합니다.

그러면 아래의 화면이 보일텐데 아래와 같이 노란 부분만 세팅해주도록 합니다. 

![cloudfront-1](\img\short-url\cloudfront-1.png)

![cloudfront-2](\img\short-url\cloudfront-2.png)



이렇게 CloudFront 를 설정하고 생성을 눌러주면 아래의 화면과 같이 뜨게 됩니다

여기서  DomainName 부분은 복사해서주세요 람다에서 사용할 겁니다.

![cloudfront-3](\img\short-url\cloudfront-3.png)



람다 함수로 들어가서 환경 변수의 PRE_FIX 값을 위에 domainName으로 변경해주세요.

![cloudfront-4](\img\short-url\cloudfront-4.png)



소스의 수정도 필요합니다. 노란부분이 `obj_key`라고 되어있을텐데 사진과 같이 바꿔줍니다.

 ![cloudfront-5](\img\short-url\cloudfront-5.png)



람다를 수정했으니 다시 저장해줍니다.



이거 설정하다 보면 CloudFront 가 배포를 마쳤을 겁니다.



포스트맨을 이용해서 다시 요청을 해보도록 합시다. {도메인 네임}/{encode_url} 형식으로 제대로 나오나요?

![cloudfront-6](\img\short-url\cloudfront-6.png)



## 마치며

- cloudFront 도메인 이름은 솔직히 짧은 편은 아닙니다. 보통 short-url이라고 하면 더 짧은 주소가 많은데요. 이는 따로 도메인을 구입해서 AWS Route53 에 등록하고 또 Route53과 CloudFront를 연결해주는 작업이 필요합니다. 포스팅 하기 전에는 도메인사서 이 부분도 마저 하고 싶었는데 생각보다 제가 원하는 도메인이 없는 것도 있지만, 뭔가 잘 사용안할 것 같은 느낌으로... 이 부분은 건너뛰도록 하겠습니다. 

 

- 가능하다면 공식 문서를 읽어가면서 필요한 설정, 옵션 등을 만져보시는걸 추천드립니다.  



- 포스팅을 시작하기 전에는 정말 수준높은 글을 쓰고 싶었는데 체력이 갈수록 딸려서 대충 대충 만들게 된 점 죄송합니다. 지금은 없지만 조만간 댓글 플러그인을 추가하려고 하는데 궁금하신점이 있으시면 댓글에 달아주시면 최대한 빠르게 응답하도록 하겠습니다. 정말 급하신 분은 우측 프로필 아래의 E-mail 로 연락주시면 감사하겠습니다.  