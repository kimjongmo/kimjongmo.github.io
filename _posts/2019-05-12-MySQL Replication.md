---
layout: post
title: Replication 
data: 2019-05-12 22:05:00
categories: database
permalink: /database/replication
tags: db replication
---



## Replication

`Replication`이란 한글로 `복제`를 뜻한다. Database에서 Replication은 서버(Master)에서 하나 혹은 그 이상의 데이터베이스 서버(Slave)에 데이터를 복사하는 기능을 말한다. 이러한 복사의 장점은 다음과 같다. 

- `스케일 아웃` : 쓰기나 업데이트 작업은 마스터에게만 맡김으로 <u>마스터는 업데이트 기능에만 전념</u>할 수 있고, <u>읽기 작업은 슬레이브에서 할 수 있도록</u> 하고 읽기 성능을 향상시키기 위해 그 숫자를 늘리면 된다.(읽기 작업에 대해서 부하 분산)
- `데이터 보안` : 슬레이브는 <u>마스터 데이터를 손상시키지 않고 백업 서비스를 실행</u>할 수 있다.
- `분석` : <u>슬레이브에서 일어나는 정보 분석은 마스터의 성능에 영향을 주지 않는다</u>.



![](/img/database_replication.PNG)



### 복제 방식

이 방식을 요약하자면 다음과 같이 된다.

1) 마스터에서 업데이트 및 변경 사항(DML command)이 생김

2) 마스터의 Binary Log에 쿼리를 기록

3) 마스터의 Binlog dump 스레드가 이를 슬레이브의 I/O 스레드에 전송

4) 슬레이브의 I/O스레드는 받은 내용을 Relay Log에 기록.

5) 슬레이브의 SQL 스레드가 이 내용을 로컬에 적용



복제 방식에는 크게 두가지가 있다.  

- 바이너리 로그 파일을 이용한 복제 

- GTID 트랜잭션을 이용한 복제

바이너리 로그 파일을 이용한 복제의 경우 로그 파일명 위치 등 동기화 작업이 필요한 것에 비해 GTID 트랜잭션은 트랜잭션 고유의 ID를 이용하여 동기화하기 때문에 좀 더 간단하고, 다른 문제들(failover가 일어났을 때)을 해결할 수 있는 장점이 있다고 한다. 

또한 각 복제에서도 바이너리 로그 파일에 기록을 할 때 `SQL자체를 기록하는 방법`과 `변경사항에 대한 내용만 기록하는 방법`으로 다시 나뉘게 된다.
