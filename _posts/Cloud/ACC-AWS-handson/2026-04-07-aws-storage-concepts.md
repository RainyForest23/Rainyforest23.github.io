---
published: true
title: "AWS Storage 핵심 개념 정리 (S3, CloudFront)"
last_modified_at: 2026-04-07
categories: [Cloud, ACC-AWS-handson]
tags:
    - S3
    - CloudFront
    - EBS
    - EFS
    - Storage
toc: true
toc_sticky: true
toc_label: "목차"
---

클라우드 스토리지의 종류 - File Storage, Block Storage, Object Storage

---

## I. 스토리지 종류

### 1. File Storage

데이터를 파일 단위의 계층 구조로 저장하는 데이터 스토리지

- e.g. **Amazon EFS**

### 2. Block Storage

데이터를 고정된 크기의 블록 단위로 저장하는 데이터 스토리지

- e.g. **Amazon EBS**

### 3. Object Storage

데이터를 하나의 오브젝트 단위로 저장하는 데이터 스토리지
대용량 미디어 파일, 이미지, 백업 등의 비정형 데이터를 저장

- e.g. **Amazon S3**

---

## II. Amazon S3

파일을 하나의 객체로 저장하고, 키를 통해 관리함.

- **확장성**
- **데이터 보호**
- **비용 효율성**
- → 대량의 데이터를 안전하게 저장 가능

### 구성

```
버킷 (데이터를 담는 최상위 컨테이너, 폴더 같은 것)
└── 객체 (파일, 데이터)
        = 객체 데이터 + 메타데이터(객체의 속성)
```

객체는 전역이 아닌 **버킷 내에서 고유하게 식별**됨

객체 구성 요소 : `Key`, `Value`, `Version ID`, `Metadata`, `태그`, `액세스 제어 정보`

### 암호화

데이터를 암호화하는 방법에는 두 가지가 있음

- **서버 측 암호화** : AWS가 업로드된 데이터를 서버에서 암호화
- **클라이언트 측 암호화** : 파일 업로드 전 로컬에서 암호화하여 전송

### Access 제어

- **IAM** : 사용자/역할 단위 권한 관리
- **ACL (Access Control List)** : 버킷/객체 단위 접근 제어
- **버킷 정책** : JSON 기반 버킷 수준 정책
- **CORS** : 다른 도메인에서의 요청 허용 설정
- **Pre-signed URL** : 임시 접근 URL 발급

### 스토리지 관리

- **버전 관리** : 객체의 이전 버전 보존/복구 가능
- **객체 복제** : 다른 버킷으로 자동 복제
- **스토리지 클래스** (SAA 자주 출제)

| 클래스 | 특징 |
|---|---|
| Standard | 자주 접근하는 데이터 |
| Intelligent-Tiering | 접근 패턴에 따라 자동 계층 이동 |
| Standard-IA | 자주 접근하지 않는 데이터, 빠른 조회 필요 |
| One Zone-IA | 단일 AZ, 비용 저렴, 가용성 낮음 |
| Glacier | 장기 아카이브, 저비용 |

- **수명 주기** : 객체를 효율적으로 저장/관리하기 위해 일정 기간 후 자동으로 클래스 변경 또는 삭제

---

## III. CloudFront

**CDN 서비스** + 기본 보안을 통해 S3 보호

### CDN이란?

원본 서버 위치와 상관없이, 사용자에게 더 가까이 있는 서버에서 캐싱된 컨텐츠를 배포하여 속도가 빠름

- 캐싱해두는 곳 = **Edge Location**
- 사용자 요청을 가까운 Edge Location으로 라우팅

### 3계층 캐시 구조

```
Origin (원본 서버)
  │
  ▼
REC (Regional Edge Cache)  ← 중간 캐시 계층
  │
  ▼
Edge Location              ← 사용자와 가장 가까운 캐시
  │
  ▼
사용자
```

캐시 미스 시 Edge Location → REC → Origin 순으로 거슬러 올라가 데이터를 가져옴
