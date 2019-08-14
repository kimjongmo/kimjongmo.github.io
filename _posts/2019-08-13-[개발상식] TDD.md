---
layout: post
title: TDD
data: 2019-08-13 22:35:00
categories: common-sense
permalink: /common-sense/introduce-tdd
author: kimjongmo
tags: TDD
---

## 테스트

[테스트란?](https://kimjongmo.github.io/common-sense/what-is-test)

## 테스트 주도 개발(Test-driven development TDD)

- 테스트 주도 개발은 매우 짧은 사이클을 반복하는 소프트웨어 개발 프로세스 중 하나
- <mark>애자일(Agile)</mark>의 대표적인 개발 방법론 중 하나
- 먼저 테스트 케이스를 작성 후, 테스트 코드를 생성한다. 후에 개발을 진행하는 방식.
- 프로젝트의 품질 향상, 설계 개선, 유지보수의 접근성에 효과적
- 전통적인 개발 프로세스에 비해 초기 투자 비용(설계, 시간, 노력 등)이 증가



## TDD 프로세스

![](/img/19-08-13/traditional-dev-process.png)

[전통적인 개발 프로세스]

![](/img/19-08-13/test-driven-dev.png)

[테스트 주도 개발 프로세스]

<br>

## TDD는 무조건 좋은것인가?

위에 TDD를 설명했을 때 얘기했던 것처럼 TDD를 적용하게 되었을 때 소프트웨어의 품질이 향상된다는 점과 같이 프로젝트의 성숙도를 높이는 개발 방법이라 생각한다. 하지만 회사의 개발 문화가 어떤지에 따라서 TDD를 적용할지 안할지를 결정한다고 생각한다. 예로 SI 환경에서는 기간 내에 프로젝트를 마치는 것에 집중하기 때문에 유지보수까지 고려하는 경우는 드물다고 생각한다. 또한 개발 문화가 빠른 개발,리팩토링에 초점을 맞출 경우에도 TDD를 굳이 적용할 필요는 없다고 생각한다. 

[TDD가 SI업체와 궁합이 잘 맞을까? (slipp.net)](https://www.slipp.net/questions/161)

<br>

## TDD 연습하기

[박재성 - 의식적인 연습으로 TDD, 리팩토링 연습하기 (OKKYCON:2018)](https://www.youtube.com/watch?v=cVxqrGHxutU) 

내용 중..

- 1단계 - <mark>단위 테스트 연습하기</mark>
  - TDD와 단위 테스트를 만드는 것은 다르다
  - 단위 테스트가 먼저 되어야 한다
- 2단계 - <mark>TDD 연습하기(테스트 실패 ~ 테스트 성공)</mark>
  - 토이 프로젝트(웹, 모바일 UI나 DB에 의존관계를 가지지 않는 프로젝트)로 시작해라
- 3단계 - <mark>리팩토링 연습하기</mark>
  - 들여 쓰기가 2인 곳을 메서드로 분리하라
  - else를 쓰지 않는다.
  - 하나의 메서드가 하나의 기능만을 하도록 만들어라
  - 클래스로 분리한다 = 객체지향적인 코드로 만들기
    - 모든 원시값과 문자열을 포장하라
    - 일급 컬렉션을 사용하라
- 4단계 - <mark>토이 프로젝트의 난이도를 높여간다</mark>
  - 게임과 같은 요구사항이 명확한 프로그램으로 연습
  - 의존관계(웹 UI,데이터베이스, 외부 API와 같은..)가 없이 연습
  - 약간은 복잡한 로직이 있는 프로그램
  - 추천 : 로또,사다리타기,체스 게임,볼링 게임,지뢰 찾기
- 5단계 - <mark>의존 관계 추가를 통한 난이도 높이기</mark>
  - 웹, 모바일, 데이터베이스 등과 같은 의존 관계 추가

<br>

### TDD 경험해보기

> "125+127" 의 문자열의 숫자와 기호를 읽고 계산해서 답을 산출하는 프로그램 작성해보기
>
> 조건1) 문자열은 2개의 수와 기호로 이루어져 있다.
>
> 조건2) 조건1에 맞지 않는 문자열이 주어지면 ArithmeticException을 발생시킨다.

- 프로젝트 생성 및 github 연동
  - Java, Sping Boot, Maven로 구성하여 프로젝트 생성
  - https://www.gitignore.io/ 를 이용하여 .gitignore 생성 및 추가
  - github 레포지토리 생성 후 프로젝트를 이곳에 푸시
  
- <mark>실패하는 테스트 코드</mark> 작성

  - 의존 라이브러리를 뒤져보았을 때 junit버전이 4였으므로 https://junit.org/junit4/ 필요해보이는 것은 이곳을 통해서 계속 찾았다.
    ![](/img/19-08-14/junit-version.PNG)
  - 처음에는 컴파일을 실패하는 테스트 코드를 작성한다.
    ![](/img/19-08-14/compile-fail.PNG)
  - 컴파일까지는 할 수 있도록 소스를 구현시켜준다.
    ![](/img/19-08-14/compile-pass.PNG)
  - 컴파일이 가능하게 되었을 때 테스트를 진행시켜 실패가 나오도록 한다.
    ![](/img/19-08-14/test-fail.PNG)

- 테스트 <mark>통과 코드 구현</mark>

  ![](/img/19-08-14/test-pass.PNG)
  
- <mark>리팩토링</mark> 하기

  - 리팩토링 전 코드

    ```java
    public class StringCalculator {
    
        private static Logger logger = LoggerFactory.getLogger(StringCalculator.class);
    
        /**
         * @param text Text is must consist of two number and sign.
         */
        public static int cal(String text) {
    
            char[] arr;
            int len;
            int operationIndex = -1;
            int ret = 0;
            int num1;
            int num2;
    
            if (text == null || Strings.isEmpty(text)) {
                throw new ArithmeticException("빈문자 혹은 널값은 불가합니다.");
            }
    
            if (!text.matches("^([0-9]+)([+]|[-]|[/]|[*])([0-9]+)$")) {
                throw new ArithmeticException("적절하지 않은 포맷입니다.");
            }
    
            arr = text.toCharArray();
            len = arr.length;
    
            for (int i = 0; i < len; i++) {
                if (arr[i] < '0' || arr[i] > '9') {
                    operationIndex = i;
                    break;
                }
            }
    
            num1 = Integer.parseInt(text.substring(0, operationIndex));
            num2 = Integer.parseInt(text.substring(operationIndex+1));
            switch (arr[operationIndex]) {
                case '+':
                    ret = num1 + num2;
                    break;
                case '-':
                    ret = num1 - num2;
                    break;
                case '*':
                    ret = num1 * num2;
                    break;
                case '/':
                    ret = num1 / num2;
                    break;
            }
            return ret;
        }
    
    }
    ```

  - 들여쓰기가 2인 곳을 없애자

    연산자 인덱스 `operationIndex`를 구하는 곳이 들여쓰기가 2개 째가 되었는데 이를 <u>일단 메서드로 빼지 않고 코드를 수정</u>하여 들여쓰기를 하나로 만들었다

    ```java
    int temp = 0;
    
    while ('0' <= arr[temp++] && arr[temp] <= '9') ;
    
    operationIndex = temp - 1;
    ```

    다음으로 `switch 구문`에 들여쓰기가 2이상이게 되는데 if-else if 구문으로 들여쓰기를 2이하로 줄일 수 있지만 switch문이 좀 더 코드가 깔끔하다고 생각하기 때문에 이 부분은 냅두기로 하였다.

  - else 제거

  - 하나의 메서드가 하나의 기능만 하도록 만들기
    cal메서드에는 아래와 같은 기능들을 하고 있다.

    - 텍스트 검사
    - 연산자 인덱스 찾기
    - 텍스트의 숫자로 표기된 부분 파싱
    - 연산 기호에 따른 계산

    이들을 각각 메서드로 만들어 분리하도록 하였다. 리팩토링을 한 후 다시 테스트를 하여 통과하는지 보도록 한다.

  - 객체지향적인 코드 만들기

    - 연산 기호에 따른 계산 메서드를 Calculator라는 클래스로 만들었다.

      ```java
      public class Calculator {
      
          public static int cal(int[] number, char op) {
              int ret = 0;
              switch (op) {
                  case '+':
                      ret = add(number[0], number[1]);
                      break;
                  case '-':
                      ret = minus(number[0], number[1]);
                      break;
                  case '*':
                      ret = mul(number[0], number[1]);
                      break;
                  case '/':
                      ret = div(number[0], number[1]);
                      break;
              }
      
              return ret;
          }
      
          public static int add(int num1, int num2) {
              return num1 + num2;
          }
      
          public static int minus(int num1, int num2) {
              return num1 - num2;
          }
      
          public static int mul(int num1, int num2) {
              return num1 * num2;
          }
      
          public static int div(int num1, int num2) {
              return num1 / num2;
          }
      }
      ```

    - 어렵다.. 

지금까지 했던 연습 코드는 https://github.com/kimjongmo/practice-tdd 에 올려두었다. 앞으로도 꾸준하게 TDD를 생각하면서 연습을 해야겠다. 



