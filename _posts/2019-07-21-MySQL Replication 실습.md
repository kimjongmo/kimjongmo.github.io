---
layout: post
title: Replication 실습 
data: 2019-07-21 20:56:00
categories: database
permalink: /database/replication-training
author: kimjongmo
tags: db replication mysql docker
---



# Replication 실습하기

## 참고 자료

<https://dev.mysql.com/doc/refman/5.7/en/replication.html>

## 목표

- docker를 이용해 DB를 여러 대 실행시키고, 이들을 연결시켜 replication을 실습하는 것을 목표한다.

## 준비물

- ubuntu
- docker

## Binary log file position based Replication

### Mysql 설치

1) mysql 이미지 다운로드

```bash
docker pull mysql/mysql-server:5.7 
```

2) mysql 실행

```bash
cd ~
mkdir mysql
touch my.cnf
```

```bash
docker run --name=mysql -d  -p 3306:3306 \
--mount type=bind,source=/home/ubnutu/mysql,target=/var/lib/mysql \
--mount type=bind,source=/home/ubuntu/my.cnf,target=/etc/my.cnf \
mysql/mysql-server:5.7
```

`-v`태그를 이용하여 바인딩할 때 `:`왼쪽의 경로는 마운팅할 로컬의 경로로써 나는 ubuntu를 사용중이니 위와 같이 지정한 것이다. 

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
docker run --name=mysql -d  -p 3306:3306 \
--mount type=bind,source=/home/ubnutu/mysql,target=/var/lib/mysql \
--mount type=bind,source=/home/ubuntu/my.cnf,target=/etc/my.cnf \
mysql/mysql-server:5.7
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

아래의 내용을 추가했다면 container를 내리고 다시 올리도록 하자. 그리고 다시 mysql 서버에 접속해준다.

<br>

### Slave의 Replication를 위한 유저 계정 만들어 주기

각 Slave에서 Master 에 접속하기 위해서는 Master에서 이들이 접속할 수 있도록 유저 계정을 만들어주어야 한다.

```sql
create user 'repl'@'%' identified by '[패스워드]';
grant replication slave on *.* to 'repl'@'%';
```

그리고 이왕 접속한김에 데이터베이스랑 테이블이나 하나씩 만들어주자.

```sql
create database test;
use test;
create table test_table(
  id int,
  name varchar(50),
  primary key(id)
);
insert into test_table values(1,'kim');
insert into test_table values(2,'lee');
```



### 마스터 Binary Log 좌표 얻기

 Slave가 정확한 지점에서 replication 과정을 시작할 수 있도록 master의 바이너리 로그 안에 최근 지점를 알려줄 필요가 있다. 

만약 데이터 스냅샷을 만들기 위해 마스터를 종료할 계획이라면 이 과정을 건너뛰어도 된다. 단 데이터  스냅샷과 함께 바이너리 로그 인덱스 파일의 복사본을 저장해야한다.

<br>

마스터에서 바이너리 로그 좌표를 얻기 위해서는 아래와 같은 단계들을 따라야 한다.

1. 마스터의 CLI에 접속한다. 그리고 FLUSH TABLES WITH READ LOCK 명령어를 통해서 모든 테이블을 바이너리 로그 파일에 플러시하고 쓰기 명령문을 차단한다.

   ```sql
   FLUSH TABLES WITH READ LOCK;
   ```

   > FlUSH TABLES
   >
   > : 열려있는 테이블들을 닫는 동작을 한다. 테이블을 닫기 전 변경된 것들은 디스크에 flush 되고 걸려있던 모든 락이 해제된다. 
   >
   > FLUSH 명령은 binlog에 기록되게 되는데,  Replication에 경우 위의 FlUSH TABLES WITH READ LOCK이 binlog에 기록되어 slave들이 실행하면 안되므로 몇몇의 명령어들 혹은 추가적인 옵션을 통해서 binlog에 쓰이지 않도록 되어 있다.
   
2. 마스터의 다른 세션에서 SHOW MASTER STATUS 명령어를 사용하여 최근 바이너리 로그 파일 이름과 위치를 본다.

   ```sql
   SHOW MASTER STATUS
   ```

   ![결과창](/img/show_master_status.PNG)

   <br>

   이 이후의 단계에서는 마스터에 기존 데이터가 있는지 여부에 따라 다르다. 아래의 옵션 중 하나를 선택해야한다

   - 기존의 데이터가 있는 경우 잠금을 그대로 유지하도록 클라이언트(세션)를 실행 중인 상태로 둔다. 후에 아래의 `데이터 스냅숏 방식 선택하기`를 이어서 진행합니다.
   - 기존의 데이터가 없이 새로운 마스터와 슬레이브를 세팅 중이라면, 클라이언트를 나감으로 read lock을 해제한다. 후에 아래의 `Replication Slaves 세팅`로 내려가서 진행한다.
   
   <br>

### 데이터 스냅숏 방식 선택하기

만약 마스터 데이터베이스에 기존의 데이터가 존재한다면 이러한 데이터를 각각의 슬레이브에 복사하여야 한다. 마스터 데이터베이스로부터 데이터를 덤프할 여러 방식이 있다.

- mysqldump 툴을 사용하여 복제하려는 모든 데이터베이스의 덤프를 생성하는 법. 이 방법은 `InnoDB`를 사용 중이라면 추천된다.
- 만약 `binary portable files`에 저장중이였다면 원시 데이터 파일을 슬레이브에 복사 할 수 있다. mysqldump보다 더 효율적이며 각 슬레이브에 importing 할 수 있다. 

<br>

#### mysqldump를 이용하여 데이터스냅숏 만들기

shell 에 먼저 접속한다.

```bash
docker exec -it mysql bash
```

아래의 명령어를 통해서 모든 데이터베이스들이`dbdump.db`파일에 덤프시키고, 이 파일에  `change master`명령문을 추가시킬수 있는  `--master-data`라는 옵션을 준다.

```bash
mysqldump --all-databases --master-data -u root -p > dump.sql
```

이제 만들어진 이 파일을 각각의 슬레이브에 importing시켜주면 된다.

도커를 사용하고 있으니.. 쉘에서 빠져나와 docker cp 문을 사용하여 복사해주면 되겠다.

```bash
docker cp mysql:dump.sql .
```

<br>

#### Rasw Data Files를 이용하여 데이터스냅숏 만들기

[작성 예정...](https://dev.mysql.com/doc/refman/5.7/en/replication-snapshot-method.html)

<br>

작성 예정..

### Replication Slaves 세팅

<br>

#### 구조 변경

![mysql config tree](/img/mysql_config_tree.PNG)

slave를 위한 구조를 다시 만들어보겠다. 위의 사진과 같은 구조가 되어야한다.

`mysql_config` 폴더를 만들고 그 안에 `slave`, `master`를 만든다. 기존의 master를 실행시킬 때 이용했던 `my.cnf`와 `mysql` 폴더는 master폴더안에 넣어준다. 그리고 slave안에는 mysql 폴더를 하나 만들고 master의 my.cnf를 복사하여 넣어준다.

그리고 slave안에 있는 `my.cnf`의 내용에 아래의 내용을 변경시키겠다. 위에서 설명했듯이 각 데이터베이스는 <u>유니크한 서버 아이디</u>를 가져야 함으로` server-id`를 2로 변경시켰다.

```bash
server-id=2
skip-slave-start
```

slave 데이터베이스를 실행시키기 위해서는 아래의 명령어를 이용한다.

```bash
MYSQL_HOME=${HOME}/mysql_config

docker run --name=slave_mysql -d -p 3316:3306 \
--mount type=bind,source=${MYSQL_HOME}/slave/mysql,target=/var/lib/mysql \
type=bind,source=${MYSQL_HOME}/slave/my.cnf,target=/etc/my.cnf \
mysql/mysql-server:5.7
```

그리고 master 를 실행시키기 위해서 아래의 명령어로 바꾼다.

```shell
MYSQL_HOME=${HOME}/mysql_config

docker run --name=master_mysql -d -p 3306:3306 \
--mount type=bind,source=${MYSQL_HOME}/master/mysql,target=/var/lib/mysql \
type=bind,source=${MYSQL_HOME}/master/my.cnf,target=/etc/my.cnf \
mysql/mysql-server:5.7
```

<u>slave 를 실행시키고 비밀번호 바꿔주는 거 잊지 말고!</u>

#### replication을 위한 master세팅

위에서 dump한 파일을 slave에 적용하기 위해 아까 dump해놓은 `dbdump.db`파일을 slave에 복사해주자.

```bash
docker cp dump.sql slave_mysql:.
```

다음 `slave_mysql`의 `bash` 에 접속한다.

```bash
docker exec -it slave_mysql bash
```

`dbdump.db`파일을 적용시킨다.

```bash
mysql -u root -p < dump.sql
```

다음으로 `mysql`에 접속하여 slave가 replication을 위해 master와 통신하기 위해 필수적으로 연결 정보를 설정해야 한다. master 클라이언트에 접속하여 `show master status;`명령어를 통해 `파일 이름` & `pos`를 확인하고 slave 클라이언트에 접속하여 아래와 같이 연결 정보를 설정한다.

```sql
CHANGE MASTER TO
MASTER_HOST='master_host_name',
MASTER_USER='replicatin_user_name',
MASTER_PASSWORD='replication_password',
MASTER_LOG_FILE='recorded_log_file_name',
MASTER_LOG_POS=recorded_log_position;
```

```sql
start slave;
```

이제 아래의 명령을 통해 slave의 상태를 확인한다.

```sql
show slave status\G;
```

![slave status](/img/show_slave_status.PNG)

> 상태창을 보면서 error가 난 부분은 없는지 확인하길 바란다. 



slave가 정상적으로 master와 연결되었는지 확인을 해보기 위해  master로 들어가서 데이터를 삽입하고, slave에서 이를 확인하는 작업을 해보자.

먼저 마스터 `mysql`에 들어가서

```sql
insert into test_table values(3,'lim');
```

다음으로 slave의 `mysql`에 들어가서

```sql
select * from test_table;
```

위의 명령어를 쳤을 때, 적용이 되어 있지 않다면 제대로 설정이 안되있는 것이다.

![replication result](/img/replication_result.PNG)

