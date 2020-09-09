---
layout: post
title: Serverless-Short-URL-SERVICE-1
data: 2020-09-07 22:07:00
categories: aws
permalink: /aws/serverless-short-url-service
tags: python s3 cloud-front lambda
author: kimjongmo

---



# AWS를 활용한 서버리스 단축 URL 만들기 1편

단축 URL 이란 길고 복잡한 주소를 짧은 주소로 매핑하는 기술입니다. 흔히 볼 수 있는 me2, bit, goo.gl 등이 이 단축 URL에 해당됩니다.

제가 생각하는 단축 URL 장점은 아래와 같습니다.

1) 제한된 문자수가 있는 메시지나 그 외 등에서 짧은 URL은 큰 도움이 될수 있다.

​	ex) 제한된 문자수가 100글자인 부분에 URL 길이 50글자가 들어간다면 URL 외에 사용할 수 있는 공간이 지극히 적어지게 됩니다.

2) URL에 개인정보나 그외에 민감할 수 있는 정보가 들어가 있는 URL 주소를 숨길 수 있다. 

​	ex) kjm.com?userId=aaa123&token=124901490149&hp=01012345678

3) 보기에 좋다.

단순하게 제 생각의 기준으로 내뱉은 말이기 때문에 위에 장점이 실제로도 그렇다고 생각하시면 안될 것 같습니다. 여러분도 같이 생각해보시죠!



단축 URL을 구현하는 방법 라이브러리는 상당히 많아서 구현하는 방법은 다양하지만 저는 그 중에서도 AWS를 사용하여 구현할 것입니다. 장점으로 보자면 일단 서버리스 환경의 서비스를 구축가능하다는 것입니다. 비유로 치면 내가 노래방을 가고 싶어서 노래방을 만드는게 아니라, 이미 완성된 환경의 노래방에서 일정 금액을 내고 사용하는 것입니다. 서버를 만들기 위한 어떤 프레임워크를 생성할 필요도 없고, 이 어플리케이션을 운영할 장비가 필요없으며, 어플리케이션을 관리하기 위한 UI작업이나 구현 등이 필요없게 되었습니다. 



## 사용할 서비스 & 구성도

> AWS 계정이 있어야 됩니다... 꼭이요!!



### 사용할 서비스

단축 URL 서비스를 만들기 위해 사용할 서비스는 아래와 같습니다.

- Amazon S3 : 리다이렉트 및 Short url Object 저장 용도
- AWS Lambda : Short Url 생성 및 S3에 적재
- Amazon API Gateway : 람다를 외부에서 엑세스하고 실행시킬 수 있도록 하는 역할
- CloudFront :  사용자가 현재 지역에서 가장 가까운 곳에 있는 S3 오브젝트에 접속할 수 있도록 도와줌
- Route : DNS



### 전체 아키텍처

이 각각의 서비스는 아래와 같이 구성이 됩니다.

![구성도](/img/short-url/short_url_architecture.PNG)



### 구현 방향 

구성 순서는 아래와 같이 하려고 한다.

1. S3 생성 및 정적 웹 사이트 호스팅 전환
2. 람다 생성, short url 라이브러리를 이용하여 주소 단축, s3 저장
3. API Gateway 와 람다 연결 및 REST 보안 설정
4. CloudFront 및 DNS 설정



---------------------------------------------------------------------------------------

***이번 포스팅에서는 1번과 2번 까지를 목표로 한다.***

----------------------------------------------------------



## S3 생성 및 정적 웹 사이트 호스팅 전환

S3 서비스에 들어가서 버킷을 하나 만듭니다. 이름은 자유로 하시고 저는 `my-short-url`이라고 만들었습니다. 다른 설정들은 디폴트로 두어도 상관없고 중간에 `권한 설정` 구간에서 `모든 퍼블릭 엑세스 차단`을 해제해주셔야 합니다.

![s3_config_1](/img/short-url/s3_config_1.png)

![s3_config_2](/img/short-url/s3_config_2.png)



이렇게 버킷이 생성되면 버킷을 클릭하여 들어가줍니다. 그다음에는 상단의 탭중에 권한 탭을 들어가 줍니다. 그러면 아래와 같은 화면이 나올텐데 이중에 정적 웹 사이트 호스팅을 누르고 아래와 같이 활성화 해줍니다.

![s3_config_3](/img/short-url/s3_config_3.png)



다음으로는 외부에서 객체(리소스)에 접근할 수 있도록 `버킷 정책`을 수정해주셔야 합니다. 다음과 같이 구성하도록 합니다.

```json
{
    "Version": "2012-10-17",
    "Id": "PublicReadGetObject",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-short-url/*"
        }
    ]
}
```



후에 `개요` 탭으로 돌아가 short_url 오브젝트들만 모을 폴더를 하나 생성 해줍니다. 저는 폴더의 이름을 `surl`로 사용하였습니다

![s3_config_4](/img/short-url/s3_config_4.png)

## 람다 설정

### 람다 생성

Lambda 서비스로 들어가줍니다.

오른쪽 상단의 `함수 생성`을 클릭해줍니다. 다른 부분은 건드리지 않고 함수의 이름과 런타임을 python 3.7로 설정해줍니다. 원한다면 본인이 편한 언어를 사용하셔도 좋을 것 같습니다. 다만 이 뒤에 사용할 short-url은 python library 이기 때문에 이를 대체할 다른 라이브러리를 찾아야 할 것입니다. 

![lambda_config_1](/img/short-url/lambda_config_1.png)



### 권한 설정

다음으로 구성해야 하는 부분은 권한 문제입니다. 최초 디폴트로 생성한 람다함수는 가장 기본적인 로그 관련 권한만 가지게 되는데 앞으로 우리가 하려는 부분은 s3에 대한 읽기 및 쓰기 권한이 필요하기 때문에 아래와 같은 단계를 거쳐야 합니다.

`권한` 탭 -  `실행 역할`  부분에 역할 이름 부분을 클릭하여 AWS IAM 서비스로 넘어가겠습니다

![lambda_config_2](/img/short-url/lambda_config_2.png)



IAM 서비스로 넘어와서 오른쪽을 보면 `인라인 편집기` 라는 버튼이 보입니다. 클릭해 줍니다. 그러면 아래와 같은 화면이 나올텐데 우리는 `s3`서비스의 대한 `getObject` 및 `putObject` 권한이 필요하기에 아래 사진과 같이 설정해줍니다. 

![lambda_config_3](/img/short-url/lambda_config_3.png)

다음으로는 리소스를 특정지어야 할 차례입니다. 내가 정확하게 어떤 S3 버킷에 대해 위의 작업을 할 것인지를 지정해야합니다. 방금 전에 만들었던 버킷 이름인 `my-short-url`을 저는 적어넣었습니다. 

![lambda_config_4](/img/short-url/lambda_config_4.png)

추가를 누른 후 하단에 `정책 검토`를 클릭하고 정책의 이름만 대충 설정해준뒤 다시 하단의 `정책 생성`을 눌러 완료해줍니다.

 

### 라이브러리 설치

라이브러리 선언해주면 알아서 설치 ...

좀 해주지 귀찮은 작업이 하나 있는데 로컬에서 라이브러리를 다운받고 이걸 zip으로 모아서 압축한다음에 upload 시켜줘야 한다. mac이라서 커맨드라인으로 사용할텐데 윈도우는 음... 어카징? Git bash 같은 tool로 할 수 있다면 좋을텐데 안되면 그냥 계속 마우스로 따라해줘야 할 것 같다. 

>pip 를 이용할건데 없으먼 설치하도록 하자.



작업 공간을 적당히 찾아서 아래의 커맨드를 따라가주도록 하자.

```bash
touch lambda_function.py

# 필요 라이브러리 설치
pip3 install --target . short_url  

# zip 생성
zip -r9 function.zip ./short_url

# zip에 함수 추가
zip -g function.zip lambda_function.py
```

이렇게 만들어진 zip을 Lambda에 업로드 시켜줘야 한다. 람다 함수로 돌아가서 내리다보면 코드 편집기와 그 우측 상단에 아래와 같은 버튼이 있다. 여기서 `zip 파일 업로드`를 클릭하고 위에서 만든 zip 을 업로드 해주자.

![lambda_config_5](/img/short-url/lambda_config_5.png)



업로드가 성공했다면 아래 사진과 같이 short_url 라이브러리 코드가 추가된 것이 보일 것이다.

![lambda_config_6](/img/short-url/lambda_config_6.png)



### 코딩 시작

드디어 코딩을 시작하게 되었습니다 후아 

코드를 일일이 설명하기 보다는 느낌만 보는 걸로 하면 좋을 것 같습니다.

```python
import short_url
import json
import boto3
import botocore
import os
from botocore.client import Config

BUCKET_NAME = os.environ["BUCKET_NAME"]
PREFIX = os.environ['PRE_FIX']
S3_CLIENT = boto3.client('s3')
S3_RESOURCE = boto3.resource('s3')
# URL을 만들 때 사용할 문자들, block_size 조정
shortener = short_url.UrlEncoder("PbNVLaqiJjMzHvTRs2ex35yXockGwCg8Bp7ESFZQtduWn6KAYm9r1DhU4f", 32)

def exists_s3_key(key):
  try:
    resp = S3_CLIENT.head_object(Bucket=BUCKET_NAME, Key=key)
    return True
  except botocore.exceptions.ClientError as e:
    if (e.response['Error']['Code'] == "404"): return False
    if (e.response['Error']['Code'] == "403"): return False
    raise e     

def lambda_handler(event, context):
    
  # 파라미터 가져오기
  long_url = event.get("long_url")
  number = event.get("number")

  # long_url 자체를 인코딩하지 않고 number를 이용해서 인코딩
  encode_uri = shortener.encode_url(number)
    
  # s3 저장 위치, 오브젝트 이름
  obj_key = "surl/"+encode_uri
    
  # 이미 존재하는 key 인지 확인
  if (exists_s3_key(obj_key)):
    print("Object Key is Collision")
    return { "short_url": "", "result":"ERROR" }
  else:
    print("Object Key Find!! "+obj_key)
    
  # s3 저장    
  print("Object Save...")
  resp = S3_CLIENT.put_object(Bucket=BUCKET_NAME,
                       Key=obj_key,
                       Body=b"",
                       WebsiteRedirectLocation=long_url, # redirect location
                       ContentType="text/plain")
    
  print("Saved!!")
    
  ret_url = PREFIX+"/"+obj_key    
  print("Return value is "+ret_url)
    
  return { "short_url": ret_url, "result":"OK" }
```



코드 편집기 아래로 가면 환경변수라는 곳이 있는데 이곳에는 아래 사진과 같이 추가해주도록 한다

여기서 BUCKET_NAME은 s3  만들 때 버킷의 이름을 , PRE_FIX는 나중에 바꿀거긴 하지만 지금은 일단 s3 정적 웹사이트 호스팅 주소를 입력한다. 

주소는 여기서 복사해오도록 한다.

![스크린샷 2020-09-09 오전 1.07.16](/img/short-url/lambda_config_7.png)

![스크린샷 2020-09-09 오전 1.05.40](/img/short-url/lambda_config_8.png)



> 동시성을 생각한다면 number 위부에서 받아오는 것은 딱히 추천되지 않습니다. Atomic Counter 같은 특별한 장치가 필요한데 저 같은 경우에는 mongoDB의 findOneAndUpdate를 이용해서 이런 카운터를 구현해서 사용하였습니다. 사용해보지는 않았지만 AWS Dynamo DB를 이용해서도 이러한 원자성 카운터를 구현할 수 있습니다.
>
> https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/WorkingWithItems.html#WorkingWithItems.AtomicCounters
>
> 외부에서는 URL만 받도록 하고 람다 내부적으로는 유니크한 숫자를 받아와서 short-url로 만드는 것이 좋은 그림인 것 같습니다.

### 테스트

아직 API Gateway를 앞단에 붙여넣지 않았기 때문에 외부에서 람다를 호출하지 못할 것인데 이를 내부에서 실행시키고 테스트해볼 수 있도록 되어있는 것이 있다. 상단 고정바에 있는 테스트를 누르고 아래 사진과 같이 세팅해준다.

![lambda_test_1](/img/short-url/lambda_test_1.png)

테스트를 저장 후에는 상단에 있는 `테스트`를 클릭해서 실제로 작동이 되는지를 확인한다.

![lambda_test_2](/img/short-url/lambda_test_2.png)

위와 같이 성공이 되었다면 S3에도 오브젝트가 제대로 저장이 되어있을겁니다

![lambda_test_3](/img/short-url/lambda_test_3.png)



s3의 정적 웹 사이트 주소를 사용하느라 엄청나게 길어져 버린 URL 이지만 <u>이 부분을 나중에 구입하실 짧은 도메인 주소를 붙일 것이니</u>까 이것자체에는 크게 신경쓰지 말고 접속해보도록 합시다.

제대로 Redirect 되나요?



## 1편 끝

1편에서는 가장 코어가 되는 람다와 S3를 이용해서 short-url이 어떻게 생성되는지 봤습니다. 

2편에서는 람다를 외부에서 실행할 수 있도록 앞 단에 API Gateway, Route53, CloudFront를 붙여줄 것입니다.

1편 끝가지 읽어주셔서 감사합니다. 2편에서 뵙겠습니다.



## Reference

> https://aws.amazon.com/ko/blogs/compute/build-a-serverless-private-url-shortener/