---
layout: post
title: 배포(Deployment)
data: 2019-08-22 13:42:00
categories: common-sense
permalink: /common-sense/what-is-deployment
tags: deployment
author: kimjongmo
---

## 배포

개발이 완성된 프로그램을 서버에 이전시키는 일

## 배포는 왜 어려울까

- 개발이 완성된 프로그램이 설치되어 있어야한다(installed)
- 프로그램이 구동될 수 있도록 구성되어 있어야한다(configuration)
- 내 개발 프로그램을 서버에 배포 했을 때 제대로 돌아가야 한다.(running)
- 프로그램이 제대로 작동하는지 테스트한다(testing)

## 애플리케이션 라이프 싸이클에 따른 환경들

- `Local`
  - 개발자가 가지고 있는 로컬 환경
  - 기능 개발이 이루어진다.
- `Dev`
  - 각 기능들을 통합해서 테스트하는 환경
- `Staging`
  - 프로덕션에 넘기기전 프로덕션과 동일한 환경으로 구성하여 테스트
- `Production`
  - 실제 서비스를 하는 환경



### 과정의 예시

1. A개발자가 기능 개발 혹은 버그 수정을 마친 후 Dev 환경에 알린다
2. Dev환경에서 A개발자의 기능을 추가시킨 후 테스트한다.
   1. 테스트를 실패했을 시 다시 1번 과정으로
3. 통합된 코드를 Staging 환경에 넘긴다.
4. Staging 환경은 실제와 거의 동일하게 구성된 환경으로 실제 프로덕션 환경에서도 제대로 기능이 동작하는지 테스트한다.
   1. 테스트를 실패했을시 다시 1번 과정으로
5. 테스트를 통과하게 되었을 때 Production 환경으로 넘긴다.

## 기존의 프로세스

- 병합의 날이 존재
- 각 개발자들이 개발한 코드를 하나씩 병합한다.
- 빌드 및 테스트
- 독립적으로 작성된 코드는 다른 개발자가 동시에 적용하는 변경 사항과 충돌할 가능성이 크다.
- 통합 지옥
- 통합하는 과정은 부드럽고 매끄럽지 않기 때문에 통합하는 과정에서 수정을 하게 되는데 이 과정이 몇 시간 또는 며칠이 걸리게 될 수 도 있다. 
- 통합 완료가 됨(개발 완료)
- 테스트 환경에서 다시 빌드 및 테스트
- 프로덕션 환경에 적용
  - ssh 접속
  - 저장소에서 새로운 코드를 업데이트
  - 빌드 및 테스트
  - 프로그램 재 실행

## CI/CD

### CI(Continuous Integration):지속적 통합

`지속적 통합(CI)`이란 `통합 지옥`을 해결하기 위한 방안으로 애플리케이션에 대한 새로운 코드 변경 사항이 정기적으로 통합 서버에서 빌드되고 테스트되어 결과를 전달해주는 것.

개개인의 개발자는 자신이 작업한 것을 커밋했을 때 CI를 통해서 해당 기능이 전체 프로그램에 영향을 주는지 확인을 할 수 있다.

<img src="https://deploybot.com/assets/blog/_normal/CIinfographic.png">

(이미지 출처 : [https://deploybot.com/blog/the-expert-guide-to-continuous-integration](https://deploybot.com/blog/the-expert-guide-to-continuous-integration))

### CD(Continuous Delivery):지속적 배달

CI를 걸쳐 성공적으로 통합이 되었을 때, 테스트 환경과 스테이징 환경까지 배포하는 것을 자동화시키는 것.

### CD(Continuous Deployment):지속적 배포

지속적 배달이 CD가 테스트 환경과 스테이징 환경까지 범위라면 Continuous Deployment는 프로덕션환경에 릴리즈하는 과정까지 자동화시키는 것. 


CI/CD는 <mark>애플리케이션 개발 단계를 자동화</mark>여 애플리케이션을 보다 짧은 주기로 고객에게 제공하는 방법.

## Reference

>- [https://www.redhat.com/ko/topics/devops/what-is-ci-cd](https://www.redhat.com/ko/topics/devops/what-is-ci-cd)
>
>- [https://programmers.co.kr/learn/courses/9453](https://programmers.co.kr/learn/courses/9453)