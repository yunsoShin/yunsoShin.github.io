---
layout: post
title: "AWS Docker 컨테이너의 MariaDB 데이터베이스 덤프 파일 생성 및 복사"
date: 2024-05-21
categories: [AWS, Docker, MariaDB, Backup]
---

# AWS Docker 컨테이너의 MariaDB 데이터베이스 덤프 파일 생성 및 복사

## 개요

이 블로그 글에서는 AWS 인스턴스의 Docker 컨테이너에서 MariaDB 데이터베이스 덤프 파일을 생성하고, 이를 호스트 로컬 폴더로 복사하는 방법을 단계별로 설명합니다.

SSH로 공유된 원격 인스턴스에 접근하여 인스턴스 내부에서 돌아가는 DB환경을 로컬로 재구성하기위해서 필요한 작업을 합니다.

## 1. Docker 컨테이너 내부에서 MariaDB 덤프 파일 생성

먼저 Docker 컨테이너 내부에 접속하여 MariaDB 데이터베이스 덤프 파일을 생성합니다.

### 컨테이너 이름을 사용하는 방법:

```bash
sudo docker exec -it [container_name] /bin/bash
```

### 컨테이너 ID를 사용하는 방법:

```bash
sudo docker exec -it [container_id] /bin/bash
```

컨테이너 내부에서 mysqldump 명령어를 실행하여 덤프 파일을 생성합니다:

```bash
mysqldump  -uroot -p [password] [database] > /tmp/[dumpFileName].sql
```

## 열 통계 정보 포함 비활성화:

MySQL 8.0.21부터 mysqldump는 기본적으로 열 통계 정보를 덤프 파일에 포함시킵니다. 그러나 일부 MariaDB 버전 또는 특정 설정에서는 이 기능이 호환되지 않을 수 있습니다. 따라서 이 옵션을 사용하면 열 통계 정보를 포함하지 않도록 설정할 수 있습니다.

만약 mariaDB를 사용하는 환경이고 , 아래와같은 에러가 발생한다면 열통계를 끄는 옵션을 추가하여주세요

```bash
mysqldump: Couldn't execute 'SELECT COLUMN_NAME,JSON_EXTRACT(HISTOGRAM, '$."number-of-buckets-specified"')  FROM information_schema.COLUMN_STATISTICS                WHERE SCHEMA_NAME = '[DatabaseName]' AND TABLE_NAME = '[TableName]';': Unknown table 'COLUMN_STATISTICS' in information_schema (1109)
```

이 오류는 mysqldump 명령어가 information_schema 데이터베이스의 COLUMN_STATISTICS 테이블을 찾지 못해서 발생합니다. 이 문제는 MariaDB와 MySQL 버전 차이로 인해 발생할 수 있습니다.

현재 우리의 DB구성은 mariaDB였기에 버전호환성 문제를 일으키는 것 같습니다.

이를 해결하기 위해 --column-statistics=0 옵션을 추가하여 COLUMN_STATISTICS 기능을 비활성화할 수 있습니다. 다음은 명령어 예제입니다:

```bash
mysqldump --column-statistics=0 -uroot -p [password] [database] > /tmp/[dumpFileName].sql
```

## 2. 덤프 파일 존재 확인

컨테이너 내부에서 덤프 파일이 잘 생성되었는지 확인합니다:

```bash
ls -al /tmp/
```

### 예상 출력 예시:

```plaintext
root@[container_id]:/# ls -al /tmp/
total 12
drwxrwxrwt  2 root root 4096 May 21 12:00 .
drwxr-xr-x 21 root root 4096 May 21 12:00 ..
-rw-r--r--  1 root root 2048 May 21 12:00 [dumpedFileName].sql
```

## 3.컨테이너에서 호스트로 덤프 파일 복사

덤프파일이 생성된 컨테이너의 임시볼륨에 접근하고 SQL파일을 특정경로로 복사해줍니다

```bash
sudo docker cp [container_id or container_name]:/tmp/[dumpedFileName].sql /home/ubuntu/[copyFileName].sql
```

또는 현재 디렉토리에 복사하려면 다음과 같이 합니다:

```bash
sudo docker cp 615ff7727e34:/tmp/backup0521.sql .
```

요약
이 과정을 통해 AWS 인스턴스의 Docker 컨테이너에서 MariaDB 데이터베이스 덤프 파일을 생성하고, 이를 호스트 머신으로 복사할 수 있습니다. 아래는 전체 과정의 요약입니다:

- Docker 컨테이너 내부에 접속
- mysqldump 명령어를 사용하여 덤프 파일 생성
- ls -al 명령어로 덤프 파일 존재 확인
- docker cp 명령어로 덤프 파일을 호스트로 복사
