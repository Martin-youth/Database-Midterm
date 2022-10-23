# Database-Midterm

* CDC(Change Data Capture)란?    
  데이터베이스에서 데이터 변경 발생 시 변경된 데이터를 사용하여 동작을 취할 수 있도록 지원하는 기능이다.


  ```
* Debezium이란?
  Apache Kafka를 기반으로 구축되었으며, 
  특정 DBMS를 모니터링하는 Kafka Connect 호환 커넥터를 제공하기 위해 시작된 프로젝트이다.
  다양한 DBMS의 변경 사항을 캡쳐하고 유사한 구조의 변경 이벤트를 produce 하는 커넥터 라이브러리를 구축한다.
  ```

# Architecture

* Spring Boot Module
    * Debezium Engine을 기반으로 CDC 기능 구현
    * Metadata 및 Task, Step, Flow 등의 개체 관리
    * 테이블 및 컬럼 매핑 정보 관리 및 Go Module로 송신
    * CDC 로그를 json 형태로 Go Module로 송신

* Go Module
    * 테이블 및 컬럼 매핑 정보 컴파일
    * 타겟 데이터베이스 로드 기능

* MetaDB : MariaDB
    * CDC에 필요한 Property 저장
    * Metadata 및 Task, Step, Flow 등의 개체 정보 저장
    * 테이블 및 컬럼 매핑 정보 및 history 저장
    * CDC 운영 로그 저장

* Frontend : Vue.js
    * Metadata 및 Task, Step, Flow 입력 (현재 미구현으로 Swagger로 작업 가능)
    * 기본 매핑 및 컴파일 수행
    * CDC 시작/중단/재시작 수행


## CDC Converter Backend - Spring Boot
* openjdk1.8
* Exception
    * 사용자 정의 Exception 경우 UserExceptionHandler 에서 잡아주지 않으면 GlobalExceptionHandler에서 공통처리
* Sample
    * http://localhost:8080/swagger-ui.html 접속하여 REST API 확인 및 테스트
    * 기본적인 Control - Service - Repository 형태이며 리턴은 BaseResponse 로 통일

## CDC Converter Backend - Go
* 소스: https://github.kakaocorp.com/solution-delivery/etl_converter_go

* Import Library

    ```
    go get github.com/dailyburn/ratchet
    go get github.com/go-sql-driver/mysql
    go get github.com/gorilla/mux
    go get github.com/albrow/forms
    ```

## ETL Converter Frontend - Vue.js
* frontend 폴더
    * node.js v14.15.4
    * @vue/cli 4.5.10
    * 실행 방법
        * local : yarn local
        * dev : yarn dev
        * prod :yarn prod
* 개발 현황
    * Task, Metadata 목록 (좌측 palette)
    * Task control 버튼 (workspace의 상부 tarbar)
    * 중앙 Canvas 화면은 미구현으로 샘플 화면으로만 고정되어 있음

# CDC 사전 설정 작업
* MySQL
    1. CDC 설정
        * bin log 설정 및 binlog_format = ROW 필요
        * 설정 예

            ```
            [mysqld]
            server-id=1
            log-bin=/log/mysql-log/bin.log
            binlog_format = ROW
            binlog_row_image = FULL
            binlog_cache_size=2M
            max_binlog_size=512M
            expire_logs_days=7
            ```
        * 설정 후 조회
            ```
            show variables like 'log_bin’;
            SHOW MASTER STATUS;
            SELECT @@binlog_format;
            ```
    2. 권한 할당
        * CDC 작업에 사용할 계정 생성 및 필요한 권한 할당
        * 권한 할당 예
            ```
            CREATE USER 'etl_user'@'%' IDENTIFIED BY 'converter';
            GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'etl_user'@'%';
            ```
            
# Tutorial

1. 개요

   본 Tutorial에서는 미리 생성해 둔 테스트용 MySQL DB (Docker Container명 : source-db)를 소스로 하여 Straw 또는 MySQL을 타겟으로 하는 CDC 작업을 구성하고 실행해 보고자 한다.

   CDC 실행 중 소스DB에서 DML이나 DDL을 실행하여 타겟에서 곧바로 반영되는 것을 확인할 수 있다.

   아쉽지만 아직 Frontend 개발이 미진하여 Swagger를 이용하여 Task와 그 하위 구성요소인 Step, Flow를 생성해야 한다.

   현재 Transform 지원 기능으로는 필터링, 조인, API 호출, SQL 실행, custom mapping 등 다양한 변환 기능을 지원하나, 본 Tutorial에서는 자동 매핑 기능으로 진행하도록 한다.

2. 사전 준비

    1. Docker 기반으로 진행되므로 미리 Docker 환경이 준비되어 있어야 함

3. KIC Docker Registry Login & image download

   kakao i cloud Docker Registry에 각 모듈의 이미지가 업로드되어 있으므로 로그인하여 이미지를 다운로드 받는다.

    ```
    docker login demo-cdc-converter.kr-central-1.kcr.dev --username 5e1e82585848493db299e578f4095b55 --password R22rlwCix2pezb9FPcFuCWIG0olWNhxY-ZOu46Qn71gB8iMOn3e87tOT82YALe9mdL_9XWm_aKC2DORt4hAVrw
    docker pull demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-spring:v0.1
    docker pull demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-go:v0.1
    docker pull demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-db:v0.1
    docker pull demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-front:v0.1
    docker pull demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-test-mysql:v0.1   
    ```

4. Docker 실행

   이미지를 다운로드받은 후 각 이미지로 컨테이너를 실행함   
   서로 link가 필요하기 때문에 순서대로 수행해야 함

    ```
    docker run -d -p 3307:3306 --name source-db demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-test-mysql:v0.1
    docker run -d -p 3306:3306 --name converter-db demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-db:v0.1
    docker run -d -p 3000:3000 --name converter-go --link source-db  demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-go:v0.1
    docker run -d -p 8080:8080 --name converter --link converter-db:converter-db-svc --link converter-go:converter-go-svc --link source-db demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-spring:v0.1
    docker run -d -p 9000:9000 --name converter-front --link converter:converter-spring-svc demo-cdc-converter.kr-central-1.kcr.dev/repo-cdc-converter/converter-front:v0.1
    ```

5. 서비스 확인

    * Frontend 접속

      링크: http://localhost:9000/#/flow

      왼쪽 탭 > Task > Sample Task 존재 확인

    * Swagger 접속

      링크: http://localhost:8080/swagger-ui.html

    * 참조 : docker logs를 이용하여 각 모듈의 로그를 확인하며 진행하면 편리합니다!

      사용 예
        ```
        docker logs converter -f
        ```

6. Metadata 생성

   Swagger > 7. Metadata Management > /api/v1/metadata/create 실행 > metadataMap 입력

    * RDBMS 생성
        ```
        {
        "metadataName": "MySQL Test",
        "metadataDesc": "따라하기 테스트 - MySQL",
        "dataTypeId":"1",
        "hostname":"source-db",
        "port":"3306",
        "user":"etl_user",
        "password":"converter"
        }
        ```
      별도 소스 DB 사용하려면 hostname, port, user, password에 적절한 값으로 변경


    Metadata를 생성한 후, frontend 화면 (http://localhost:9000/#/flow) 에서 페이지 새로고침을 한 후 왼쪽 탭 > Metadata 를 클릭하면 현재까지 생성한 Metadata 확인 가능

7. Task & Step & Flow 생성
    * MySQL -> MySQL 테스트 수행 시

      MySQL을 타겟으로 하는 경우는 기본적으로 생성한 converter-test-mysql 이미지의 DB에
      target_db를 미리 생성해 두어 이를 사용함 (이미지에는 생성되어 있음)
        1. 타겟DB 생성 확인
        ```
      docker exec -it source-db bash 
      mysql -u root -p converter
      mysql> show databases; 
        ```
      다른 DB를 사용하고자 할 때에는 미리 metadata를 생성하여 해당 metadataId와 database 값을 이용하도록 함
        2. Task 생성

           Swagger > 1. Task Management > /api/v1/task/create 실행 > task 입력

           Response에 return된 Task ID를 확인하고 다음 단계에서 사용함

              ```
              {
              "taskName": "CDC 따라하기 test",
              "taskDesc": "MySQL → MySQL",
              "folderId": 1,
              "taskTypeCd": 1,
              "versionId": "0.1"
              }
              ```
           Task 생성 후, front 화면을 새로고침 후 왼쪽에서 Task를 눌러 방금 생성한 Task를 확인 가능함

        3. Step 생성

           Swagger > 2. Step Management > /api/v1/step/create 실행 > stepMap 입력

           기본적으로 소스 Step, 타겟 Step, 매퍼 Step의 3개 Step을 생성함

           각 Response에 return된 Step ID를 확인하고 다음 단계에서 사용함
            1. Source CDC Step 생성
                ```
                {
                "stepName": "source db - MySQL",
                "stepDesc": "따라하기 테스트",
                "taskId": "2",
                "versionId": "0.1",
                "componentId": "1",
                "metadataId": "1",
                "database":"source_db"
                }
                ```
               taskId는 앞 단계에서 받은 Task ID 값을 이용함
            2. Target : MySQL 생성
                ```
                {
                "stepName": "target db - MySQL",
                "stepDesc": "따라하기 테스트",
                "taskId": "2",
                "versionId": "0.1",
                "componentId": "3",
                "metadataId": "1",
                "database":"target_db"
                }
                ```
               taskId는 앞 단계에서 받은 Task ID 값을 이용함
               별도 타겟 이용 시에는 미리 metadata 생성 후 해당 metadataId 및 database 이용
            3. Mapper Step 생성
                ```
                {
                "stepName": "Test Mapper",
                "stepDesc": "따라하기 테스트",
                "taskId": "2",
                "versionId": "0.1",
                "componentId": "5"
                }
                ```
               taskId는 앞 단계에서 받은 Task ID 값을 이용함
        4. Flow 생성

           Swagger > 3. Flow Management > /api/v1/flow/create 실행 > flow 입력

           step 간의 선후관계를 정의하는 것으로, 소스 -> 매퍼, 매퍼 -> 타겟 Flow를 생성함
            1. 소스 -> 타겟 연결
                ```
                {
                "taskId": 2,
                "beforeStepId": 1,
                "afterStepId": 3,
                "flowName": "소스-매퍼"
                }
                ```
               beforeStepId와 afterStepId는 앞 단계에서 생성한 Step Id 사용
            2. 매퍼 -> 타겟 연결
                ```
                {
                "taskId": 2,
                "beforeStepId": 3,
                "afterStepId": 2,
                "flowName": "매퍼-타겟"
                }
                ```
               beforeStepId와 afterStepId는 앞 단계에서 생성한 Step Id 사용

8. Mapping & Compile 수행

   이 단계부터는 frontend 화면 (http://localhost:9000/#/flow) 에서 수행함

    1. Mapping 수행

       해당 작업은 소스 DB에서 전체 테이블에 대해 소스-타겟 동일 매핑을 수행하고 저장함

       예외) 타겟이 Straw인 경우에는 id 컬럼이 단일 컬럼만 지원하기 때문에, multi PK 테이블의 경우에는 이를 합성한 별도 컬럼을 생성하여 매핑 수행

       왼쪽 Task 목록에서 원하는 Task를 선택하여 위쪽 탭에서 [Auto Mapping] 버튼 클릭

    2. Compile 수행

       mapping을 수행한 후, Go module에서 이 mapping 정보를 컴파일함

       CDC 로그 발생 시 이 컴파일 정보를 기반으로 변환하여 타겟으로 송신하도록 함

       위쪽 탭에서 [Compile] 버튼 클릭 -> 1분 이내 시간 소요되므로 잠시 기다리세요~

       참조 : docker logs를 활용하여 완료 로그를 볼 수 있습니다!

       이 단계가 완료되면 타겟이 Straw인 경우, Straw Console에서 소스 테이블에 대응하는 Type이 생성된 것을 볼 수 있음

9. CDC 실행 및 DML, DDL 테스트 수행

    1. CDC 실행

       Mapping & Compile까지 완료되면 위쪽 탭에서 [CDC Start] 버튼을 클릭하여 CDC 작업을 시작함

       1~2초 이내에 타겟 쪽에서 동일 데이터가 적재된 것을 확인 가능함

       타겟이 MySQL인 경우에는 DB 접속해서 조회해 보고, Straw인 경우에는 postman 등을 활용하여 데이터 확인 가능

    2. DML 테스트

       소스 DB에 접속하여 데이터 변경(insert, update, delete)을 수행하여 타겟 쪽에 반영됨을 확인해 보자.

        * 소스 DB 접속 예
        ```
        docker exec -it source-db bash
        mysql -uroot -pconverter
        ```
        * DML 수행 예
        ```MySQL
        use source_db;
        update tb_job set job_title='SES' where job_title = 'SE';
