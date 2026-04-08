---
title: 🏛️ AWS Identity & Access Management — Advanced
published: 2026-03-22
tags: [AWS, Cloud, Certificates]
category: AWS
draft: true
---
# 🏛️ AWS Identity & Access Management — Advanced

> AWS Organizations · IAM Conditions · Permission Boundaries
> 
> 
> IAM Identity Center · Directory Services · Control Tower
---
## 목차

1. [AWS Organizations](#aws-organizations)
2. [IAM Conditions (정책 조건)](#iam-conditions-정책-조건)
3. [IAM Roles vs. Resource-Based Policies](#iam-roles-vs-resource-based-policies)
4. [IAM Permission Boundaries (권한 경계)](#iam-permission-boundaries-권한-경계)
5. [AWS IAM Identity Center (구: AWS SSO)](#aws-iam-identity-center-구-aws-sso)
6. [Microsoft Active Directory (AD) & AWS Directory Services](#microsoft-active-directory-ad--aws-directory-services)
7. [AWS Control Tower](#aws-control-tower)
8. [전체 Identity 서비스 선택 가이드](#전체-identity-서비스-선택-가이드)
9. [📌 시험 자주 출제 포인트](#-시험-자주-출제-포인트)
---

## AWS Organizations

- **Global Service** — 여러 AWS 계정을 중앙 관리
- **Management Account**: 최상위 계정 (결제 권한 보유)
- **Member Accounts**: 하나의 Organization에만 소속 가능

### 주요 장점

| 항목 | 내용 |
| --- | --- |
| **Consolidated Billing** | 모든 계정 비용을 단일 결제 수단으로 통합 |
| **Volume Discount** | 사용량 합산으로 EC2, S3 등 볼륨 할인 |
| **Reserved Instance 공유** | 미사용 RI를 다른 계정에서 공유 |
| **API 자동화** | API로 계정 자동 생성 가능 |

### OU (Organizational Units)

```
Root
├── Management Account
├── OU: Production
│     ├── Account A
│     └── Account B
├── OU: Development
│     └── Account C
└── OU: Security
      └── Account D (Log/Audit)
```

---

### SCP (Service Control Policies)

- OU 또는 계정에 적용하는 **IAM Policy 형태의 최대 권한 상한선**
- **Management Account에는 적용되지 않음** (항상 Full Admin)
- Allow/Deny 모두 명시 가능 — **기본적으로 아무것도 허용하지 않음 (IAM과 동일)**
- Root에서 각 OU를 거쳐 Target Account까지 **명시적 Allow가 있어야** 효과 발생

```
실제 유효 권한 =
  SCP(Org 레벨) AND SCP(OU 레벨) AND IAM Identity Policy
```

> 💡 **Use Case**: 특정 리전 잠금, 특정 서비스 사용 금지, 루트 계정 작업 제한
> 

---

### Tag Policies

- Organization 전체에서 **태그 표준화** 강제
- 특정 서비스/리소스에 비준수 태깅 작업 방지
- 태그가 없는 리소스에는 영향 없음
- AWS Cost Allocation Tags + ABAC(Attribute-based Access Control)와 연계
- 비준수 리소스 리포트 생성 + EventBridge로 비준수 태그 모니터링

---

## IAM Conditions (정책 조건)

주요 조건 키:

| 조건 키 | 설명 | 예시 |
| --- | --- | --- |
| `aws:SourceIp` | API 호출 출처 IP 제한 | 회사 IP에서만 접근 허용 |
| `aws:RequestedRegion` | API 호출 대상 리전 제한 | ap-northeast-2만 허용 |
| `ec2:ResourceTag` | 태그 기반 접근 제한 | Environment=prod 인스턴스만 |
| `aws:MultiFactorAuthPresent` | MFA 적용 여부 | MFA 없으면 차단 |

### S3 버킷 vs. 객체 권한 레벨

```
Bucket 레벨: arn:aws:s3:::test
  → s3:ListBucket 적용

Object 레벨: arn:aws:s3:::test/*
  → s3:GetObject, s3:PutObject, s3:DeleteObject 적용
```

### aws:PrincipalOrgId

- 모든 **Resource Policy**에서 사용 가능
- AWS Organization의 멤버 계정에서 오는 요청만 허용

```json
"Condition": {
    "StringEquals": {
        "aws:PrincipalOrgID": "o-xxxxxxxxxx"
    }
}
```

---

## IAM Roles vs. Resource-Based Policies

### Cross-Account 접근 시 두 가지 방법

| 방법 | 특징 |
| --- | --- |
| **IAM Role Assume** | Role을 Assume하면 **원래 권한을 포기**하고 Role 권한만 가짐 |
| **Resource-based Policy** | Principal이 **원래 권한을 유지**하면서 리소스에 접근 |

**중요한 시나리오:**

```
Account A의 User가 Account A의 DynamoDB를 스캔하고
Account B의 S3에 결과를 저장해야 할 때:

→ IAM Role Assume 불가 (Account A DynamoDB 권한 포기해야 해서)
→ Account B S3에 Resource-based Policy로 Account A User 직접 허용 ← 정답
```

### EventBridge — 서비스별 권한 방식

| Target | 필요한 권한 방식 |
| --- | --- |
| Lambda, SNS, SQS, S3, API Gateway | **Resource-based Policy** |
| Kinesis Stream, EC2 ASG, SSM Run Command, ECS Task | **IAM Role** |

---

## IAM Permission Boundaries (권한 경계)

- **User와 Role**에만 적용 (Groups에는 불가)
- Managed Policy로 IAM 엔티티가 가질 수 있는 **최대 권한 상한선** 설정

```
실제 유효 권한 =
  Permission Boundary AND Identity-based Policy
```

**Organizations SCP와 함께 사용 시:**

```
유효 권한 = Organizations SCP AND Permission Boundary AND IAM Policy
```

### Permission Boundary Use Cases

| Use Case | 설명 |
| --- | --- |
| 비관리자에게 권한 위임 | 새 IAM User를 만들 수 있되 Permission Boundary 내에서만 |
| 개발자 Self-service | 자신의 정책을 직접 관리 가능, 단 권한 에스컬레이션(관리자 권한 획득) 방지 |
| 특정 User만 제한 | Organizations/SCP 대신 단일 User에게만 적용 |

---

## AWS IAM Identity Center (구: AWS SSO)

- **Single Sign-On**: 한 번 로그인으로 모든 것에 접근
- 지원 대상:
    - AWS Organizations의 모든 계정
    - 비즈니스 클라우드 앱 (Salesforce, Box, Microsoft 365)
    - SAML 2.0 지원 앱
    - EC2 Windows Instance

### Identity Providers (IdP)

| 유형 | 내용 |
| --- | --- |
| **Built-in** | IAM Identity Center 내장 Identity Store |
| **3rd Party** | Active Directory (AD), Okta 등 |

### Fine-grained 권한 및 할당

**Multi-Account Permissions:**

- **Permission Sets**: IAM Policy 컬렉션 → 사용자/그룹에 할당 → AWS 계정 접근 정의

**Application Assignments:**

- 많은 SAML 2.0 비즈니스 앱에 SSO 접근
- URL, 인증서, 메타데이터 제공

**ABAC (Attribute-Based Access Control):**

- 사용자 속성(부서, 직책, 로케일) 기반 세밀한 권한
- 권한을 한 번 정의 → 속성 변경으로 AWS 접근 수정

---

## Microsoft Active Directory (AD) & AWS Directory Services

### Active Directory 개념

- Windows Server에서 운영, AD Domain Services 포함
- 사용자 계정, 컴퓨터, 프린터, 파일 공유, 보안 그룹의 중앙 관리 DB
- 객체(Object)는 Tree로 구성, Tree 그룹 = Forest

### AWS Directory Services 3가지

| 서비스 | 설명 | On-premises AD |
| --- | --- | --- |
| **AWS Managed Microsoft AD** | AWS에서 자체 AD 운영, MFA 지원 | Trust 연결 가능 |
| **AD Connector** | On-premises AD로 요청 프록시 (Gateway 역할) | 사용자는 On-premises에서 관리 |
| **Simple AD** | AD 호환 관리형 디렉터리 | On-premises AD와 연결 불가 |

### IAM Identity Center + AD 통합 방법

**방법 1: AWS Managed Microsoft AD 사용 (바로 통합)**

```
[IAM Identity Center] ↔ [AWS Managed Microsoft AD]
→ Out-of-the-box 통합
```

**방법 2: Self-Managed (On-premises) AD 사용**

```
방법 A: Two-way Trust
[IAM Identity Center] ↔ [AWS Managed AD] ↔(Trust)↔ [On-premises AD]

방법 B: AD Connector (더 높은 Latency)
[IAM Identity Center] ↔ [AD Connector] ↔(Proxy)↔ [On-premises AD]
```

---

## AWS Control Tower

- 보안/규정 준수 Multi-Account AWS 환경을 **빠르게 설정 및 관리**
- **AWS Organizations를 사용**하여 계정 생성

### 주요 장점

| 항목 | 내용 |
| --- | --- |
| **자동화** | 환경 설정 클릭 몇 번으로 자동 완료 |
| **Policy 관리** | Guardrail로 지속적 정책 관리 |
| **규정 준수 모니터링** | 인터랙티브 대시보드 |
| **위반 자동 수정** | 정책 위반 감지 및 remediation |

### Guardrails

| 유형 | 수단 | 예시 |
| --- | --- | --- |
| **Preventive (예방)** | **SCP** | 특정 리전 외 리소스 생성 금지 |
| **Detective (탐지)** | **AWS Config** | 태그 없는 리소스 식별 |

---

## 전체 Identity 서비스 선택 가이드

```
AWS 계정 다수 관리         → AWS Organizations + SCP
한 번 로그인으로 전체 접근  → IAM Identity Center (SSO)
외부 사용자 앱 인증         → Amazon Cognito User Pools
AWS 리소스 직접 접근 권한   → Amazon Cognito Identity Pools
Windows AD 마이그레이션     → AWS Managed Microsoft AD
On-premises AD 연동 프록시  → AD Connector
AD 없는 단순 호환 디렉터리  → Simple AD
Multi-Account 거버넌스       → AWS Control Tower
```

---

## 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| SCP 적용 대상 | **Management Account 제외** 모든 계정/OU |
| SCP 기본 동작 | **아무것도 허용하지 않음** (명시적 Allow 필요) |
| Tag Policy 영향 | **태그 없는 리소스에는 영향 없음** |
| Permission Boundary 적용 | **User와 Role만** (Groups 불가) |
| 유효 권한 공식 | **SCP AND Boundary AND IAM Policy** |
| Cross-Account: Role vs Resource Policy | Role: 원래 권한 포기 / Resource Policy: **원래 권한 유지** |
| EventBridge Lambda/SNS/SQS 권한 | **Resource-based Policy** |
| EventBridge Kinesis/ECS 권한 | **IAM Role** |
| IAM Identity Center 구 이름 | **AWS Single Sign-On (SSO)** |
| IAM Identity Center + AD 바로 통합 | **AWS Managed Microsoft AD** |
| AD Connector 역할 | **프록시** (사용자는 On-premises에서 관리) |
| Simple AD 제한 | **On-premises AD와 연결 불가** |
| Control Tower 계정 생성 방법 | **AWS Organizations** 사용 |
| Preventive Guardrail 수단 | **SCP** |
| Detective Guardrail 수단 | **AWS Config** |
| aws:PrincipalOrgID 조건 | Organization 멤버 계정에서 오는 요청만 허용 |