---
title: 🔐 AWS S3 Security
published: 2026-03-20
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🔐 AWS S3 Security

> S3 보안의 모든 것 — Encryption, Access Control, Data Protection
> 
> 
> AWS SAA 시험에서 S3 관련 문제의 절반 이상이 이 파일의 내용에서 출제됩니다.
> 
---
## 목차
1. [S3 Encryption Overview](#1-s3-encryption-overview)
2. [SSE-S3 (S3-Managed Keys)](#2-sse-s3-s3-managed-keys)
3. [SSE-KMS (KMS-Managed Keys)](#3-sse-kms-kms-managed-keys)
4. [SSE-C (Customer-Provided Keys)](#4-sse-c-customer-provided-keys)
5. [DSSE-KMS (Dual-Layer SSE)](#5-dsse-kms-dual-layer-sse)
6. [Client-Side Encryption (CSE)](#6-client-side-encryption-cse)
7. [Encryption in Transit](#7-encryption-in-transit)
8. [Default Encryption vs. Bucket Policy 우선순위](#8-default-encryption-vs-bucket-policy-우선순위)
9. [S3 Bucket Key](#9-s3-bucket-key)
10. [S3 Access Logs](#10-s3-access-logs)
11. [Pre-signed URLs](#11-pre-signed-urls)
12. [MFA Delete](#12-mfa-delete)
13. [S3 Object Lock & Glacier Vault Lock](#13-s3-object-lock-glacier-vault-lock)
14. [CORS (Cross-Origin Resource Sharing)](#14-cors-cross-origin-resource-sharing)
15. [S3 Access Points](#15-s3-access-points)
16. [S3 Object Lambda](#16-s3-object-lambda)
17. [VPC Endpoint for S3](#17-vpc-endpoint-for-s3)
18. [핵심 요약 & 시험 포인트](#18-핵심-요약--시험-포인트)
19. [참고 자료](#-참고-자료)
---

## 1. S3 Encryption Overview

### 암호화 유형 한눈에 비교

| 유형 | 키 관리 주체 | 키 저장 | 요청 Header | 특징 |
| --- | --- | --- | --- | --- |
| **SSE-S3** | AWS | AWS 내부 | `x-amz-server-side-encryption: AES256` | 기본값, 무료 |
| **SSE-KMS** | AWS KMS | KMS | `x-amz-server-side-encryption: aws:kms` | CloudTrail 감사, 추가 비용 |
| **DSSE-KMS** | AWS KMS | KMS | `x-amz-server-side-encryption: aws:kms:dsse` | 이중 암호화, 규정 준수 |
| **SSE-C** | Customer | Customer (AWS에 저장 안 함) | HTTPS 필수, 매 요청마다 키 전달 | 고객 완전 통제 |
| **CSE** | Customer | Customer | - | 업로드 전 클라이언트 측 암호화 |

**선택 가이드:**

```
별도 요구사항 없음                → SSE-S3 (기본값)
키 사용 감사 로그 필요 (규정 준수)  → SSE-KMS (Customer Managed Key)
이중 암호화 규정 준수 필요         → DSSE-KMS
키를 완전히 직접 통제             → SSE-C
AWS가 원본 데이터를 절대 보면 안 됨 → Client-Side Encryption
```

---

## 2. SSE-S3 (S3-Managed Keys)

- AWS가 키를 완전히 생성, 관리, 교체 — 사용자 개입 없음
- **AES-256** 알고리즘 사용
- **2023년 1월 5일부터 모든 신규 Object에 기본 자동 적용** (추가 비용 없음)
- Cross-Account 공유 가능

**요청 방법:**

```
PUT /my-object HTTP/1.1
x-amz-server-side-encryption: AES256
```

**Bucket Policy로 SSE-S3 강제 예시:**

```json
{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
        "StringNotEquals": {
            "s3:x-amz-server-side-encryption": "AES256"
        }
    }
}
```

---

## 3. SSE-KMS (KMS-Managed Keys)

- AWS KMS(Key Management Service)를 통해 키 관리
- SSE-S3 대비 **추가적인 제어와 감사 기능** 제공

**SSE-KMS 장점:**

- 키 사용 이력 → **AWS CloudTrail에 자동 기록** (언제, 누가, 어떤 Key로 접근했는지)
- Customer Managed Key (CMK) 생성/교체/비활성화 직접 제어
- Cross-Account 접근 세밀한 권한 설정

**SSE-KMS 단점 (시험 포인트):**

- Upload 시 `GenerateDataKey` KMS API 호출
- Download 시 `Decrypt` KMS API 호출
- **KMS 쿼터 제한**: 리전별 5,500 / 10,000 / 30,000 req/s
- 고처리량 환경에서 KMS 병목 발생 가능 → Service Quotas Console에서 증가 요청

**요청 방법:**

```
PUT /my-object HTTP/1.1
x-amz-server-side-encryption: aws:kms
x-amz-server-side-encryption-aws-kms-key-id: arn:aws:kms:...  (선택, 생략 시 AWS Managed Key 사용)
```

**Bucket Policy로 SSE-KMS 강제 예시:**

```json
{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
        "StringNotEquals": {
            "s3:x-amz-server-side-encryption": "aws:kms"
        }
    }
}
```

**Cross-Account 고려사항:**

- AWS Managed Key (aws/s3): 같은 계정에서만 사용 가능
- **다른 계정과 SSE-KMS 데이터 공유**: 반드시 **Customer Managed Key(CMK)** 사용

---

## 4. SSE-C (Customer-Provided Keys)

- 고객이 키를 직접 생성/관리, AWS는 **암호화/복호화 작업만** 수행
- AWS는 키를 **저장하지 않음** — 요청 처리 후 즉시 메모리에서 삭제
- 고객이 키 분실 시 데이터 복구 **완전 불가**

**필수 요건:**

- **반드시 HTTPS** 사용 (HTTP 요청 시 S3가 거부)
- 모든 Upload/Download 요청마다 HTTP Header로 키 전달

```
PUT /my-object HTTP/1.1
x-amz-server-side-encryption-customer-algorithm: AES256
x-amz-server-side-encryption-customer-key: [Base64 인코딩된 키]
x-amz-server-side-encryption-customer-key-MD5: [키의 MD5]
```

:::warning
⚠️ AWS 콘솔에서 SSE-C 객체 접근 불가 (CLI/SDK만 가능 — 키를 매번 전달해야 하므로)
:::

---

## 5. DSSE-KMS (Dual-Layer SSE)

- KMS 키를 사용한 **이중(Dual-Layer) 서버 사이드 암호화**
- AES-256을 **두 번** 독립적으로 적용:
    1. AWS KMS Data Encryption Key로 1차 암호화
    2. 별도 S3 관리 암호화 키로 2차 암호화
- 엄격한 규정 준수(Compliance) 요구사항이 있는 환경용
- SSE-KMS보다 높은 비용 및 지연 시간 증가

---

## 6. Client-Side Encryption (CSE)

- 데이터를 S3에 Upload하기 **전** 클라이언트 측에서 직접 암호화
- AWS SDK의 **Amazon S3 Client-Side Encryption Library** 활용
- AWS는 암호화된 데이터만 수신 — **원본 데이터에 접근 불가**
- 고객이 키와 암호화 라이프사이클 **완전 관리**

```
[클라이언트]                    [S3]
  원본 데이터
      │ 클라이언트 측 암호화
      ▼
  암호화된 데이터  ──HTTPS──→  암호화된 데이터 저장
                              (S3는 내용 모름)
```

---

## 7. Encryption in Transit

- S3는 두 가지 Endpoint 제공:
    - **HTTP Endpoint**: 암호화 없음
    - **HTTPS Endpoint**: In-flight Encryption (SSL/TLS)
- HTTPS 권장, **SSE-C에서는 HTTPS 필수**
- 대부분의 클라이언트/SDK는 기본적으로 HTTPS 사용

### HTTPS 강제 (aws:SecureTransport)

```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": [
            "arn:aws:s3:::my-bucket",
            "arn:aws:s3:::my-bucket/*"
        ],
        "Condition": {
            "Bool": {
                "aws:SecureTransport": "false"
            }
        }
    }]
}
```

:::tip
`aws:SecureTransport: false`를 Deny하면 HTTP 요청 차단. Bucket과 Object 모두 Resource에 포함해야 함.
:::
---

## 8. Default Encryption vs. Bucket Policy 우선순위

:::tip
**Bucket Policy가 Default Encryption보다 먼저 평가됨**
:::

```
Object Upload 요청
        │
        ▼
1️⃣ Bucket Policy 평가
   (Deny 조건에 해당하면 요청 거부)
        │ Policy 통과
        ▼
2️⃣ Default Encryption 적용
   (Header 없으면 기본 암호화 설정으로 처리)
```

**실제 시나리오:**

- Default Encryption을 SSE-KMS로 설정해도, Bucket Policy에서 `aws:kms` Header가 없는 요청을 Deny하면 **정책이 우선 적용**
- `x-amz-server-side-encryption` Header 없이 Upload하면 → Default Encryption 적용 (SSE-S3 또는 SSE-KMS)

---

## 9. S3 Bucket Key

- **SSE-KMS 사용 시 KMS API 호출 횟수를 줄이기 위한 기능**
- Bucket Key가 활성화되면:
    - S3가 버킷 레벨에서 단기 Bucket-level KMS Key를 생성
    - 개별 Object마다 KMS 호출 대신 Bucket Key로 Data Key 생성
    - KMS 호출 수 대폭 감소 → **비용 절감 + CloudTrail 로그 감소**

```
Bucket Key 비활성화:
  Object 1  →  KMS API Call  →  Data Key 1
  Object 2  →  KMS API Call  →  Data Key 2
  Object 3  →  KMS API Call  →  Data Key 3  (많은 KMS 비용)

Bucket Key 활성화:
  Bucket  →  KMS API Call  →  Bucket Key
    Object 1  →  (Bucket Key로)  →  Data Key 1
    Object 2  →  (Bucket Key로)  →  Data Key 2
    Object 3  →  (Bucket Key로)  →  Data Key 3  (KMS 비용 대폭 감소)
```

---

## 10. S3 Access Logs

- 감사(Audit) 목적으로 S3에 대한 **모든 요청을 다른 S3 Bucket에 로그**로 기록
- 어떤 계정, 어떤 요청(인가 여부 포함)이든 모두 기록
- 로그 데이터를 Athena 등 데이터 분석 도구로 분석 가능
- **로깅 대상 Bucket과 로그 저장 Bucket은 같은 AWS Region에 있어야 함**

:::warning
❌ 절대 금지: 모니터링 대상 Bucket = 로그 저장 Bucket
   → Logging Loop 발생 → Bucket 크기 지수적 증가

✅ 올바른 구성:
   [모니터링 Bucket]  →  Access Logs  →  [별도 Logging Bucket]
:::
---

## 11. Pre-signed URLs

- **제한된 시간 동안만 유효**한 S3 객체 접근 URL
- S3 Console, AWS CLI, SDK로 생성 가능
- URL을 받은 사용자는 **URL 생성자의 권한(Permission)을 그대로 상속** (GET/PUT)
- Bucket을 Public으로 열지 않고 **일시적으로 특정 사용자에게 접근 허용**

### 유효 시간

| 생성 방법 | 유효 시간 |
| --- | --- |
| S3 Console | 최소 1분, 최대 **12시간 (720분)** |
| AWS CLI | 기본 3,600초, 최대 **604,800초 (7일)** |

### 활용 예시

```
[앱 서버] ─ Pre-signed URL 생성 ─→ [사용자 브라우저]
                                          │
                                          ▼
                                    [S3 객체 직접 접근]
                                    (Bucket은 Private 유지,
                                     유효 시간 내에만 접근 가능)
```

**Use Cases:**

- 로그인한 사용자에게만 프리미엄 영상 다운로드 허용
- 동적으로 생성되는 업로드 URL (사용자 Profile 이미지 업로드)
- 일시적인 파일 공유

---

## 12. MFA Delete

- Versioning이 활성화된 Bucket에서 중요 작업 시 **MFA(Multi-Factor Authentication) 코드 요구**

### MFA가 필요한 작업

```
✅ MFA 필요:
  - Object Version 영구 삭제 (Permanently delete an object version)
  - Versioning 비활성화 또는 Suspend

❌ MFA 불필요:
  - Versioning 활성화
  - Delete Marker 추가 (= 일반 Delete 작업)
  - Version 목록 조회
```

### 설정 규칙

| 항목 | 내용 |
| --- | --- |
| **설정 가능 주체** | **Bucket Owner (Root 계정)만** 가능 — IAM User 불가 |
| **전제 조건** | Bucket에 **Versioning 활성화** 필수 |
| **설정 방법** | CLI 또는 SDK만 가능 (Console 미지원) |

---

## 13. S3 Object Lock & Glacier Vault Lock

### S3 Object Lock

- **WORM (Write Once Read Many)** 모델 적용
- 지정된 기간 동안 Object Version 삭제/수정 불가
- **Versioning 활성화 필수**

**Retention Mode 비교:**

| Mode | 설명 | Root 포함 삭제 가능? |
| --- | --- | --- |
| **Compliance Mode** | 어떤 사용자도(Root 포함) 수정/삭제 불가. 보존 기간 단축 불가 | ❌ 절대 불가 |
| **Governance Mode** | 대부분 사용자는 불가. `s3:BypassGovernanceRetention` 권한 보유자만 가능 | ✅ 특별 권한자만 |

**Retention Period:**

- Object를 고정 기간 동안 보호
- 기간 연장(Extend)은 가능, 단축(Shorten)은 Compliance Mode에서 불가

**Legal Hold:**

- 보존 기간과 무관하게 **무기한 잠금**
- `s3:PutObjectLegalHold` IAM 권한으로 자유롭게 적용/해제 가능
- 예: 소송 진행 중 증거 보전

### Glacier Vault Lock

- S3 Glacier Vault에 **Vault Lock Policy (JSON)** 적용
- WORM 모델로 아카이브 데이터 보호
- **한 번 Lock되면 누구도 정책 변경/삭제 불가** — Root 포함
- 규정 준수(Compliance) 및 데이터 보존(Data Retention) 요구사항 충족

```
[Vault Lock Policy 작성]
        │
        ▼
[24시간 이내 검증 기간] ← 이 기간 중에만 정책 수정 가능
        │ 검증 완료 후 Lock 실행
        ▼
[정책 영구 잠금] ← 이후 절대 변경 불가
```

---

## 14. CORS (Cross-Origin Resource Sharing)

### 기본 개념

- **Origin** = Scheme (Protocol) + Host (Domain) + Port
    - 예: `https://example.com` (scheme=https, host=example.com, port=443)
- 웹 브라우저의 Same-Origin Policy: 다른 Origin의 리소스 요청 기본 차단
- CORS Headers로 다른 Origin에서의 요청 허용 가능

### Same Origin vs. Cross Origin

```
Same Origin   (허용):  https://example.com/app1  →  https://example.com/app2
Cross Origin  (차단):  https://example.com        →  https://other.com
```

### S3에서 CORS 설정이 필요한 경우

1. **S3 정적 웹사이트에서 다른 S3 Bucket의 리소스 참조**
2. **웹 앱(다른 도메인)이 S3 Bucket의 파일을 직접 요청**

### CORS 설정 예시 (S3 Bucket에 적용)

```xml
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>https://www.example.com</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
  </CORSRule>
</CORSConfiguration>
```

- `<AllowedOrigin>`: 허용할 출처 (로 전체 허용 가능)
- `<AllowedMethod>`: 허용할 HTTP 메서드
- `<AllowedHeader>`: 허용할 Header
- `<MaxAgeSeconds>`: Pre-flight 결과 캐시 시간

:::tip
CORS 설정은 **리소스를 제공하는 Bucket** (파일이 있는 쪽)에 설정. 요청하는 쪽이 아님.
:::
---

## 15. S3 Access Points

### 개념

- S3 Bucket에 대한 **독립적인 접근 포인트** 생성
- 각 Access Point마다 **고유한 DNS 이름 + 독립적인 Access Point Policy** 보유
- 대규모 Bucket의 보안 관리 단순화 (Bucket Policy 복잡성 해소)

```
[S3 Bucket]
    │
    ├── [Access Point: /finance/*]
    │       Access Point Policy: Finance 팀만 읽기/쓰기
    │
    ├── [Access Point: /analytics/*]
    │       Access Point Policy: Analytics 팀만 읽기
    │
    └── [Access Point: /logs/*]
            Access Point Policy: Logging 서비스만 쓰기
```

### Access Point Policy

- Bucket Policy와 유사한 JSON 형식
- Access Point Policy + Bucket Policy 모두 허용해야 접근 가능

### VPC Origin (VPC 전용 Access Point)

```
[EC2 - Private VPC]
        │
        ▼
[VPC Endpoint (Gateway or Interface)]
        │
        ▼
[S3 Access Point (VPC Origin)]
        │
        ▼
[S3 Bucket]
```

- VPC 내부에서만 접근 가능한 Access Point 정의
- **VPC Endpoint 생성 필수**
- VPC Endpoint Policy에서 대상 Bucket과 Access Point에 대한 접근 허용 필요

---

## 16. S3 Object Lambda

### 개념

- S3에서 Object를 반환할 때 **Lambda Function이 데이터를 변환**한 후 반환
- **원본 Object는 변경 없음** — 요청자에게만 변환된 결과 제공
- Bucket 1개로 여러 형태의 데이터 제공 가능

### 아키텍처

```
                    [원본 S3 Bucket]
                           │
            ┌──────────────┼─────────────────┐
            │              │                 │
    [일반 Access Point] [Object Lambda   [Object Lambda
                          Access Point 1]   Access Point 2]
                               │                  │
                         [Lambda: PII         [Lambda: XML
                          Redaction]           → JSON 변환]
                               │                  │
                         [Analytics App]    [Modern App]
```

### 주요 Use Cases

| Use Case | 설명 |
| --- | --- |
| **PII Redaction** (개인정보 마스킹) | 분석용/비운영 환경에서 개인정보 제거 후 반환 |
| **Format Conversion** | XML → JSON, CSV → Parquet 등 포맷 변환 |
| **Image Processing** | 요청자에 맞게 이미지 리사이징, 워터마크 삽입 |

---

## 17. VPC Endpoint for S3

- S3를 **인터넷 경유 없이** VPC 내부에서 직접 접근
- **Gateway Endpoint** 타입 사용 — **무료**
- Private Subnet의 EC2가 NAT Gateway 없이 S3 접근 가능

```
❌ 인터넷 경유 방식:
[EC2 - Private Subnet] → [NAT Gateway] → [Internet] → [S3]
              (NAT Gateway 비용 + 데이터 전송 비용 발생)

✅ VPC Gateway Endpoint 방식:
[EC2 - Private Subnet] → [VPC Gateway Endpoint] → [S3]
              (비용 없음, 인터넷 경유 없음)
```

**설정 구성요소:**

- Route Table에 S3용 Prefix List 경로 추가 (자동)
- VPC Endpoint Policy로 접근 가능한 Bucket/Action 제한 가능

:::tip
S3와 DynamoDB는 **Gateway Endpoint** (무료). 다른 서비스는 Interface Endpoint (비용 발생).
:::
---

## 18. 핵심 요약 & 시험 포인트

### 암호화 결정 트리

```
암호화 필요?
├── 아니요 → 그래도 SSE-S3 기본 적용됨 (2023년~)
└── 예
    ├── 키 감사 로그 필요? (CloudTrail)
    │   ├── 예 → SSE-KMS (CMK)
    │   │         KMS 쿼터 고려, S3 Bucket Key로 비용 절감
    │   └── 아니요 → SSE-S3 (무료, 간단)
    ├── 이중 암호화 규정 준수?  → DSSE-KMS
    ├── 키를 직접 통제/관리?   → SSE-C (HTTPS 필수)
    └── AWS가 원본 못 봐야 함?  → Client-Side Encryption
```

### 보안 레이어 평가 순서

```
Object Upload 요청
    │
    1️⃣ Bucket Policy 평가 (Deny 조건 확인)
    │
    2️⃣ Default Encryption 적용 (Header 없으면 기본값)
    │
    3️⃣ IAM Policy 확인 (User/Role 권한)
```

---

### 📌 시험 자주 출제 포인트 총정리

| 포인트 | 내용 |
| --- | --- |
| S3 기본 암호화 (2023~) | **SSE-S3 자동 적용**, 추가 비용 없음 |
| SSE-S3 Header | `x-amz-server-side-encryption: AES256` |
| SSE-KMS Header | `x-amz-server-side-encryption: aws:kms` |
| SSE-C 필수 요건 | **HTTPS** (HTTP 요청 시 S3가 거부) |
| SSE-C 키 저장 | **AWS는 키 저장 안 함**, 매 요청 시 Header로 전달 |
| SSE-KMS Cross-Account | **Customer Managed Key(CMK) 필수** |
| SSE-KMS 병목 | KMS 쿼터 초과 가능, **S3 Bucket Key로 KMS 호출 감소** |
| Bucket Policy 평가 | Default Encryption보다 **먼저 평가** |
| HTTPS 강제 정책 | `aws:SecureTransport: false` → Deny |
| Access Logs Loop 방지 | 모니터링 Bucket ≠ 로그 저장 Bucket **(절대 같아선 안 됨)** |
| Pre-signed URL 유효 기간 | Console **12h** / CLI **최대 7일 (604,800초)** |
| Pre-signed URL 권한 | **생성자의 권한 상속** |
| MFA Delete 설정 주체 | **Root 계정(Bucket Owner)만** 가능 |
| MFA Delete 전제 | **Versioning 활성화** 필수 |
| MFA Delete 필요 작업 | 영구 버전 삭제, Versioning Suspend |
| MFA Delete 불필요 작업 | Versioning 활성화, Delete Marker 추가 |
| Object Lock 전제 | **Versioning 활성화** 필수 |
| Compliance Mode | **Root 포함 누구도** 수정/삭제 불가 |
| Governance Mode | `s3:BypassGovernanceRetention` 권한자만 수정 가능 |
| Legal Hold | **보존 기간과 무관**, 무기한 잠금 |
| Glacier Vault Lock | **한 번 Lock 후 변경 불가** (Root 포함) |
| CORS 설정 위치 | **리소스를 제공하는 Bucket**에 설정 (요청하는 쪽 아님) |
| Access Points | 각자 **독립 DNS + 독립 Policy** |
| VPC Access Point | VPC Endpoint + Endpoint Policy 모두 설정 필요 |
| S3 Object Lambda | **원본 변경 없음**, Lambda가 변환 후 반환 |
| VPC Endpoint 타입 | S3, DynamoDB → **Gateway Endpoint (무료)** |

---

## 📚 참고 자료

- [S3 Encryption 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html)
- [SSE-KMS 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html)
- [S3 Object Lock 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [S3 Access Points 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points.html)
- [S3 Object Lambda 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transforming-objects.html)
- [S3 Glacier Vault Lock 공식 문서](https://docs.aws.amazon.com/amazonglacier/latest/dev/vault-lock.html)