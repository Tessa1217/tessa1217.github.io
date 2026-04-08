---
title: 🔐 AWS IAM (Identity and Access Management)
published: 2026-03-14
tags: [AWS, Cloud, Certificates]
category: AWS
draft: true
---
# 🔐 AWS IAM (Identity and Access Management)

> IAM은 AWS 리소스에 대한 접근을 안전하게 제어하는 **Global Service (글로벌 서비스)**입니다.
> 
> 
> "누가(Who) → 무엇을(What) → 어떤 조건에서(Condition) 할 수 있는가"를 정의합니다.
> 

---

## 목차

1. [IAM 기본 개념](#1-iam-기본-개념)
2. [IAM 정책 (Policies)](#2-iam-정책-policies)
3. [IAM MFA & 비밀번호 정책](#3-iam-mfa-비밀번호-정책)
4. [AWS 접근 방법](#4-aws-접근-방법)
5. [IAM 역할 (Roles)](#5-iam-역할-roles)
6. [IAM 보안 도구](#6-iam-보안-도구)
7. [Best Practices](#7-best-practices)
8. [핵심 요약](#8-핵심-요약)
9. [참고 자료](#-참고-자료)

---

## 1. IAM 기본 개념

### 🌍 AWS Region vs IAM

| 구분 | 특징 |
| --- | --- |
| **AWS Region** | 특정 지역에 종속 (Compliance, 지연속도, 가용 서비스, 가격 고려) |
| **IAM** | **글로벌 서비스** — 리전에 종속되지 않음 |

> **Region 선택 기준**
> 
> - **Compliance**: 데이터 주권 (Data governance) / 법적 요구사항 (Legal requirements)
> - **Proximity**: 고객과의 물리적 거리 (지연속도 - Reduced Latency)
> - **Available Services**: 신규 서비스는 일부 리전에서만 먼저 제공
> - **Pricing**: 리전마다 가격 상이

---

### 👤 Root Account

- AWS 계정 생성 시 자동으로 만들어지는 **최고 권한 계정**
- **일상적인 작업에 절대 사용 금지** — 초기 설정 및 루트만 가능한 작업에만 사용
- 루트 계정으로만 가능한 작업 예시: 결제 정보 변경, 계정 삭제, Support Plan 변경

---

### 👥 Users & Groups

```
AWS Account
├── Root User (최상위, 공유 금지)
├── Group: Developers
│   ├── User: Alice
│   └── User: Bob
├── Group: Admins
│   └── User: Charlie
└── User: Dave (그룹 미소속도 가능)
```

- **User**: 조직 내 개별 사람 또는 애플리케이션
- **Group**: User들의 집합 (그룹 안에 그룹 중첩 불가)
- 하나의 User는 **여러 Group에 동시에 소속** 가능
- User는 반드시 Group에 속할 필요 없음 (단, 권장하지 않음)

---

## 2. IAM 정책 (Policies)

### 📋 정책의 종류

| 정책 유형 | 설명 | 사용 시기 |
| --- | --- | --- |
| **AWS Managed Policy** | AWS가 관리하는 사전 정의 정책 | 빠른 시작, 일반적인 Use Case |
| **Customer Managed Policy** | 직접 생성/관리하는 정책 | 세밀한 권한 제어 필요 시 |
| **Inline Policy** | 특정 User/Group/Role에 직접 첨부 | 1:1 대응, 재사용 불필요 시 |

### 🔗 정책 상속 구조

```
Group: Developers  ──── Policy A (S3 Full Access)
    └── User: Alice  ── Policy B (EC2 Read, Inline)

Group: QA  ──────────── Policy C (CloudWatch Read)
    └── User: Alice (중복 소속)

→ Alice가 최종적으로 갖는 권한: Policy A + Policy B + Policy C
```

---

### 📄 IAM 정책 JSON 구조

```json
{
    "Version": "2012-10-17",
    "Id": "S3-Account-Permissions",
    "Statement": [
        {
            "Sid": "AllowS3Access",
            "Effect": "Allow",
            "Principal": {
                "AWS": ["arn:aws:iam::123456789012:root"]
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": ["arn:aws:s3:::mybucket/*"],
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "ap-northeast-2"
                }
            }
        }
    ]
}
```

**각 필드 설명:**

| 필드 | 필수 | 설명 |
| --- | --- | --- |
| `Version` | 권장 | 정책 언어 버전 (항상 `"2012-10-17"` 사용) |
| `Id` | 선택 | 정책 식별자 |
| `Statement` | **필수** | 권한 규칙 배열 |
| `Sid` | 선택 | Statement 식별자 |
| `Effect` | **필수** | `Allow` 또는 `Deny` |
| `Principal` | 조건부 | 정책이 적용될 대상 (Resource-based policy에서 사용) |
| `Action` | **필수** | 허용/거부할 API 작업 목록 |
| `Resource` | **필수** | 작업이 적용될 AWS 리소스 ARN |
| `Condition` | 선택 | 정책이 적용되는 조건 |

> 💡 **최소 권한 원칙 (Least Privilege Principle)**
> 
> 
> 사용자에게 필요한 최소한의 권한만 부여합니다.
> 
> 불필요한 권한은 공격 표면(Attack Surface)을 넓힙니다.
> 

---

### 🏷️ 정책 예시: EC2 + CloudWatch 읽기 권한

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ec2:Describe*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudwatch:ListMetrics",
                "cloudwatch:GetMetricStatistics",
                "cloudwatch:Describe*"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## 3. IAM MFA & 비밀번호 정책

### 🔑 비밀번호 정책 설정 항목

- 최소 길이 지정
- 특정 문자 유형 필수 포함 (대문자, 숫자, 특수문자 등)
- IAM 사용자의 자체 비밀번호 변경 허용 여부
- 비밀번호 만료 기간 설정 (정기적 변경 강제)
- 이전 비밀번호 재사용 방지

---

### 📱 MFA (Multi-Factor Authentication)

**MFA = 알고 있는 것 (비밀번호) + 소유한 것 (기기)**

비밀번호가 탈취되더라도, 물리적 기기 없이는 계정 접근 불가.

**MFA 기기 옵션:**

| 유형 | 제품 | 특징 |
| --- | --- | --- |
| **Virtual MFA** | Google Authenticator, Authy | 단일 기기에 다수 계정 등록 가능 |
| **U2F Security Key** | YubiKey (Yubico) | 물리적 USB 키, 다수 계정 지원 |
| **Hardware Key Fob** | Gemalto 제품 | 전용 하드웨어 OTP |
| **GovCloud용 Key Fob** | SurePassID 제품 | AWS GovCloud(US) 전용 |

> 💡 **실무 권장**: Root 계정과 관리자 계정에는 반드시 MFA 활성화.
> 
> 
> 콘솔 접근 권한이 있는 모든 IAM 사용자에게도 MFA 강제 적용 권장.
> 

---

## 4. AWS 접근 방법

### 세 가지 접근 방식

| 방법 | 보호 수단 | 주요 용도 |
| --- | --- | --- |
| **Management Console** | 비밀번호 + MFA | 사람이 직접 사용 |
| **CLI (Command Line Interface)** | Access Key | 스크립트, 자동화 |
| **SDK (Software Development Kit)** | Access Key | 애플리케이션 코드 내 |

---

### 🔑 Access Key 관리

- AWS 콘솔에서 사용자가 직접 생성
- **비밀번호처럼 취급** — 절대 공유 금지, 코드에 하드코딩 금지
- Access Key ID = 사용자 이름 / Secret Access Key = 비밀번호 (비유)

```bash
# Access Key 설정 예시 (CLI)
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: ap-northeast-2
# Default output format: json
```
:::warning
Access Key는 장기 자격증명(Long-term Credentials)입니다.
가능하면 <strong>임시 자격증명(IAM Role + STS)</strong>을 사용하고, 불가피한 경우 90일 주기로 교체(Rotation)하세요.
:::
---

### 🖥️ AWS CLI

- AWS 서비스의 Public API에 직접 접근
- 반복 작업 자동화 및 스크립트화 가능
- AWS SDK(Python용 Boto3) 기반으로 빌드됨

```bash
# 주요 CLI 예시
aws iam list-users                         # 사용자 목록
aws iam create-user --user-name newuser    # 사용자 생성
aws iam attach-user-policy \               # 사용자에 정책 연결
  --user-name newuser \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

---

### 📦 AWS SDK

- 언어별 라이브러리 제공 (Python Boto3, Java, Node.js, Go 등)
- 애플리케이션 내에 임베드되어 AWS 서비스 호출
- Mobile SDK (iOS, Android), IoT Device SDK도 지원

```python
# Python Boto3 예시
import boto3

iam = boto3.client('iam')
response = iam.list_users()
for user in response['Users']:
    print(user['UserName'])
```

---

## 5. IAM 역할 (Roles)

### 🎭 역할이란?

IAM Role은 **사람이 아닌 AWS 서비스**가 다른 서비스에 접근할 때 사용하는 임시 자격증명입니다.

```
EC2 Instance
    └── IAM Role (S3 Read Access 부여)
            └── S3 Bucket 접근 가능
```

Access Key를 EC2에 직접 저장하는 것보다 **훨씬 안전**합니다.

---

### 주요 Role 유형

| Role 유형 | 설명 | 예시 |
| --- | --- | --- |
| **EC2 Instance Role** | EC2가 AWS 서비스 접근 시 | EC2 → S3 파일 읽기 |
| **Lambda Function Role** | Lambda 실행 시 필요한 권한 | Lambda → DynamoDB 쓰기 |
| **CloudFormation Role** | 인프라 배포 권한 | Stack 생성 시 리소스 접근 |
| **Cross-Account Role** | 다른 계정의 리소스 접근 | 멀티 계정 환경 |

---

### 💡 Role vs User 비교

| 구분 | IAM User | IAM Role |
| --- | --- | --- |
| 자격증명 | 장기 (비밀번호/Access Key) | **임시** (STS 토큰, 기본 1시간) |
| 대상 | 사람 | 서비스, 앱, 다른 계정 |
| 보안 수준 | 낮음 (키 유출 위험) | **높음** (자동 만료) |
| 권장 여부 | 사람만, 최소화 | **워크로드에는 반드시 사용** |

---

### 🔐 Permissions Boundary (권한 경계)

- Role이나 User가 가질 수 있는 **최대 권한의 상한선** 설정
- 예: DevOps 엔지니어에게 Role 생성 권한은 주되, dev 환경으로만 제한

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "ap-northeast-2"
                }
            }
        }
    ]
}
```

---

## 6. IAM 보안 도구

### 🔍 IAM Credentials Report (계정 수준)

- **위치**: IAM 콘솔 → Credential Report
- 계정 내 **모든 사용자**의 자격증명 상태를 CSV로 다운로드
- 확인 항목: 마지막 로그인, MFA 활성화 여부, Access Key 사용 일자, 비밀번호 교체 일자

```bash
# CLI로 생성 및 다운로드
aws iam generate-credential-report
aws iam get-credential-report --query Content --output text | base64 -d
```

---

### 🔍 IAM Access Advisor (사용자 수준)

- **위치**: IAM 콘솔 → 특정 User 선택 → Access Advisor 탭
- 사용자에게 부여된 서비스 권한과 **마지막 접근 시간** 확인
- 오랫동안 사용하지 않은 권한 → 제거 검토

---

### 🔍 IAM Access Analyzer

- 외부 공개 또는 교차 계정 접근을 허용하는 리소스 자동 탐지
- 정책 검증 기능: 100개 이상의 체크 항목으로 정책 오류 탐지
- **신기능 (2025)**: 미사용 접근(Unused Access) 분석 → 최소 권한 달성 지원

---

## 7. Best Practices

### ✅ 기본 보안 원칙

| 항목 | 내용 |
| --- | --- |
| **Root 계정 보호** | MFA 설정 후 사용 최소화, 자격증명 잠금 보관 |
| **1인 1계정** | 자격증명 절대 공유 금지 |
| **그룹으로 권한 관리** | 개별 User에 직접 정책 첨부 지양 |
| **최소 권한 원칙** | 필요한 작업에 딱 맞는 권한만 부여 |
| **MFA 강제 적용** | 콘솔 접근 모든 계정, 특히 관리자/루트 |
| **Access Key 최소화** | 장기 키 대신 Role + 임시 자격증명 사용 |
| **정기 감사** | Credentials Report + Access Advisor 주기적 검토 |

---

### 🏗️ 실무 아키텍처 Best Practice

### 1. 사람 사용자 → Federation (SSO) 권장

> AWS IAM 공식 문서는 사람 사용자에게 직접 IAM User를 생성하는 대신,
> 
> 
> **AWS IAM Identity Center (구 AWS SSO)** 또는 외부 IdP(Okta, Azure AD, Google Workspace)를 통한 Federation을 권장합니다.
> 

```
개발자 (Okta 로그인)
    └── IAM Identity Center
            └── IAM Role Assume → 임시 자격증명 발급
                    └── AWS 리소스 접근
```

**장점:**

- 퇴사자 계정 한 곳에서 일괄 비활성화
- 장기 Access Key 없음 (자동 만료 토큰)
- CloudTrail에서 개인 추적 가능

---

### 2. 워크로드 → 반드시 IAM Role 사용

```python
# ❌ 나쁜 예: Access Key를 코드에 하드코딩
import boto3
client = boto3.client(
    's3',
    aws_access_key_id='AKIAIOSFODNN7EXAMPLE',      # 절대 금지!
    aws_secret_access_key='wJalrXUtnFEMI...'        # 절대 금지!
)

# ✅ 좋은 예: EC2 Instance Role 사용 (자동으로 인증)
import boto3
client = boto3.client('s3')  # Role에서 자동으로 자격증명 가져옴
```

---

### 3. Access Key 사용 불가피한 경우

- **90일 이내 주기적 교체** (Rotation)
- 환경변수 또는 `~/.aws/credentials` 파일로 관리
- 코드/레포지토리에 절대 커밋 금지
- 불필요한 Key는 즉시 비활성화 및 삭제

```bash
# Access Key 교체 절차
# 1. 새 Key 생성
aws iam create-access-key --user-name myuser

# 2. 애플리케이션에 새 Key 적용 및 테스트

# 3. 기존 Key 비활성화
aws iam update-access-key --access-key-id OLD_KEY_ID --status Inactive

# 4. 문제 없으면 삭제
aws iam delete-access-key --access-key-id OLD_KEY_ID
```

---

### 4. 정책 작성 팁

```json
// ✅ 좋은 예: 특정 리소스 + 조건 지정
{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/uploads/*",
    "Condition": {
        "StringEquals": {"aws:RequestedRegion": "ap-northeast-2"},
        "Bool": {"aws:MultiFactorAuthPresent": "true"}
    }
}

// ❌ 나쁜 예: 과도하게 넓은 권한
{
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
}
```

---

### 5. 멀티 계정 환경 (AWS Organizations)

- 서비스별/팀별/환경별(Dev/Staging/Prod) 계정 분리
- **SCP (Service Control Policy)**: 조직 전체 권한 상한선 설정
- 계정 간 접근은 Cross-Account Role로만 허용

```
AWS Organization
├── Root OU
│   ├── Management Account (빌링만)
│   ├── Security OU
│   │   └── Security Account (로그, 감사)
│   ├── Dev OU
│   │   └── Dev Account
│   └── Prod OU
│       └── Prod Account (SCP로 위험 작업 차단)
```

---

### 🚨 자주 하는 실수 (Anti-Patterns)

| 실수 | 문제점 | 해결책 |
| --- | --- | --- |
| Root 계정으로 일상 작업 | 전체 계정 탈취 위험 | 별도 Admin IAM User 사용 |
| `Action: "*"` 남발 | 과도한 권한 부여 | 필요한 Action만 명시 |
| Access Key 코드 내 하드코딩 | 키 유출 시 즉각 위협 | IAM Role / 환경변수 사용 |
| 오래된 Access Key 방치 | 미사용 키 유출 위험 | 주기적 감사 및 삭제 |
| 개인별 정책 관리 | 일관성 없는 권한 관리 | Group 또는 Role 기반으로 통합 |
| MFA 미설정 | 비밀번호만으로 접근 가능 | 모든 콘솔 접근 계정에 MFA 적용 |

---

## 8. 핵심 요약

```
IAM 구성 요소
├── Users       → 사람 또는 앱, 콘솔 비밀번호 또는 Access Key
├── Groups      → User들의 집합, 그룹 중첩 불가
├── Policies    → JSON 형식의 권한 문서
├── Roles       → 서비스/앱을 위한 임시 자격증명
└── Root User   → 최고 권한, 일상 작업에 사용 금지

접근 방법
├── Console    → 비밀번호 + MFA
├── CLI        → Access Key
└── SDK        → Access Key

보안 도구
├── Credentials Report → 계정 전체 자격증명 현황
├── Access Advisor     → 사용자별 미사용 권한 파악
└── Access Analyzer    → 외부 노출 및 미사용 권한 자동 탐지
```

---

## 📚 참고 자료

- [AWS IAM 공식 문서](https://docs.aws.amazon.com/iam/)
- [AWS IAM Best Practices (공식)](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS IAM Identity Center (구 SSO)](https://aws.amazon.com/iam/identity-center/)
- [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
- [AWS Organizations SCP](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)