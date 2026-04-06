---
published: true
title: "AWS 네트워크 핵심 개념 정리 (Route 53, VPC, NAT Gateway, Bastion Host, NACL)"
last_modified_at: 2026-03-31
categories: AWS
tags:
    - Route53
    - VPC
    - NAT Gateway
    - Bastion Host
    - NACL
toc: true
toc_sticky: true
toc_label: "목차"
---

AWS 실습하다 보면 VPC 콘솔에서 낯선 단어들이 쏟아지는데, 한번 싹 정리하고 가보자.

---

## I. Route 53

한마디로 **AWS의 DNS 서비스**. DNS가 뭔지부터 짚고 가자.

- **DNS (Domain Name System)**

    - 우리가 브라우저에 `www.google.com` 치면, 컴퓨터는 사실 이 이름을 모름
    - 컴퓨터가 실제로 이해하는 건 IP 주소 (`142.250.196.68` 이런 것들)
    - DNS는 이 **도메인 이름 → IP 주소** 변환을 해주는 전화번호부 같은 것
- **Route 53이 하는 일**

    - 도메인 등록 : `myapp.com` 같은 도메인을 직접 구매/관리 가능
    - DNS 라우팅 : 사용자 요청을 EC2, ALB, S3 등으로 연결해줌
    - 헬스체크 : 서버 죽으면 자동으로 다른 서버로 트래픽 돌림
- **왜 이름이 Route 53이냐?**

    - DNS가 사용하는 포트 번호가 **53번**이라서. 심플.

---

## II. VPC (Virtual Private Cloud)

한마디로 **AWS 안에 나만의 가상 사설 네트워크를 만드는 것**.

- **왜 필요한가?**
    - AWS에 EC2 서버 만들면 기본적으로 인터넷에 다 노출됨
    - DB 서버 같은 건 외부에서 절대 직접 접근하면 안 됨
    - VPC로 **"이 리소스들은 내 네트워크 안에서만 통신해"** 라는 울타리를 치는 것

```
[인터넷] ──→ [VPC 경계]
                 ├── Public Subnet  (인터넷 접근 가능)
                 │       └── EC2 (웹 서버)
                 └── Private Subnet (인터넷 접근 불가)
                         └── RDS (DB 서버)
```

- **Subnet (서브넷)**

    - VPC 안을 더 잘게 쪼갠 것
    - **Public Subnet** : 인터넷과 직접 통신 가능한 구역
    - **Private Subnet** : 인터넷과 직접 통신 불가능한 구역. 외부에서 못 들어옴.
- **Internet Gateway (IGW)**

    - VPC와 인터넷을 연결해주는 문. 이게 없으면 VPC 안 리소스는 인터넷 자체가 안 됨.
    - Public Subnet은 라우팅 테이블에 IGW 경로가 등록되어 있음
- **CIDR 블록** (`10.0.0.0/16` 이런 것들)

    - VPC에 할당되는 IP 주소 범위
    - `/16`이면 65,536개, `/24`면 256개의 IP를 쓸 수 있다는 의미
    - 그냥 "이 VPC에서 사용할 수 있는 IP 주소 묶음"이라고 생각하자

---

## III. NAT Gateway

한마디로 **Private Subnet 안의 서버가 인터넷에 요청(아웃바운드)은 할 수 있게 해주는 중간 대리인**.

- **왜 필요한가?**
    - Private Subnet은 외부에서 접근이 안 되는 게 장점인데...
    - DB 서버도 패키지 업데이트, 외부 API 호출 같은 걸 해야 할 때가 있음
    - 이때 **"나가는 요청"은 허용하되, "들어오는 연결"은 차단**하고 싶은 것

```
[Private Subnet의 EC2]
        │
        ▼ (아웃바운드 요청)
[NAT Gateway] ─→ [Internet Gateway] ─→ [인터넷]

인터넷 ─→ NAT Gateway (여기서 차단, Private EC2 IP는 외부에 노출 안 됨)
```

- **핵심 포인트**
    - NAT Gateway는 반드시 **Public Subnet**에 위치해야 함 (자기가 인터넷 접근이 가능해야 하니까)
    - 방향이 중요 : **나가는 건 OK, 들어오는 건 NO**
    - 비용이 발생함 (시간 + 데이터 전송량 과금)

---

## IV. Bastion Host

한마디로 **Private Subnet 안의 서버에 SSH 접속하기 위한 중간 경유 서버**.

- **왜 필요한가?**
    - Private Subnet의 EC2는 외부에서 직접 SSH 접속이 불가능
    - 그렇다고 DB 서버를 Public으로 바꿀 수는 없음
    - 그래서 Public Subnet에 **"출입문 역할만 하는 작은 서버"** 를 하나 두는 것

```
[내 로컬 PC]
     │  SSH
     ▼
[Bastion Host] ← Public Subnet, 외부에서 접근 가능
     │  SSH
     ▼
[Private EC2 / RDS] ← Private Subnet, 외부 직접 접근 불가
```

- **특징**
    - Bastion Host 자체는 최소한의 역할만 함 (SSH 포트포워딩 정도)
    - 보안 그룹을 아주 엄격하게 설정해야 함 (내 IP만 22번 포트 허용 등)
    - "Jump Server", "Jump Host" 라고도 불림

---

## V. NACL (Network Access Control List)

한마디로 **Subnet 단위로 트래픽을 필터링하는 방화벽**.

- **보안 그룹 (Security Group) 과의 차이가 헷갈리는 경우가 많음**

|구분|NACL|보안 그룹 (Security Group)|
|---|---|---|
|적용 단위|Subnet|EC2 인스턴스|
|상태 추적|**Stateless** (인/아웃 각각 설정)|**Stateful** (인바운드 허용 시 아웃바운드 자동 허용)|
|규칙 적용|번호 순서대로 (낮을수록 우선)|모든 규칙 검사 후 허용/거부|
|기본 동작|명시 안 하면 거부 가능|명시 안 하면 거부|

- **Stateless가 핵심**

    - 보안 그룹은 인바운드 허용하면 응답(아웃바운드)이 자동으로 허용됨
    - NACL은 그런 거 없음. 인바운드 허용했으면 **아웃바운드도 명시적으로 허용**해줘야 함
    - 그래서 NACL 설정할 때 ephemeral port(임시 포트, 1024-65535)도 아웃바운드 허용해줘야 응답이 나감
- **규칙 번호 순서**

    - 규칙 100번이 `허용`, 규칙 200번이 `거부`면 → **100번이 먼저 적용되어 허용**됨
    - 그래서 더 구체적인 규칙을 낮은 번호에 놓아야 함

---

## 전체 구조 한눈에 보기

```
인터넷
  │
  ▼
[Route 53] → 도메인을 IP로 변환하여 라우팅
  │
  ▼
[Internet Gateway]
  │
  ▼
┌─────────────────────── VPC ────────────────────────┐
│                                                     │
│  [NACL] ← Subnet 경계에서 트래픽 필터링             │
│                                                     │
│  Public Subnet                                      │
│  ├── Bastion Host  (SSH 접속 경유용)                │
│  └── NAT Gateway   (Private → 인터넷 아웃바운드)    │
│                                                     │
│  Private Subnet                                     │
│  ├── EC2 (앱 서버)                                  │
│  └── RDS (DB 서버)                                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

각 개념이 독립적인 것 같아 보여도, 실제 VPC 구성할 때는 이 다섯 가지가 전부 맞물려서 동작하니까 흐름을 같이 이해해두면 훨씬 편함!
