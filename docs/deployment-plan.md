# 배포계획서

## 1. 배포 개요

본 프로젝트는 Spring Boot 기반 MSA(Microservices Architecture) 구조로 개발되었으며, Docker Compose를 이용하여 각 서비스를 컨테이너 단위로 배포합니다.

배포 환경은 AWS EC2 서버를 기반으로 구성하며, Jenkins를 CI/CD 도구로 사용하여 소스 코드 빌드, Docker 이미지 생성 및 Docker Registry 업로드를 자동화합니다.

배포 대상 서비스는 다음과 같습니다.

- Frontend
- Gateway Service
- Eureka Service
- Config Service
- Account Service
- Transaction Service
- Notification Service
- Kafka
- Schema Registry
- Kafka UI
- Account Service용 MariaDB
- Transaction Service용 MariaDB
- Notification Service용 MariaDB

각 마이크로서비스는 독립적인 Docker 컨테이너로 실행하며, 서비스별 데이터베이스를 분리하여 MSA의 데이터 독립성을 유지합니다.

---

## 2. 배포 환경

### 2.1 서버 환경

배포 서버는 AWS EC2를 사용합니다.

- Cloud: AWS
- Compute: EC2
- Container Runtime: Docker
- Container Management: Docker Compose
- CI/CD: Jenkins
- Container Registry: Docker Hub
- OS: Linux

Jenkins는 프로젝트의 Git Repository를 기준으로 빌드 및 배포 작업을 자동화합니다. 

> AWS 서버의 메모리가 부족하여 개인 서버에서 Jenkins 컨테이너를 만들어서 프로젝트를 진행하였습니다.

---

## 3. 배포 아키텍처

전체 배포 구조는 다음과 같습니다.

```text
                         사용자
                           │
                           ▼
                    ┌─────────────┐
                    │   Frontend  │
                    │   Nginx     │
                    │    :80      │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Gateway   │
                    │    :8080    │
                    └──────┬──────┘
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
       Account Service  Transaction   Notification
             │           Service         Service
             │             │               │
             ▼             ▼               ▼
         MariaDB       MariaDB          MariaDB
         
                           │
                           │ Kafka
                           ▼
                    ┌─────────────┐
                    │    Kafka    │
                    │    :9092    │
                    └──────┬──────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Schema Registry │
                  │      :8081      │
                  └─────────────────┘


       ┌─────────────────┐      ┌─────────────────┐
       │ Eureka Service  │      │ Config Service  │
       │      :8761      │      │      :8888      │
       └─────────────────┘      └─────────────────┘
```

모든 컨테이너는 Docker Compose를 통해 동일한 Docker 네트워크에서 실행되며, 컨테이너 이름을 이용하여 서로 통신합니다.

---

## 4. Docker 이미지 구성

애플리케이션 서비스는 Docker Image로 패키징하여 Docker Hub에 저장합니다.

이미지는 다음과 같은 형식으로 관리합니다.

```text
uniquecolor/frontend:1.0.1
uniquecolor/gateway-service:1.0.0
uniquecolor/eureka-service:1.0.0
uniquecolor/config-service:1.0.0
uniquecolor/account-service:1.0.0
uniquecolor/transaction-service:1.0.0
uniquecolor/notification-service:1.0.0
```

Kafka, Schema Registry, MariaDB 등 외부 오픈소스 이미지는 Docker Hub의 공식 또는 공개 이미지를 사용합니다.

```text
confluentinc/cp-kafka:7.6.1
confluentinc/cp-schema-registry:7.6.1
provectuslabs/kafka-ui:latest
mariadb:latest
```

애플리케이션 이미지는 Jenkins를 통해 빌드하고 Docker Hub에 Push 합니다.

---

## 5. CI/CD 구성

본 프로젝트에서는 Jenkins를 CI/CD 서버로 사용합니다.

전체적인 CI/CD 과정은 다음과 같습니다.

```text
Git Push
   │
   ▼
Jenkins
   │
   ├── Source Code Checkout
   │
   ├── Application Build
   │
   ├── Test
   │
   ├── Docker Image Build
   │
   ├── Docker Hub Login
   │
   └── Docker Image Push
           │
           ▼
       Docker Hub
           │
           ▼
       AWS EC2 (또는 개인 서버)
           │
           ├── Docker Pull
           │
           └── Docker Compose Up
                   │
                   ▼
              서비스 배포
```

---

## 6. CI 단계

### 6.1 소스 코드 Checkout

Jenkins가 Git Repository에서 최신 소스 코드를 가져옵니다.

```bash
git checkout
```

Jenkins Pipeline에서 Repository와 인증 정보를 설정하여 자동으로 소스 코드를 가져오도록 구성합니다.

---

### 6.2 애플리케이션 빌드

각 Spring Boot 서비스의 소스 코드를 빌드합니다.

각 모듈이 Maven을 사용하기 때문에 다음 명령어를 사용합니다.

```bash
mvn clean compile package -DskipTests=true
```

빌드 과정에서 컴파일 오류가 발생하면 이후 단계로 진행하지 않습니다.

---

## 7. Docker Image Build

빌드와 테스트가 성공하면 각 서비스의 Docker Image를 생성합니다.

예:

```bash
docker build -t uniquecolor/account-service:1.0.0 ./account-service
docker build -t uniquecolor/transaction-service:1.0.0 ./transaction-service
docker build -t uniquecolor/notification-service:1.0.0 ./notification-service
```

Frontend 역시 Docker Image로 생성합니다.

```bash
docker build -t uniquecolor/miniproject-frontend:1.0.1 ./fin-account-msa-frontend
```

Gateway, Eureka, Config Server도 동일한 방식으로 Docker Image를 생성합니다.

---

## 8. Docker Image Push

Jenkins에서 Docker Hub 인증 후 생성된 Image를 Registry에 Push 합니다.

```bash
docker login
```

이후 각 Image를 Push 합니다.

```bash
docker push uniquecolor/account-service:1.0.0
docker push uniquecolor/transaction-service:1.0.0
docker push uniquecolor/notification-service:1.0.0
docker push uniquecolor/gateway-service:1.0.0
docker push uniquecolor/eureka-service:1.0.0
docker push uniquecolor/config-service:1.0.0
docker push uniquecolor/miniproject-frontend:1.0.1
```

Jenkins에는 Docker Hub 계정 정보를 직접 작성하지 않고 Jenkins Credentials를 이용하여 관리합니다.

---

## 9. AWS EC2 배포

Docker Image Push가 완료되면 Jenkins에서 AWS EC2 배포 서버에 접속하여 최신 Image를 가져옵니다.

EC2 서버에서는 다음 명령을 수행합니다.

```bash
docker compose pull
```

이 명령을 통해 Docker Compose에 정의된 최신 Image를 Docker Hub에서 가져옵니다.

이후 기존 컨테이너를 새로운 Image 기준으로 재생성합니다.

```bash
docker compose up -d
```

배포 후 컨테이너 상태를 확인합니다.

```bash
docker compose ps
```

필요한 경우 다음 명령으로 컨테이너 로그를 확인합니다.

```bash
docker compose logs
```

특정 서비스의 로그만 확인할 수도 있습니다.

```bash
docker compose logs gateway-service
docker compose logs account-service
docker compose logs transaction-service
```

---

## 10. Docker Compose 배포

AWS EC2에서는 제공된 `docker-compose.yml`을 이용하여 전체 시스템을 관리합니다.

주요 애플리케이션 서비스는 다음과 같습니다.

```text
frontend
gateway-service
eureka-service
config-service
account-service
transaction-service
notification-service
```

인프라 구성요소는 다음과 같습니다.

```text
kafka
schema-registry
kafka-ui
```

각 서비스의 데이터베이스는 다음과 같이 독립적으로 구성합니다.

```text
mariadb-account-service
mariadb-transaction-service
mariadb-notification-service
```

이를 통해 Account Service, Transaction Service, Notification Service가 서로 다른 데이터베이스를 사용하도록 구성합니다.

---

## 11. 서비스 기동 순서

서비스 간 의존성을 고려하여 다음 순서로 기동하는 것을 기본으로 합니다.

```text
MariaDB
   │
   ▼
Kafka
   │
   ▼
Schema Registry
   │
   ├───────────────┐
   ▼               ▼
Eureka          Config Server
   │
   ▼
Account Service
Transaction Service
Notification Service
   │
   ▼
Gateway
   │
   ▼
Frontend
```

특히 Transaction Service와 Notification Service는 Kafka 및 Schema Registry가 정상적으로 실행된 이후 기동하도록 Docker Compose의 `depends_on` 및 Health Check를 활용합니다.

---

## 12. 환경 변수 관리

서비스의 실행 환경에 따라 변경되는 값은 환경 변수로 관리합니다.

예를 들어 Gateway는 다음과 같은 환경 변수를 사용합니다.

```text
eureka.client.service-url.defaultZone
CORS_ALLOWED_ORIGIN
```

Transaction Service는 다음과 같은 환경 변수를 사용합니다.

```text
eureka.client.service-url.defaultZone
spring.kafka.bootstrap-servers
spring.kafka.properties.schema.registry.url
spring.datasource.url
```

Notification Service 역시 Kafka, Schema Registry 및 MariaDB 접속 정보를 환경 변수로 관리합니다.

데이터베이스 비밀번호, Docker Hub 인증정보 등의 민감정보는 Git Repository에 직접 저장하지 않고 Jenkins Credentials 또는 서버 환경 변수 등을 사용하여 관리합니다.

---

## 13. 배포 검증

배포 완료 후 다음 항목을 검증합니다.

### 13.1 컨테이너 상태 확인

```bash
docker compose ps
```

모든 필수 서비스가 정상적으로 실행되고 있는지 확인합니다.

---

### 13.2 Eureka 등록 확인

Eureka Server에 접속하여 다음 서비스가 정상적으로 등록되었는지 확인합니다.

```text
ACCOUNT-SERVICE
TRANSACTION-SERVICE
NOTIFICATION-SERVICE
GATEWAY-SERVICE
```

---

### 13.3 Gateway API 확인

Frontend 또는 API Client를 통해 Gateway의 API를 호출합니다.

예:

```text
POST /transactions/deposit
POST /transactions/withdraw
POST /transactions/transfer
```

정상적인 HTTP 응답이 반환되는지 확인합니다.

---

### 13.4 Kafka 동작 확인

Transaction Service에서 발생한 이벤트가 Kafka를 통해 정상적으로 전달되는지 확인합니다.

```text
Transaction Service
        │
        ▼
      Kafka
        │
        ▼
Notification Service
```

Kafka UI를 이용하여 Topic과 Message가 정상적으로 생성 및 전달되는지 확인합니다.

---

### 13.5 데이터베이스 확인

각 서비스가 자신의 데이터베이스에 정상적으로 접근하는지 확인합니다.

```text
Account Service
      │
      ▼
mariadb-account-service

Transaction Service
      │
      ▼
mariadb-transaction-service

Notification Service
      │
      ▼
mariadb-notification-service
```

---

## 14. 배포 실패 대응

배포 과정에서 다음과 같은 문제가 발생할 경우 배포를 중단합니다.

* 애플리케이션 빌드 실패
* 테스트 실패
* Docker Image Build 실패
* Docker Image Push 실패
* EC2 접속 실패
* Container 실행 실패
* Health Check 실패
* 서비스 간 통신 실패

CI 단계에서 오류가 발생한 경우 해당 버전의 Image를 배포하지 않는다.

배포 이후 문제가 발생한 경우 Docker Container 로그를 확인하여 원인을 분석합니다.

```bash
docker compose logs <service-name>
```

---

## 15. 롤백 계획

새로운 버전의 배포 후 서비스 장애가 발생하면 이전에 검증된 Docker Image 버전으로 롤백합니다.

예를 들어 기존 버전이 다음과 같습니다고 가정합니다.

```text
account-service:1.0.0
```

새로운 버전을 다음과 같이 배포합니다.

```text
account-service:1.0.1
```

배포 후 문제가 발생하면 `docker-compose.yml`에서 이전 버전으로 변경합니다.

```yaml
account-service:
  image: uniquecolor/account-service:1.0.0
```

이후 다음 명령을 실행합니다.

```bash
docker compose pull account-service
docker compose up -d account-service
```

다른 서비스 역시 동일한 방식으로 이전 버전 Image를 사용하여 복구합니다.

---

## 16. 배포 완료 기준

다음 조건을 모두 만족하면 배포가 완료된 것으로 판단합니다.

- 모든 필수 Docker Container가 정상적으로 실행된다.
- Eureka에 마이크로서비스가 정상적으로 등록된다.
- Gateway를 통한 API 호출이 정상적으로 동작한다.
- Account Service와 Database 간 통신이 정상적으로 동작한다.
- Transaction Service와 Database 간 통신이 정상적으로 동작한다.
- Kafka Message가 정상적으로 생성 및 전달된다.
- Notification Service가 Kafka Message를 정상적으로 처리한다.
- Frontend에서 주요 기능을 정상적으로 사용할 수 있다.
- 주요 기능에 대한 통합 테스트가 정상적으로 완료된다.

---

## 17. 최종 배포 절차

최종 배포 과정은 다음과 같이 수행합니다.

```text
개발자
  │
  │ Git Push
  ▼
Jenkins
  │
  ├── Checkout
  │
  ├── Build
  │
  ├── Test
  │
  ├── Docker Image Build
  │
  └── Docker Image Push
          │
          ▼
      Docker Hub
          │
          ▼
       AWS EC2
          │
          ├── docker compose pull
          │
          └── docker compose up -d
                  │
                  ▼
             배포 완료
                  │
                  ▼
              기능 검증
                  │
          ┌───────┴───────┐
          │               │
        성공              실패
          │               │
          ▼               ▼
       운영 완료       이전 Image로
                       롤백
```

이를 통해 Jenkins를 중심으로 빌드 → 테스트 → Docker Image 생성 → Docker Hub Push → AWS EC2 배포 → 검증 → 장애 발생 시 롤백까지의 전체 배포 과정을 자동화합니다.
