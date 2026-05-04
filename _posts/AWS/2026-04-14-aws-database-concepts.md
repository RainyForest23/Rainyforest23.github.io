---
published: true
title: "AWS Database 핵심 개념 정리 (RDS, DynamoDB)"
last_modified_at: 2026-04-14
categories: AWS
tags:
    - RDS
    - DynamoDB
    - Database
    - EC2
toc: true
toc_sticky: true
toc_label: "목차"
---

AWS에서 제공하는 대표적인 데이터베이스 서비스 두 가지 - RDS와 DynamoDB를 정리해보자.

---

## I. RDS (Relational Database Service)

한마디로 **AWS가 관리해주는 관계형 데이터베이스 서비스**.

직접 EC2에 MySQL 설치해서 운영하는 것과 달리, OS 패치, 백업, 복구, 스케일링 등을 AWS가 대신 처리해줌.

### 지원 엔진

- MySQL
- PostgreSQL
- MariaDB
- Oracle
- Microsoft SQL Server
- **Amazon Aurora** (AWS 자체 개발, MySQL/PostgreSQL 호환)

### 핵심 기능

- **자동 백업** : 설정한 보존 기간(최대 35일) 동안 자동으로 백업 수행
- **Multi-AZ 배포** : 다른 가용 영역(AZ)에 Standby 인스턴스를 두어 장애 시 자동 전환 (Failover)
- **Read Replica** : 읽기 전용 복제본을 만들어 읽기 트래픽 분산
- **스토리지 자동 확장** : 용량 부족 시 자동으로 스토리지 증가

### 구조

```
애플리케이션
     │
     ▼
[RDS Endpoint]
     │
     ├── Primary DB (읽기/쓰기)
     │
     └── Standby DB (Multi-AZ, 장애 대비)
              └── Read Replica (읽기 전용 트래픽 처리)
```

### Aurora

- 일반 RDS 대비 MySQL은 5배, PostgreSQL은 3배 빠른 성능
- 스토리지가 3개 AZ에 6개 복사본으로 자동 분산 저장
- **Serverless 옵션** : 트래픽에 따라 자동으로 용량 조절

---

## II. DynamoDB

한마디로 **AWS의 완전관리형 NoSQL 데이터베이스**.

테이블 스키마를 미리 정의하지 않아도 되고, 어떤 규모에서도 한 자릿수 밀리초 응답 속도를 보장함.

### RDS와 비교

| 구분 | RDS | DynamoDB |
|---|---|---|
| 데이터 모델 | 관계형 (테이블, 스키마 고정) | NoSQL (Key-Value / Document) |
| 스케일링 | 수직 확장 (인스턴스 크기 변경) | 수평 확장 (자동) |
| 쿼리 | SQL | API (GetItem, Query, Scan 등) |
| 적합한 용도 | 복잡한 조인, 트랜잭션 | 대규모 단순 조회, 유연한 스키마 |
| 관리 부담 | 중간 | 거의 없음 (완전관리형) |

### 핵심 구성 요소

- **테이블** : 데이터를 저장하는 단위
- **아이템** : 테이블의 각 행 (RDB의 row)
- **속성** : 아이템의 각 필드 (RDB의 column), 아이템마다 속성이 달라도 됨
- **파티션 키 (Partition Key)** : 데이터를 분산 저장할 때 사용하는 기본 키, 필수
- **정렬 키 (Sort Key)** : 파티션 키와 함께 사용해 복합 기본 키 구성, 선택

### 주요 기능

- **TTL (Time to Live)** : 특정 시간이 지나면 아이템 자동 삭제
- **DynamoDB Streams** : 테이블 변경 사항을 실시간으로 캡처 (Lambda 연동 등)
- **글로벌 테이블** : 여러 리전에 자동으로 복제되는 다중 리전 테이블
- **DAX (DynamoDB Accelerator)** : DynamoDB 앞단의 인메모리 캐시, 마이크로초 응답

### 언제 DynamoDB를 선택하나?

- 스키마가 유연하게 변해야 할 때
- 초당 수만 건 이상의 대규모 트래픽
- 서버리스 아키텍처와 연동 (Lambda + DynamoDB 조합이 자주 쓰임)
- 세션 데이터, 사용자 프로필, 게임 리더보드 등

---

## III. 실습 후기

### EC2 접속 시 사용자 이름은 기본값으로

EC2 인스턴스에 SSH 접속할 때 사용자 이름을 임의로 설정했더니 계속 인증 오류가 발생했다.
원인은 간단했는데, EC2 AMI마다 기본 사용자 이름이 정해져 있다.

| AMI | 기본 사용자 이름 |
|---|---|
| Amazon Linux 2 | `ec2-user` |
| Ubuntu | `ubuntu` |
| CentOS | `centos` |
| RHEL | `ec2-user` 또는 `root` |

임의로 바꾸면 SSH 키 인증 자체가 실패함. **그냥 AMI에 맞는 기본값 쓰는 게 제일 안전하다.**

### 실습 끝나면 과금 리소스 반드시 확인

실습 끝나고 리소스를 제대로 정리 안 하면 조용히 과금이 쌓인다.
벌써 4천원째... 주요 과금 발생 리소스들 체크리스트:

- [ ] EC2 인스턴스 중지 또는 종료
- [ ] RDS 인스턴스 중지 또는 삭제
- [ ] NAT Gateway 삭제 (시간당 과금)
- [ ] Elastic IP 해제 (연결 안 된 EIP는 과금)
- [ ] Load Balancer 삭제
- [ ] S3 버킷 내 데이터 삭제

> 실습 직후 **Billing Dashboard**에서 예상 청구액 꼭 확인하자.
