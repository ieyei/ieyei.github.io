---
layout: post
title:  "repmgr를 이용하여 postgresql 이중화 구성"
---

### repmgr (Replication Manager for PostgreSQL Clusters)

------

repmgr은 PostgreSQL 클러스터의 replication과 failover를 관리해 주는 open-source 툴입니다. 

PostgreSQL 기술 지원, 개발, 컨설팅을 수행하는 **2ndQuadrant** 사에서 개발되었으며, GNU Public License(GPL) v3 라이선스를 따르고 있습니다. 또한 repmgr 버전 4 이후부터는 PostgreSQL extension으로 설치가 되어 매우 가볍고 사용이 쉽습니다.

##### 주요기능

- standby 서버 구성
- replication 모니터링
- failover
- 수동 switchover

##### 참조 사이트

<https://repmgr.org/>



### DB 이중화 아키텍처

------

##### 이중화 구성

- 개요

  - PostgreSQL의 Streaming Replication 기능을 사용하여 DB 이중화 구성
  - 장애발생시 장애감지, 자동 failover 수행을 위하여 오픈소스소프트웨어인 repmgr을 사용
  - repmgrd 데몬 프로세스로 운영DB의 정상여부 모니터링 수행
  - Primary 노드에서 모든 트랜잭션 처리
  - PostgreSQL JDBC 사용하여 connection pool 구성
    - 2개 노드의 IP, port를 URL에 입력 (예:jdbc:postgresql://10.1.10.14:5432,10.1.10.16:5432/postgres)
    - db 접속시 첫번째 서버 부터 접속을 시도하기 때문에 failover 시  Standby 노드로 접속을 방지하기 위하여 connection pool 크기의 min, max 를 동일하게 설정
    - WAS 제품에 따라 Primary 노드 장애시 감지가 되도록 connection pool 설정 필요

- 구성도

  ![구성도](ieyei.github.io/_pics/구성도.png)

##### Streaming Replication

- 개요
  - PostgreSQL 은 WAL(Write Ahead Log)을 사용하여 복제 환경 구성 지원
  - WAL 파일 생성 후 전달하는 WAL Archive Shipping 방식과 WAL 레코드 발생시 전달하는 WAL Streaming 방식 사용
  - 데이터 손실 위험이 상대적으로 적은 WAL Streaming 방식 사용
- 고려사항
  - Standby 노드로 WAL Streaming 완료 전 Primary 노드 장애시 데이터 손실 가능성 존재
  - Standby 노드의 복제가 늦어지지 않도록 Replication 상황 모니터링 필요(늦어진 시간만큼 데이터 손실 가능)
  - 장애감지 및 자동 Failover 기능 수행하지 않음

### 이중화 시스템 테스트

------

- **이중화 구성**

1. repmgr, repmgrd 설치 및 구성

2. 구성 절차

   1. repmgr.conf 설정[primary(10.1.10.14), standby(10.1.10.16)]

   2. Primary 서버 등록[primary(10.1.10.14)]

      ```
      $ repmgr primary register
      ```

   3. Standby 서버 구성[standby(10.1.10.16)]

      ```
      $ repmgr -h 10.1.10.14 -U repmgr -d repmgr standby clone
      ```

   4. Standby 서버 기동[standby(10.1.10.16)]

      ```
      $ pg_ctl -D /dbms/data/pgdata start
      ```

   5. Standby 서버 등록[standby(10.1.10.16)]

      ```
      $ repmgr standby register
      ```

      ​


TEST

- Failover 테스트

  1. primary stop [primary(10.1.10.14)]

     ```
     $ pg_ctl -D /dbms/data/pgdata -m immediate stop
     ```

  2. 새로운 standby 서버 구성[old primary(10.1.10.14)]

     ```
     $ repmgr -h 10.1.10.16 -p 5432 -U repmgr -d repmgr standby clone -F
     ```

  3. 새로운 standby 서버 start 및 확인 [old primary(10.1.10.14)]

     ```
     $ pg_ctl -D /dbms/data/pgdata start repmgr cluster show
     ```

  4. standby 서버로 재등록 및 확인 [old primary(10.1.10.14)]

     ```
     $ repmgr standby register -F            
     $ repmgr cluster show
     ```

     ​

- switchover 테스트

  1. 사전점검 [standby(10.1.10.14), primary(10.1.10.16)]

     ```
     $ repmgr node check
     ```

  2. switchover [standby(10.1.10.14)]

     ```
     $ repmgr standby switchover

     ```

  3. status check

     ```
     $ repmgr cluster show
     ```

     ​




