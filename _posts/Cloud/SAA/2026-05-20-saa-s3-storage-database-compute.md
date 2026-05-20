---
published: true
title: "SAA 벼락치기 - S3 심화, 스토리지, DB, 컴퓨팅 서비스 (★★★)"
last_modified_at: 2026-05-20
categories: [Cloud, SAA]
tags:
    - S3
    - EBS
    - EFS
    - FSx
    - RDS
    - Aurora
    - DynamoDB
    - ElastiCache
    - EC2
    - ELB
    - CloudFront
    - SAA
toc: true
toc_sticky: true
toc_label: "목차"
---

SAA-C03 벼락치기 핵심 정리. S3 심화 기능부터 스토리지, DB, 컴퓨팅까지 한번에.

---

## I. S3 버전 관리, 객체 잠금, 수명 주기 정책 ★★★

### S3 버전 관리 (Versioning)

같은 이름의 파일을 업로드하면 기존 파일이 덮어써진다. **버전 관리**를 활성화하면 파일을 덮어써도 이전 버전을 보존한다.

### S3 객체 잠금 (Object Lock)

**일정 기간 동안 파일의 수정/삭제를 차단하는 기능.** 실수로 파일을 수정/삭제하는 걸 방지할 때 사용.

2가지 보존 모드:

1. **거버넌스 모드** : 특별 권한이 있는 관리자만 수정/삭제 가능
2. **규정 준수 모드** : 모든 사용자 수정/삭제 불가 (루트 사용자 포함)

그리고 S3 객체 잠금에는 **법적 보존(Legal Hold)** 기능도 있다. 기간에 상관없이 수동으로 해제하기 전까지 객체의 수정/삭제를 금지하는 기능으로, **언제 끝날지 모르는 소송 같은 이슈가 발생했을 때** 사용한다.

### S3 수명 주기 정책 (Lifecycle Policy)

**파일을 N일 후에 자동으로 이동/삭제시키는 기능.**

```
S3 Standard  →(30일 후)→  S3 Standard-IA  →(1년 후)→  S3 Glacier
```

- 데이터의 접근 빈도를 예측할 수 있어서 다른 유형의 S3로 이동하고 싶을 때
- 보관 기간이 정해져 있어서 기간이 지나면 자동 삭제할 때

---

## II. S3 암호화-SSE, S3 Batch Operations ★★☆

### SSE (Server Side Encryption)

**서버가 자체적으로 암호화하는 방식.** 암호화 설정을 해두지 않으면 해킹당했을 때 해커가 파일을 바로 열어볼 수 있다.

SSE 종류:

1. **SSE-S3** : S3가 키를 알아서 생성 및 관리. 가장 기본적이고 관리 쉬운 방식
2. **SSE-KMS** : AWS KMS(Key Management Service)가 키를 생성 및 관리
   - 철저한 보안 요구 시 사용
   - KMS는 암호화 키를 주기적으로 **자동 교체**해주는 기능 보유
   - 파일이 많을 경우 **S3 Bucket Key**를 활용하면 파일마다 키를 별도로 생성하지 않고 Bucket Key를 재사용해 암호화 비용을 절감
3. **SSE-C** : 고객이 키를 직접 생성 및 관리. 권장되지 않는 방식

### S3 Batch Operations

**S3에 저장된 수백만~수십억 개의 파일에 대해 동일한 작업(복사, 삭제, 설정 적용 등)을 한 번에 실행**할 수 있게 해주는 서비스.

---

## III. EBS, EFS, FSx, 인스턴스 스토어 ★★★

### EBS (Elastic Block Store)

**외장 SSD/HDD처럼 EC2(또는 RDS)에 연결해서 사용하는 블록 스토리지 서비스.**

```
EC2 instance ←─── EBS (6GB, root volume /dev/sda1)
```

특징:
- EC2, RDS를 제외한 다른 AWS 서비스에는 연결 불가
- **단일 AZ**에서만 작동
- 고성능 스토리지로 사용할 수 있는 유형 존재

### EFS (Elastic File System)

**여러 대의 컴퓨터가 동시에 같은 파일 시스템을 공유해서 쓸 수 있는 NFS(Network File System) 스토리지.** 구글 클라우드나 iCloud처럼 여러 사용자가 동시에 같은 파일 시스템을 공유하는 방식.

특징:
- **다중 AZ**에서 작동 가능
- 고성능 스토리지는 아님
- **NFS 프로토콜**만 지원
- S3의 Intelligent-Tiering처럼 자주 접근하지 않는 파일을 **EFS IA(Infrequent Access)** 클래스로 이동시켜주는 **EFS Intelligent-Tiering** 기능도 존재

### 인스턴스 스토어 (Instance Store)

**EC2 컴퓨터에 내장되어 있는 임시용 디스크.** EC2 인스턴스를 중지하거나 종료하면 데이터가 전부 사라진다.

### FSx (File System for Extended use)

**고성능(높은 처리량, 낮은 지연 시간) 파일 스토리지 서비스.**

| 유형 | 특징 |
|---|---|
| FSx for Lustre | 머신러닝, 빅데이터 분석에 활용. S3 연동 가능. Lustre 프로토콜만 지원 |
| FSx for NetApp ONTAP | 다양한 OS(윈도우, 맥, 리눅스) 호환. NFS + SMB 둘 다 지원 |
| FSx for Windows File Server | SMB 프로토콜만 지원 |
| FSx for OpenZFS | NFS 프로토콜만 지원 |

---

## IV. DataSync, Snowball Edge, Transfer Family, Storage Gateway ★★★

### DataSync

**대용량 데이터를 전송할 때 사용하는 서비스.**

- 온프레미스 스토리지 ↔ AWS 시스템 간 데이터 전송
- AWS 스토리지 서비스들 간 데이터 전송
- 다른 클라우드 스토리지 ↔ AWS 스토리지 간 데이터 전송

### Snowball Edge

**물리적 장비 배송으로 대용량 데이터를 전송할 때 사용하는 서비스.**

1. AWS 사이트에서 Snowball Edge를 신청
2. AWS에서 Snowball Edge 장비를 택배로 보내줌
3. 받은 장비를 데이터를 옮기고자 하는 컴퓨터에 직접 연결해 데이터를 옮겨 담음 (물리적으로 직접 연결하기 때문에 속도가 매우 빠름)
4. 데이터를 다 옮겨담았다면 장비를 다시 AWS로 보냄

### Transfer Family

**파일을 송수신할 때 필요한 FTP(SFTP, FTPS) 서버 역할을 하는 서비스.**

- Transfer Family를 활용하면 S3와 연동해서 파일을 송수신할 수 있다.

### Storage Gateway

**온프레미스 환경에서 S3 스토리지를 로컬 스토리지처럼 쓸 수 있게 연결해주는 서비스.**

- 데이터를 옮기는 것이 목적이 아닌, **연결해서 쓰는 것**이 목적
- 통로(Gateway) 역할만 할 뿐, 자체적인 스토리지 기능은 없음

| 유형 | 설명 |
|---|---|
| 파일 게이트웨이 | 온프레미스 데이터를 '공유 폴더' 형태로 연동해 S3에 저장. NFS/SMB 사용. 실시간 사용 가능 |
| 볼륨 게이트웨이 | 온프레미스 데이터를 '블록 스토리지(디스크)' 형태로 연동해 S3에 저장. EBS 스냅샷 업로드. 실시간 사용 불가 |
| 테이프 게이트웨이 | 온프레미스의 '테이프(tape)'에 저장된 데이터를 S3에 저장. 백업 및 아카이빙 용도 |

---

## V. RDS, 스토리지 유형, 기능 ★★★

### RDS 스토리지 유형

1. **범용 SSD 스토리지 (gp3, gp2)** : 범용적으로 사용하는 스토리지. 일반적으로 이 스토리지 사용
2. **프로비저닝된 IOPS SSD** : 높은 처리량과 짧은 지연 시간의 스토리지. IOPS를 직접 설정 가능 (IOPS : 초당 처리할 수 있는 읽기/쓰기 작업량)

### RDS 주요 기능

1. **Multi AZ 배포 (다중 AZ 배포)**
   - 기존 DB를 복제해서 대기 DB(Standby DB) 생성, 장애 시 자동으로 Standby DB를 Primary DB로 전환
   - **고가용성** 확보를 위해 사용
   - 동시에 같이 작동시키는 DB가 아닌, 대기 DB를 추가하는 것이기 때문에 **성능 향상이 일어나지 않는다**

2. **Multi AZ DB Cluster (다중 AZ 클러스터 배포)**
   - 기존 DB를 복제해서 읽기 전용 DB를 2대 이상 더 추가해서 구성
   - 고가용성 + **읽기 성능 개선**

3. **Read Replica (읽기 전용 복제본)**
   - 읽기 전용 DB를 추가함으로써 **읽기 성능을 향상시키고 기존 DB의 부하를 줄이기** 위한 기능

4. **Cross Region Read Replica (리전 간 읽기 전용 복제본)**
   - **재해 복구(DR, Disaster Recovery)** 를 위한 기능

5. **RDS Proxy (프록시)**
   - 애플리케이션과 DB 사이에서 **DB 연결(Connection)이 과도하게 많이 일어나서** 발생하는 문제를 막아주는 기능
   - DB 연결(Connection)을 재사용하고 관리함으로써 **DB 연결 관련 문제를 예방**

6. **RDS Blue/Green Deployment (블루/그린 배포)**
   - 프로덕션 환경(블루)과 동기화된 별도의 스테이징 환경(그린)을 생성
   - 스테이징 환경에서 DB를 변경하고 테스트한 뒤, 다운타임을 1분 미만으로 최소화하면서 트래픽을 그린으로 전환

---

## VI. Aurora, DynamoDB, ElastiCache ★★★

### Aurora

**'고성능' 관계형 데이터베이스(RDBMS) 서비스.** Aurora MySQL, Aurora PostgreSQL이 대표적.

| 기능 | 설명 |
|---|---|
| Global Database (글로벌 데이터베이스) | 여러 리전에 걸쳐 여러 개의 DB를 구축. 재해 복구(DR) + 전 세계 읽기 성능 최적화 |
| Auto Scaling (오토스케일링) | CPU 사용량 같은 지표를 모니터링하다가, 임계치를 넘어가면 읽기 전용 복제본을 자동으로 생성 |

### DynamoDB

**NoSQL 데이터베이스 서비스.** MySQL, PostgreSQL 같은 RDBMS는 지원하지 않는다.

| 기능 | 설명 |
|---|---|
| DynamoDB Streams | 데이터 변경 사항을 실시간으로 추적 및 전달하는 기능 |
| DynamoDB Accelerator (DAX) | 기존 DynamoDB보다 성능을 끌어올리고 싶을 때 사용. DynamoDB에 호환되면서 바로 장착할 수 있는 캐시 기능 → 읽기 성능 향상 |

### ElastiCache

**자주 조회되는 데이터를 빠리 조회해주는 캐시 서비스.**

- **읽기 성능을 향상**시킬 수 있다
- ElastiCache가 일부 트래픽에 대응해줌으로써 DB 부하를 줄여줄 수 있다
- **동일한 데이터를 반복적으로 조회할 때 사용**. 조회 결과가 자주 변경되는 데이터일 경우에는 적합하지 않다

---

## VII. EC2, ELB, Global Accelerator, CloudFront ★★★

### EC2

**컴퓨터를 빌려서 원격으로 접속해 사용하는 서비스.** 하나의 컴퓨터(서버)라고 생각하면 된다.

- **오토 스케일링 (Auto Scaling)** : 대상 추적 정책에 따라 자동으로 서버를 확장 또는 축소
  - 대상 추적 정책 : 원하는 목표값(Target)을 벗어나지 않도록 자동으로 서버를 확장/축소

| 요금 유형 | 설명 |
|---|---|
| 온디맨드 인스턴스 | 사용한 시간만큼 결제하는 후불 결제. 항상 실행됨. **범용적으로 많이 사용** |
| 스팟 인스턴스 | AWS의 남는 EC2를 매우 싸게 빌려 씀. AWS가 언제든 회수할 수 있어서 **서비스가 중단될 수도 있다**. 중단되도 괜찮은 작업이면서 비용을 최소화해야 되는 경우에 사용 |
| 예약 인스턴스 | 1년 또는 3년으로 장기 약정으로 예약해서 비용을 할인받는 방식. **장기 사용이 확정됐을 때** 사용 |

### ELB (Elastic Load Balancing)

**트래픽을 여러 서버로 적절하게 분배해주는 자동 분산 서비스.**

| 유형 | 설명 |
|---|---|
| ALB (Application Load Balancer) | 웹 애플리케이션(HTTP, HTTPS)에서 주로 사용. Sticky Session, WAF 연결 가능. HTTP/HTTPS 프로토콜만 지원 |
| NLB (Network Load Balancer) | TCP와 UDP 트래픽을 모두 처리. 아주 빠른 응답 속도를 보장 |
| GLB (Gateway Load Balancer) | 보안 전용(ex. 네트워크 방화벽) 로드 밸런서. 네트워크 보안 장비를 부하분산하기 위한 서비스 |

### CloudFront

**컨텐츠(이미지, 파일 등)를 캐싱해서 전세계 사용자에게 빠른 속도로 컨텐츠를 제공하는 서비스.**

- 정적 컨텐츠(이미지, jpg 등) → S3에서 캐싱
- 동적 컨텐츠(php 등) → EC2/ALB에서 처리

### Global Accelerator

**전세계 사용자에게 빠른 네트워크 통신을 제공하는 서비스.**

- AWS의 글로벌 프라이빗 네트워크를 활용해 지연 시간을 줄임
- 고정 Anycast IP 2개를 제공해 DNS 변경 없이 여러 리전의 EC2, ALB, NLB, Elastic IP로 라우팅 가능

> **CloudFront vs Global Accelerator**: CloudFront는 콘텐츠 캐싱을 통한 성능 향상, Global Accelerator는 네트워크 경로 최적화를 통한 성능 향상. CloudFront는 정적/동적 콘텐츠에, Global Accelerator는 TCP/UDP 기반 애플리케이션에 적합하다.
