---
layout: post
title: Replication 실습 
data: 2019-07-19 13:46:00
categories: database
permalink: /database/replication-training
tags: db replication-training mysql docker
---



# Replication 실습하기

## 참고 자료

<https://dev.mysql.com/doc/refman/5.7/en/replication.html>

<br>

## 목표

- docker를 이용해 DB를 여러 대 실행시키고, 이들을 연결시켜 replication을 실습하는 것을 목표한다.

<br>

## 준비물

- docker

<br>

## Binary log file position based Replication

### Mysql 설치

1) mysql 이미지 다운로드

```bash
docker pull mysql/mysql-server:5.7 
```

2) mysql 실행

```bash
docker run --name=mysql -d  -p 3306:3306 \
--mount type=bind,source=/home/ubnutu/mysql,target=/var/lib/mysql \
--mount type=bind,source=/home/ubuntu/my.cnf,target=/etc/my.cnf \
mysql/mysql-server:5.7
```

`-v`태그를 이용하여 바인딩할 때 `:`왼쪽의 경로는 마운팅할 로컬의 경로로써 나는 ubuntu를 사용중이니 위와 같이 지정한 것이다. 

> 이 부분에서 꽤나 고생했는데 my.cnf과 관련하여 오류가 난다싶으면 일단 실행할 때 my.cnf mount부분을 제외하고 container shell에 접속하여 /etc/my.cnf을 복사하여 로컬 홈 디렉토리(/home/ubuntu)에 my.cnf파일을 만들고 복사한 내용을 붙여준다. 후에 다시 온전한 위의 명령어를 다시 실행해본다.

3) mysql 비밀번호 찾기

```bash
docker logs mysql 2>&1 | grep GENERATED
```

GENERATED ROOT PASSWORD : [비밀번호]

4) mysql server에 접속

```bash
docker exec -it mysql mysql -uroot -p
```

3번에서 나왔던 비밀번호를 입력하여 접속합니다.

5) 비밀번호 변경

```bash
alter user 'root'@'localhost' identified by '[바꿀 비밀번호]';
```

비밀 번호를 변경했으면 `exit`명령어로 나와줍니다. 

6) container 중지 및 삭제

```bash
docker stop mysql
docker rm mysql
```

7) 확인

```bash
docker run --name=mysql -d  -p 3306:3306 -v /home/ubnutu/mysql:/var/lib/mysql mysql/mysql-server:5.7
```

비밀번호를 칠 때 바꾼 비밀번호를 입력해서 제대로 마운팅되었는지 확인한다. 

<br>

### Master 세팅하기

마스터가 binary log file position based replication을 사용하기 위해서는 <span style="color: red;">반드시 유니크한 아이디를 할당하고, binary logging이 가능해야 한다. </span>

로컬의 `my.cnf` 파일에 아래의 내용을 추가해준다.

```bash
log-bin=mysql-bin
server-id=1
```

아래의 내용을 추가했다면 container를 내리고 다시 올리도록 하자.

<br>

### Slave의 Replication를 위한 유저 계정 만들어 주기

각 Slave에서 Master 에 접속하기 위해서는 Master에서 이들이 접속할 수 있도록 유저 계정을 만들어주어야 한다.

```sql
create user 'repl'@'*' identified by '[패스워드]';
grant replication slave on *.* to 'repl'@'*';
```



### 마스터 Binary Log 좌표 얻기

<br> 작성 예정..

### 데이터 스냅숏 방식 선택하기

<br>

작성 예정..

### Slaves 세팅

<br>

작성 예정..

### Slaves Replication에 추가시키기

<br>

작성 예정..



