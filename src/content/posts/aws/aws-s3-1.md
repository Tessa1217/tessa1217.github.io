---
title: 🪣 Amazon S3 (Simple Storage Service)
published: 2026-03-19
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🪣 Amazon S3 (Simple Storage Service)

> AWS의 핵심 빌딩 블록 — **"무한히 확장 가능한" 객체 스토리지**
> 
> 
> 내구성 99.999999999%(11 9's), 시험 전 영역에서 고빈도 출제
> 
---
## 목차

1. [S3 기본 개념](#1-s3-기본-개념)
2. [S3 보안 (Security) 개요](#2-s3-보안-security-개요)
3. [정적 웹사이트 호스팅 (Static Website Hosting)](#3-정적-웹사이트-호스팅-static-website-hosting)
4. [버저닝 (Versioning)](#4-버저닝-versioning)
5. [복제 (Replication)](#5-복제-replication)
6. [스토리지 클래스 (Storage Classes)](#6-스토리지-클래스-storage-classes)
7. [S3 Lifecycle](#7-s3-lifecycle)
8. [성능 (Performance)](#8-성능-performance)
9. [S3 Batch Operations](#9-s3-batch-operations)
10. [S3 Event Notifications](#10-s3-event-notifications)
11. [S3 Analytics - Storage Class Analysis](#11-s3-analytics-storage-class-analysis)
12. [S3 Storage Lens](#12-s3-storage-lens)
13. [Requester Pays](#13-requester-pays)
14. [핵심 요약 & 시험 포인트](#14-핵심-요약-시험-포인트)
15. [참고 자료](#-참고-자료)
---

## 1. S3 기본 개념

### 🪣 버킷 (Buckets)

- 객체(파일)를 저장하는 컨테이너 (= 디렉터리)
- **버킷 이름은 전 세계 모든 계정에서 고유** (Globally Unique Name)
- 버킷은 **리전(Region) 레벨**에서 생성됨 — 글로벌 서비스처럼 보이지만 실제는 리전별

**버킷 이름 규칙:**

- 소문자, 숫자, 하이픈만 사용
- 3~63자
- IP 주소 형태 불가
- 소문자 또는 숫자로 시작
- `xn--` 접두사 불가, `s3alias` 접미사 불가

---

### 📄 객체 (Objects)

| 항목 | 내용 |
| --- | --- |
| **키 (Key)** | 객체의 전체 경로 (prefix + 객체명) |
| **최대 객체 크기** | **5TB** |
| **5GB 초과 업로드** | **멀티파트 업로드 (Multi-part Upload)** 필수 |
| **메타데이터** | 텍스트 키/값 쌍 (시스템 또는 사용자 정의) |
| **태그 (Tags)** | 유니코드 키/값 쌍, **최대 10개** (보안/라이프사이클 활용) |
| **Version ID** | 버저닝 활성화 시 부여 |

```
s3://my-bucket/images/2025/photo.jpg
              └─────prefix──────┘└─name─┘
```

:::note
⚠️ S3에는 실제 **"디렉터리" 개념이 없음** — 키의 슬래시(/)가 구조처럼 보이게 하는 것뿐.
:::
---

### 🎯 주요 Use Cases

```
백업/스토리지, 재해 복구(DR), 아카이브, 하이브리드 클라우드 스토리지,
애플리케이션 호스팅, 미디어 호스팅, 데이터 레이크/빅데이터 분석,
소프트웨어 배포, 정적 웹사이트
```

---

## 2. S3 보안 (Security) 개요

> 🔐 **상세 내용은 별도 파일 참고**: `AWS_S3_Security_Notes.md`
> 
> 
> (Encryption, Access Points, Object Lambda, MFA Delete, Object Lock, CORS, Pre-signed URL, Access Logs 포함)
> 

### 보안 정책 계층

| 레이어 | 종류 | 설명 |
| --- | --- | --- |
| **User-based** | IAM Policies | 특정 IAM User/Role에 API 허용 정의 |
| **Resource-based** | Bucket Policy | Bucket-wide JSON 정책, Cross-Account 가능 |
| **Resource-based** | Object/Bucket ACL | 세밀한 객체 단위 제어 (비활성화 가능) |
| **계정 레벨** | Block Public Access | 전체 계정 또는 버킷 단위 Public 차단 |

**접근 허용 조건 (IAM Principal이 S3 객체에 접근하려면):**

```
(IAM 권한 ALLOW  OR  Resource Policy ALLOW)
            AND
       명시적 DENY 없음
```

### Bucket Policy 핵심 활용

```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "PublicRead",
        "Effect": "Allow",
        "Principal": "*",
        "Action": ["s3:GetObject"],
        "Resource": ["arn:aws:s3:::examplebucket/*"]
    }]
}
```

- Public read 허용 (정적 웹사이트)
- Upload 시 Encryption 강제
- Cross-Account 접근 허용
- `aws:PrincipalOrgID` 조건으로 AWS Organizations 단위 접근 제어

### Block Public Access

- 기본값: **모든 Public 접근 차단** (데이터 유출 방지)
- **계정 레벨** 설정 가능 → 전체 계정 Bucket 일괄 차단
- 정적 웹사이트 공개 시 반드시 비활성화 + Bucket Policy 허용 둘 다 필요

---

## 3. 정적 웹사이트 호스팅 (Static Website Hosting)

- S3로 정적 HTML/CSS/JS 웹사이트 호스팅 가능
- URL 형식:
    - `http://bucket-name.s3-website-{region}.amazonaws.com`
    - `http://bucket-name.s3-website.{region}.amazonaws.com`

**설정 체크리스트:**

```
✅ Static Website Hosting 활성화
✅ Block Public Access 비활성화
✅ Bucket Policy에서 s3:GetObject 허용 (Principal: *)
✅ index.html 지정
```

> 💡 **403 Forbidden 오류**: Bucket Policy가 Public read를 허용하지 않는 것. CloudFront + OAC(Origin Access Control)를 사용하면 S3를 Public으로 열지 않고도 웹사이트 서비스 가능.
> 

---

## 4. 버저닝 (Versioning)

- **버킷 레벨에서 활성화** — 객체 단위 활성화 불가
- 같은 키로 업로드 시 버전 번호가 증가 (1, 2, 3…)
- **실수 삭제 방지** + 이전 버전 롤백 가능

| 상황 | 동작 |
| --- | --- |
| 버저닝 활성화 전 파일 | 버전 ID = `null` |
| 버저닝 비활성화(Suspend) | 기존 버전 **삭제되지 않음** — 신규 업로드만 버전 없음 |
| 파일 삭제 시 | 실제 삭제 아닌 **Delete Marker(삭제 마커)** 추가 |
| Delete Marker 삭제 시 | 파일 복원됨 |

```
버전 1 → 버전 2 → 버전 3 (현재)
                    │ 삭제
                    ▼
              [Delete Marker] → 파일이 없는 것처럼 보임
              Delete Marker 삭제 → 버전 3 복원
```

:::tip
📌 **복제(Replication) 사용 시**: 버저닝이 소스/대상 버킷 모두에서 **필수**
:::
---

## 5. 복제 (Replication)

### 종류

| 유형 | 설명 | 주요 Use Case |
| --- | --- | --- |
| **CRR** (Cross-Region Replication) | 다른 리전으로 복제 | 규정 준수, 지연 시간 감소, 계정 간 복제 |
| **SRR** (Same-Region Replication) | 같은 리전 내 복제 | 로그 집계, 운영/테스트 계정 간 실시간 동기화 |

### 주요 특성

| 항목 | 내용 |
| --- | --- |
| **사전 요건** | 소스 + 대상 버킷 모두 **버저닝 활성화 필수** |
| **복제 방식** | **비동기 (Asynchronous)** |
| **계정** | 서로 다른 AWS 계정 간 복제 가능 |
| **IAM** | S3에 적절한 IAM 권한 부여 필요 |
| **기존 객체** | 복제 활성화 후 **새 객체만** 자동 복제 |
| **기존 객체 복제** | **S3 Batch Replication** 사용 (실패 객체 재시도도 가능) |

### 삭제 동작

```
Delete Marker 복제:   선택적 설정 (기본값: 복제 안 함)
Version ID 포함 삭제: 복제 안 함 (악의적 삭제 방지)
```

### ⚠️ 체이닝 없음

```
버킷 1 → (복제) → 버킷 2 → (복제) → 버킷 3

버킷 1의 객체는 버킷 2에만 복제됨
버킷 3에는 자동 복제 안 됨 (No "chaining" of replication)
```

---

## 6. 스토리지 클래스 (Storage Classes)

### 📊 전체 비교표

| 스토리지 클래스 | 가용성 | 내구성 | AZ 수 | 검색 시간 | 최소 보존 기간 | 주요 Use Case |
| --- | --- | --- | --- | --- | --- | --- |
| **S3 Standard** | 99.99% | 11 9's | ≥3 | 즉시 | 없음 | 자주 접근하는 데이터 |
| **S3 Standard-IA** | 99.9% | 11 9's | ≥3 | 즉시 | 30일 | 비정기 접근, 빠른 조회 필요 |
| **S3 One Zone-IA** | 99.5% | 11 9's | **1** | 즉시 | 30일 | 재생성 가능한 보조 백업 |
| **S3 Glacier Instant Retrieval** | 99.9% | 11 9's | ≥3 | **밀리초** | **90일** | 분기별 접근, 즉각 복원 |
| **S3 Glacier Flexible Retrieval** | 99.99% | 11 9's | ≥3 | 분~시간 | **90일** | 백업, 아카이브 |
| **S3 Glacier Deep Archive** | 99.99% | 11 9's | ≥3 | **12~48시간** | **180일** | 장기 보관, 최저 비용 |
| **S3 Intelligent-Tiering** | 99.9% | 11 9's | ≥3 | 자동 | 없음 | 접근 패턴 예측 불가 |
| **S3 Express One Zone** | 99.95% | 높음 | **1** | **단일 자릿수 ms** | 없음 | 초저지연, AI/ML |

---

### 클래스별 상세

### S3 Standard (범용)

- 자주 접근하는 데이터, 낮은 지연 + 높은 처리량
- 동시 2개 시설 장애 허용 (내결함성)
- Use Case: 빅데이터, 모바일/게임, 콘텐츠 배포

### S3 Standard-IA / S3 One Zone-IA (비정기 접근)

- Standard보다 저렴하나 **검색 시 요금 발생**
- **One Zone-IA**: 단일 AZ → AZ 파괴 시 데이터 손실, 재생성 가능 데이터에만
- Use Case: Standard-IA → DR/백업, One Zone-IA → 온프레미스 보조 백업, 재생성 가능 데이터

### S3 Glacier (아카이브 3종)

| Glacier 유형 | 검색 옵션 | 검색 시간 | 최소 보존 |
| --- | --- | --- | --- |
| **Glacier Instant** | - | **밀리초** | 90일 |
| **Glacier Flexible** | Expedited (긴급) | 1~5분 | 90일 |
|  | Standard (표준) | 3~5시간 |  |
|  | Bulk (대량) | 5~12시간 (무료) |  |
| **Glacier Deep Archive** | Standard | **12시간** | **180일** |
|  | Bulk | **48시간** |  |

### S3 Intelligent-Tiering (자동 계층 이동)

- 월별 소액 모니터링 + 자동 계층화 수수료 발생
- **검색 요금 없음** (Intelligent-Tiering의 핵심 장점)
- 접근 패턴에 따라 자동 이동:

```
Frequent Access  (기본)         ← 자동
Infrequent Access               ← 30일 미접근 시 자동 이동
Archive Instant Access          ← 90일 미접근 시 자동 이동
Archive Access (선택, 설정 필요) ← 90~700일
Deep Archive Access (선택)      ← 180~700일
```

### S3 Express One Zone (신규, 2023)

- **단일 AZ** 내 Directory Bucket에 저장
- **단일 자릿수 밀리초 (single-digit millisecond)** 지연 시간
- S3 Standard 대비 최대 **10배 성능**, 비용 50% 절감
- SageMaker, Athena, EMR, Glue와 최적 통합
- Use Case: AI/ML 훈련, 금융 모델링, HPC, 미디어 처리

---

## 7. S3 Lifecycle

### 개요

- Lifecycle Rules를 사용해 객체를 **자동으로 다른 Storage Class로 전환**하거나 **삭제**
- 비용 최적화의 핵심 기능 — 시험에서 "가장 비용 효율적인 스토리지 전략" 문제의 정답

```
[Standard] →(30일)→ [Standard-IA] →(60일)→ [Glacier Flexible] →(365일)→ 삭제
```

---

### Lifecycle Rule 종류

| 규칙 유형 | 설명 | 예시 |
| --- | --- | --- |
| **Transition Actions** | 특정 시간 후 다른 Storage Class로 이동 | 생성 60일 후 Standard-IA로 이동 |
| **Expiration Actions** | 특정 시간 후 객체 삭제(만료) | 365일 후 로그 파일 삭제 |

---

### Transition Actions (전환 규칙)

```
Standard  →  Standard-IA  →  Glacier Instant  →  Glacier Flexible  →  Deep Archive
         ↑                ↑
     최소 30일          최소 90일
     Standard 이후       IA 이후
```

- 전환 방향: **항상 아래 등급으로만 가능** (상위 클래스로 되돌아가지 않음)
- 각 Storage Class로 전환되기까지 **최소 보존 기간 충족** 필요

---

### Expiration Actions (만료 규칙)

| 활용 | 설명 |
| --- | --- |
| **오래된 객체 삭제** | Access log 파일 365일 후 자동 삭제 |
| **이전 버전 삭제** | Versioning 활성화 시 non-current version 삭제 |
| **불완전한 Multipart Upload 삭제** | 일정 기간 완료되지 않은 Part 정리 |
| **Expired Delete Marker 삭제** | 불필요한 Delete Marker 자동 정리 |

---

### Rule 적용 범위

- **Prefix 기반**: 특정 경로에만 적용 (예: `s3://mybucket/logs/*`)
- **Tag 기반**: 특정 Tag가 있는 객체에만 적용 (예: `Department:Finance`)

---

### 시험 시나리오 예시

> **Q: 6개월간 자주 접근, 이후 1년간 가끔 접근, 그 이후 보관이 필요한 데이터의 비용 최적화 방법은?**
> 

```
생성 후 0~6개월:  S3 Standard
        ↓ Lifecycle Rule: 180일 후 전환
6개월~1.5년:     S3 Standard-IA
        ↓ Lifecycle Rule: 365일 후 전환
1.5년 이후:      S3 Glacier Flexible Retrieval
```

---

## 8. 성능 (Performance)

### 🔢 S3 Baseline Performance (기준 성능 수치)

> 📌 **시험 핵심 수치** — 자주 출제됨
> 
- S3는 높은 요청 속도에 자동으로 스케일링 (지연 시간 100~200ms)
- **Prefix당** 초당 요청 처리량:
    - **PUT/COPY/POST/DELETE: 3,500 req/s**
    - **GET/HEAD: 5,500 req/s**
- 버킷 내 Prefix 수 제한 없음
- Prefix를 4개로 분산 시: GET/HEAD **22,000 req/s** 달성 가능

```
s3://bucket/folder1/sub1/file   ← prefix: /folder1/sub1/
s3://bucket/folder1/sub2/file   ← prefix: /folder1/sub2/
s3://bucket/folder2/sub1/file   ← prefix: /folder2/sub1/
s3://bucket/folder2/sub2/file   ← prefix: /folder2/sub2/
→ 각 prefix에서 5,500 GET/s × 4 = 22,000 GET/s
```

---

### 🚀 S3 Transfer Acceleration

- S3 버킷으로의 **장거리 파일 전송 속도 향상**
- AWS CloudFront의 **엣지 로케이션(Edge Location)**을 경유:

```
[사용자 (일본)]  →  [CloudFront Edge (일본)]  →  [AWS 전용 백본망]  →  [S3 버킷 (us-east-1)]
                    빠른 업로드                       고속 전용선
```

- **Use Case**: 전 세계에서 하나의 S3 버킷으로 데이터를 빠르게 집계할 때
- **버킷 레벨**에서 활성화, 별도 엔드포인트 사용

---

### 📦 Multipart Upload (멀티파트 업로드)

- **5GB 초과 시 필수**, 100MB 초과부터 권장
- 파일을 여러 파트로 분할하여 **병렬 업로드** → 속도 향상
- 파트 업로드 실패 시 해당 파트만 재시도

```
[100GB 파일]
    │ 분할
    ├── Part 1 (10GB) ─→ S3
    ├── Part 2 (10GB) ─→ S3  (병렬)
    └── Part N (10GB) ─→ S3
                          │
                    완료 시 S3가 합성
```

---

### 🔍 S3 Byte-Range Fetches (바이트 범위 가져오기)

- 파일의 **특정 바이트 범위만** 병렬로 GET 요청
- **다운로드 속도 향상** (병렬 처리)
- 파일의 **일부만 조회** 가능 (헤더 데이터 파싱 등)

---

### 📊 S3 Select & Glacier Select

- **S3 Select**: SQL 쿼리로 S3에서 **필요한 데이터만 서버 사이드 필터링**
- 전체 파일 다운로드 없이 필요한 데이터만 추출 → **비용, 트래픽 절감**
- CSV, JSON, Parquet 형식 지원

```
전체 CSV 다운로드 후 필터링 → [큰 파일 전체 전송]  ← 비효율
S3 Select 사용             → [필요한 행만 반환]    ← 효율적
```

---

## 9. S3 Batch Operations

- 기존 S3 객체에 대해 **단일 요청으로 대규모 작업** 수행
- Job = 객체 목록 + 수행할 Action + 선택적 파라미터

### 지원하는 작업 (Actions)

| Action | 설명 |
| --- | --- |
| **Copy objects** | 버킷 간 객체 복사 |
| **Modify metadata** | 객체 메타데이터/속성 일괄 변경 |
| **Encrypt** | 미암호화 객체 일괄 암호화 (SSE-KMS 등으로 전환) |
| **Modify ACLs/Tags** | ACL 또는 Tag 일괄 수정 |
| **Restore from Glacier** | Glacier 객체 일괄 복원 |
| **Invoke Lambda** | 각 객체에 대해 Lambda Function 실행 |

### 작동 흐름

```
S3 Inventory로 객체 목록 생성
        │
        ▼
Athena로 필터링 (특정 조건의 객체만 선택)
        │
        ▼
S3 Batch Operations Job 생성
        │
        ▼
진행률 추적 + 재시도 자동 관리 + 완료 알림 + 리포트 생성
```

> 💡 **실무 활용**: 기존 미암호화 객체를 SSE-KMS로 일괄 전환할 때, S3 Replication 활성화 이전의 기존 객체를 복제할 때 (S3 Batch Replication)
> 

---

## 10. S3 Event Notifications

### 지원 이벤트 유형

- `S3:ObjectCreated`, `S3:ObjectRemoved`, `S3:ObjectRestore`, `S3:Replication` 등
- Object 이름으로 필터링 가능 (예: `.jpg`)

### 이벤트 대상 (Destinations)

```
S3 이벤트 발생
    │
    ├──→ SNS Topic  ← SNS Resource Policy 필요
    ├──→ SQS Queue  ← SQS Resource Policy 필요
    └──→ Lambda Function  ← Lambda Resource Policy 필요
```

:::important
📌 **중요**: S3 → SNS/SQS/Lambda 연결 시 IAM Role이 아닌 **Resource-based Policy(리소스 정책)** 설정 필요
:::

### S3 Event Notifications with Amazon EventBridge

```
S3 Bucket  →  Amazon EventBridge  →(Rules)→  18개 이상 AWS 서비스
```

**EventBridge 사용 시 장점:**

- **고급 필터링**: JSON Rules로 Metadata, Object size, 이름 기반 필터링
- **Multiple Destinations**: Step Functions, Kinesis Streams/Firehose 등
- **Archive / Replay Events**: 이벤트 재처리 가능
- **Reliable delivery**: 신뢰성 높은 전달 보장

> 💡 **선택 기준**: 단순한 트리거는 SNS/SQS/Lambda 직접 연결. 복잡한 라우팅이나 다양한 Destination이 필요하면 EventBridge 경유.
> 

---

## 11. S3 Analytics - Storage Class Analysis

- **언제** 객체를 다른 Storage Class로 전환할지 **분석 및 권장**
- 지원 범위: **Standard → Standard-IA 전환** 분석만 가능
    - ❌ One Zone-IA, Glacier는 직접 분석 불가
- Report는 S3 버킷에 **CSV 형식**으로 출력
- 분석 초기화 소요 시간: **24~48시간**
- Daily 업데이트

```
S3 Analytics 분석 결과
        │
        ▼
Lifecycle Rules 설계의 출발점으로 활용
```

> 💡 Lifecycle Rules를 처음 설계하거나 개선할 때 S3 Analytics를 먼저 실행해서 데이터 기반으로 전환 시점 결정 권장

---

## 12. S3 Storage Lens

- **전체 AWS Organization 수준**에서 S3 스토리지 사용량과 활동을 분석·최적화하는 도구
- 30일간 사용량 및 활동 지표 집계
- 이상 탐지(Anomaly Detection), 비용 효율화, 데이터 보호 Best Practice 적용
- 집계 단위: Organization, 특정 Account, Region, Bucket, Prefix
- 기본 대시보드(Default Dashboard) 제공 또는 커스텀 대시보드 생성
- 일별 S3 버킷으로 메트릭 내보내기 가능 (CSV, Parquet)

---

### Default Dashboard

- Multi-Region, Multi-Account 데이터 시각화
- Amazon S3가 사전 구성 (Preconfigured)
- **삭제 불가** (단, 비활성화는 가능)

---

### 주요 Metrics 카테고리

| 카테고리 | 주요 지표 | 활용 목적 |
| --- | --- | --- |
| **Summary Metrics** | StorageBytes, ObjectCount | 가장 빠르게 증가하는 Bucket/Prefix 파악 |
| **Cost-Optimization Metrics** | NonCurrentVersionStorageBytes, IncompleteMultipartUploadStorageBytes | 7일 이상된 불완전 Multipart Upload 탐지, 저비용 클래스 전환 후보 파악 |
| **Data-Protection Metrics** | VersioningEnabledBucketCount, MFADeleteEnabledBucketCount, SSEKMSEnabledBucketCount | 데이터 보호 Best Practice 미준수 Bucket 탐지 |
| **Access-Management Metrics** | ObjectOwnershipBucketOwnerEnforcedBucketCount | Object Ownership 설정 현황 파악 |
| **Event Metrics** | EventNotificationEnabledBucketCount | Event Notification 설정 현황 |
| **Performance Metrics** | TransferAccelerationEnabledBucketCount | Transfer Acceleration 활성화 현황 |
| **Activity Metrics** | AllRequests, GetRequests, PutRequests, BytesDownloaded | 스토리지 요청 패턴 파악 |
| **Detailed Status Code Metrics** | 200OKStatusCount, 403ForbiddenErrorCount, 404NotFoundErrorCount | HTTP 오류 패턴 분석 |

---

### Free vs. Advanced (Paid)

| 항목 | Free | Advanced (Paid) |
| --- | --- | --- |
| 자동 제공 여부 | ✅ 모든 고객 자동 제공 | 추가 비용 |
| 기본 Metrics 수 | ~28개 | 더 많은 추가 지표 |
| 데이터 보존 기간 | **14일** | **15개월** |
| CloudWatch 연동 | ❌ | ✅ 추가 비용 없이 CloudWatch 게시 |
| Prefix 레벨 집계 | ❌ | ✅ |
| Advanced Cost Optimization / Data Protection | ❌ | ✅ |

---

## 13. Requester Pays

- 기본: Bucket 소유자가 모든 S3 Storage 비용 + 데이터 전송 비용 부담
- **Requester Pays 활성화 시**: 요청자(Requester)가 데이터 다운로드 요청 비용 부담
- 대용량 Dataset을 다른 계정과 공유할 때 유용
- **요청자는 반드시 AWS에 인증(Authenticated)** 되어야 함 — 익명(Anonymous) 접근 불가

```
일반 Bucket:        [요청자] →(무료 다운로드)→ [S3]  → 비용: Bucket Owner 부담
Requester Pays:     [요청자] →(유료 다운로드)→ [S3]  → 비용: Requester 부담
```

---

## 14. 핵심 요약 & 시험 포인트

```
S3 Architecture Overview
├── 보안: IAM Policy + Bucket Policy + Block Public Access (+ ACL)
├── 암호화: SSE-S3(기본) / SSE-KMS(감사 필요) / SSE-C(고객 키) / CSE
│          → 상세: AWS_S3_Security_Notes.md 참고
├── 가용성: Versioning + CRR/SRR Replication
├── 비용: Storage Class + Lifecycle Rules
├── 성능: Multipart Upload + Transfer Acceleration + Prefix 분산
└── 운영: Event Notifications + Batch Operations + Storage Lens

Storage Class 비용 순서 (비쌈 → 저렴)
Standard > Standard-IA > One Zone-IA > Glacier Instant > Glacier Flexible > Deep Archive

Glacier 최소 보존 기간
├── Instant / Flexible Retrieval: 90일
└── Deep Archive:                 180일

Glacier Flexible Retrieval 검색 시간
├── Expedited: 1~5분
├── Standard:  3~5시간
└── Bulk:      5~12시간 (무료)

Glacier Deep Archive 검색 시간
├── Standard: 12시간
└── Bulk:     48시간
```

---

### 📌 시험 자주 출제 포인트 총정리

| 포인트 | 내용 |
| --- | --- |
| Bucket 이름 | **전 세계 고유**, 소문자/숫자/하이픈만, 리전 레벨 생성 |
| 최대 Object 크기 | **5TB** |
| 5GB 초과 업로드 | **Multipart Upload 필수** |
| 기본 암호화 (2023~) | **SSE-S3 자동 적용**, 추가 비용 없음 |
| SSE-C 필수 요건 | **HTTPS** (HTTP 요청 시 거부) |
| SSE-KMS Cross-Account | **Customer Managed Key(CMK) 필수** |
| Bucket Policy vs Default Encryption | **Bucket Policy가 먼저 평가됨** |
| MFA Delete 설정 권한 | **Bucket Owner (Root 계정)만** 가능 |
| Versioning Suspend | 기존 버전 **삭제 안 됨** |
| 버저닝 전 파일 Version ID | `null` |
| Replication 전제조건 | 소스+대상 모두 **Versioning 활성화** |
| 기존 객체 복제 | **S3 Batch Replication** 사용 |
| Replication 체이닝 | **없음** (Bucket 1→2→3 자동 복제 안 됨) |
| Transfer Acceleration | CloudFront **Edge Location** 경유 |
| Baseline Performance | Prefix당 PUT 3,500/s, GET 5,500/s |
| S3 Select | SQL로 **서버 사이드** 필터링, 전송 비용 절감 |
| S3 Event → SNS/SQS/Lambda | **Resource-based Policy** 필요 (IAM Role 아님) |
| S3 Event + EventBridge | 18개 이상 Destination, 고급 필터링, Replay 가능 |
| S3 Analytics 대상 | **Standard → Standard-IA만** (Glacier 불가) |
| S3 Analytics 소요 시간 | **24~48시간** |
| Storage Lens Default Dashboard | **삭제 불가** (비활성화만 가능) |
| Storage Lens 데이터 보존 | Free: 14일 / Advanced: **15개월** |
| Requester Pays | 요청자 비용 부담, **익명 접근 불가** |
| Object Lock | **WORM**, Versioning 필수 |
| Compliance Mode | **Root 포함 누구도** 수정/삭제 불가 |
| Governance Mode | 특별 권한(`s3:BypassGovernanceRetention`) 있으면 수정 가능 |
| Legal Hold | 보존 기간 무관, 무기한 잠금 |
| Glacier Vault Lock | 한 번 설정 후 **변경 불가** |
| Static Website 403 | Block Public Access 또는 Bucket Policy 미설정 |
| One Zone-IA | 단일 AZ, **AZ 파괴 시 데이터 손실** |
| Intelligent-Tiering 장점 | **검색(Retrieval) 요금 없음** |
| Pre-signed URL 유효 기간 | Console 12h / CLI 최대 **604800초(7일)** |

---

## 📚 참고 자료

- [S3 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [S3 Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)
- [S3 Storage Lens](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens.html)
- [S3 Batch Operations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html)
- [S3 Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)