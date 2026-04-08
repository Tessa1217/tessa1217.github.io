---
title: 🔐 IAM Security Best Practices
published: 2026-03-27
tags: [AWS, Cloud, Certificates]
category: AWS
draft: true
---
# 🔐 IAM Security Best Practices

> **출처:** AWS IAM 공식 문서 — Security best practices in IAM
>
> **URL:** https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
>
> **목적:** SAA-C03 시험 대비 핵심 개념 정리
---
## 목차
1. [1. Identity 관리 원칙](#1-identity-관리-원칙)
2. [2. Credential 보안 관리](#2-credential-보안-관리)
3. [3. 최소 권한 원칙 (Least Privilege)](#3-최소-권한-원칙-least-privilege)
4. [4. MFA (다중 인증)](#4-mfa-다중-인증)
5. [5. IAM Policy 구성 전략](#5-iam-policy-구성-전략)
6. [6. 크로스 계정 & 외부 접근 제어](#6-크로스-계정--외부-접근-제어)
7. [7. 모니터링 & 감사](#7-모니터링--감사)
8. [8. 루트 계정 보호](#8-루트-계정-보호)
9. [9. 시험 단골 패턴 요약](#9-시험-단골-패턴-요약)
---

## 1. Identity 관리 원칙

### 핵심 원칙: 임시 자격증명 사용

> **장기 자격증명(Long-term credentials) 대신 임시 자격증명(Temporary credentials)을 사용하라.**
> 

```
Long-term credentials  = IAM User Access Key  → 보안 리스크 높음
Temporary credentials  = IAM Role (STS)        → 권장 방식
```

---

### 1-1. 휴먼 사용자 — Federation + IdP 사용

**권장 방식:**

- 휴먼 사용자(개발자, 관리자, 운영자)에게 개별 IAM User 생성 **금지** 권고
- 대신 **외부 IdP(Identity Provider)와 페더레이션** 사용

**구현 방법:**

| 상황 | 권장 방식 |
| --- | --- |
| 다중 계정 중앙 관리 (권장) | **AWS IAM Identity Center (SSO)** |
| 단일 계정 소규모 | IAM + SAML 2.0 또는 OIDC 직접 페더레이션 |
| 모바일/웹앱 사용자 | Amazon Cognito Identity Pools |

**지원 프로토콜:**

- **SAML 2.0:** Active Directory Federation Services(ADFS), Shibboleth 등
- **OIDC:** GitHub Actions 등 AWS 외부에서 실행되는 워크플로우

**이점:**

- 사내 디렉토리(AD 등)와 통합 → 별도 IAM User 관리 불필요
- 장기 Access Key 배포/관리 불필요
- 계정 탈퇴 시 IdP에서 비활성화만 하면 AWS 접근 자동 차단
:::tip
- "사내 AD 기반 SSO로 AWS 접근" → **IAM Identity Center + SAML**
- "GitHub Actions가 AWS 리소스에 접근" → **OIDC 페더레이션**
- 중앙 집중식 다중 계정 접근 관리 → **IAM Identity Center** (구 AWS SSO)
:::

---

### 1-2. 워크로드 — IAM Role 사용

**원칙:** EC2, Lambda 등 AWS 리소스는 **IAM Role** 사용 (Access Key 절대 금지)

```
EC2 인스턴스     → Instance Profile (IAM Role)
Lambda 함수      → Execution Role (IAM Role)
ECS 태스크       → Task Role (IAM Role)
GitHub Actions   → OIDC → IAM Role AssumeRole
```

**왜 Role인가?**

- STS(Security Token Service)가 **임시 자격증명** 자동 발급·갱신
- Access Key가 코드/환경변수에 하드코딩되는 리스크 제거
- AWS SDK가 Instance Profile / 환경 체이닝으로 **자동** 자격증명 획득
:::tip
- "EC2에서 S3 접근" → EC2에 IAM Role 부여 (Access Key 코드 내 설정 X)
- IAM User가 필요한 예외 케이스: IAM Role을 지원하지 않는 레거시 시스템
:::

---

## 2. Credential 보안 관리

### 2-1. Access Key 관리

| 원칙 | 내용 |
| --- | --- |
| **장기 키 최소화** | 가능한 IAM Role로 대체. 불가피할 경우만 IAM User Access Key 사용 |
| **정기 로테이션** | 필요한 경우 주기적으로 갱신. AWS Config 룰로 90일 미사용 키 탐지 |
| **미사용 키 제거** | 45일 이상 미사용 → 비활성화 또는 삭제 (CIS Benchmark 권고) |
| **하드코딩 금지** | 코드, 환경변수, Git 저장소에 Access Key 절대 포함 금지 |

**감지 도구:**

- **IAM Credential Report:** 계정 전체 자격증명 현황 보고서
- **IAM Last Accessed Information:** 마지막 서비스/액션 사용 시간
- **AWS Config Rules:** 특정 일수 이상 미사용 키 자동 탐지

---

### 2-2. Secrets 관리 Best Practice

```
DB 패스워드, API Key 등 시크릿 → AWS Secrets Manager
                                   (자동 로테이션 지원)

단순 설정값, 비시크릿 파라미터  → SSM Parameter Store Standard
                                   (무료)
```
:::tip
RDS 패스워드 자동 로테이션 → **Secrets Manager**
:::

---

## 3. 최소 권한 원칙 (Least Privilege)

### 원칙 정의

> **작업 수행에 필요한 최소한의 권한만 부여하라.** 초과 권한은 의도치 않은 동작 및 보안 사고의 원인.
> 

### 구현 단계

```
1단계: AWS Managed Policy로 시작 (빠른 설정)
          ↓
2단계: 실제 사용 패턴 파악 (CloudTrail 로그 분석)
          ↓
3단계: IAM Access Analyzer로 최소 권한 Policy 자동 생성
          ↓
4단계: Customer Managed Policy로 세밀한 권한 적용
```

---

### IAM Access Analyzer 활용

| 기능 | 설명 |
| --- | --- |
| **Policy Generation** | CloudTrail 로그 기반으로 실제 사용 권한만 추출한 Policy 자동 생성 |
| **Policy Validation** | 100개 이상의 정책 검사. 과도하게 허용적인 권한에 보안 경고 |
| **External Access Findings** | 외부(다른 계정/인터넷)에서 접근 가능한 리소스 탐지 |

:::tip
- "CloudTrail 기반으로 최소 권한 자동 생성" → **IAM Access Analyzer Policy Generation**
- "외부 접근 가능 리소스 탐지" → **IAM Access Analyzer External Access**
:::
---

### 흔한 실수: 와일드카드 권한

```json
// ❌ 절대 금지 — Full Admin 권한
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// ✅ 올바른 방식 — 특정 서비스·리소스·조건 명시
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "Bool": {"aws:SecureTransport": "true"}
  }
}
```
:::tip
`"Action": "*"` + `"Resource": "*"` = **보안 위반**. 제거 또는 세분화 필요
:::

---

### Permission Boundary (권한 경계)

- IAM Identity/Role에 설정 가능한 **최대 허용 권한 상한선**
- 개발자가 자신보다 높은 권한의 Role/User 생성하는 **권한 상승(Privilege Escalation) 방지**
- Permission Boundary를 벗어난 권한은 실제 Policy에 있어도 거부됨

```
Permission Boundary = 울타리
Identity-based Policy = 울타리 안에서의 실제 권한

→ 두 Policy의 교집합만 유효
```
:::tip
개발자에게 IAM 관리 권한 위임 + 권한 상승 방지 → **Permission Boundary**
:::
---

## 4. MFA (다중 인증)

### MFA 적용 원칙

| 대상 | 요구 수준 |
| --- | --- |
| **루트 계정** | Hardware MFA 강력 권고 (YubiKey 등) |
| **특권 IAM User** (관리자) | MFA 필수 |
| **일반 IAM User** | MFA 강력 권고 |
| **Console 로그인** | MFA 조건 Policy로 강제 가능 |

### MFA 강제 Policy 패턴

```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}
```

→ MFA 없이 로그인 시 모든 작업 거부

**MFA 기기 유형:**

- **Virtual MFA:** Google Authenticator, Authy (소프트웨어)
- **Hardware MFA:** YubiKey, Gemalto (물리 장치) — 루트 계정 권장
- **FIDO Security Key:** FIDO2/WebAuthn 지원 하드웨어 키

:::tip
- 루트 계정 + 강력 보안 → **Hardware MFA**
- MFA 없이 API 접근 차단 → `aws:MultiFactorAuthPresent: false` 조건으로 Deny
:::

---

## 5. IAM Policy 구성 전략

### 5-1. Policy 평가 로직 (시험 핵심)

```
1. Explicit Deny       → 있으면 즉시 거부 (최우선)
2. Organizations SCP   → OU/계정 단위 guardrail
3. Resource-based Policy (+ Identity-based Policy)
4. Permission Boundary → 최대 허용 상한
5. Session Policy      → AssumeRole 시 추가 제한
6. Identity-based Policy
                        ↓
결론: Default = Implicit Deny
      모든 레이어 통과 + Explicit Allow 있을 때만 허용
```
:::tip
"Explicit Deny는 어떤 Allow보다 우선" — 평가 순서 암기 필수
:::

---

### 5-2. Policy 유형 비교

| 유형 | 부착 대상 | 특징 |
| --- | --- | --- |
| **Identity-based** | User, Group, Role | 가장 일반적. 인프라 권한 정의 |
| **Resource-based** | S3, SQS, KMS 등 리소스 | 크로스 계정 접근 허용 가능 |
| **Permission Boundary** | User, Role | 최대 허용 권한 상한 설정 |
| **SCP** | OU, 계정 (Organizations) | Guardrail. Allow 부여 안 함. 허용 범위만 제한 |
| **Session Policy** | AssumeRole 호출 시 | 임시 세션의 권한 추가 제한 |

---

### 5-3. User vs Group vs Role 운영 Best Practice

```
IAM User   → 원칙적으로 사람에게 생성 지양 (Federation 우선)
              불가피하면 최소 권한 + 주기적 자격증명 감사

IAM Group  → 다수 User에 동일 Policy 일괄 적용
              User에게 직접 Policy 부착 금지 → Group을 통해 관리

IAM Role   → AWS 리소스, 크로스 계정 접근, 페더레이션 사용자
              임시 자격증명 자동 발급 (STS AssumeRole)
```
:::tip
- Policy를 **직접 User에 부착하지 않고 Group 통해 관리** = 운영 복잡도 감소 Best Practice
- 계정 수 증가 → Group/Role 기반 관리가 필수
:::
---

## 6. 크로스 계정 & 외부 접근 제어

### 6-1. SCP (Service Control Policies)

- **AWS Organizations**에서 OU 또는 계정 단위로 적용
- **Allow를 부여하지 않음** — 최대 허용 범위(Guardrail)만 설정
- 루트 계정도 SCP를 벗어날 수 없음

**활용 예시:**

```
프로덕션 OU → SCP: us-east-1, ap-northeast-2 리전만 허용
개발 OU     → SCP: 고비용 GPU 인스턴스 유형 차단
전체 ORG    → SCP: 특정 서비스(Redshift 등) 생성 금지
```
:::tip
- 특정 리전 차단 + 조직 전체 → **SCP**
- SCP는 Allow 부여 안 함. 기존 Allow 권한의 허용 범위를 **제한**
:::
---

### 6-2. IAM Identity Center (AWS SSO)

- 다중 계정 접근을 **중앙에서 단일 포털**로 관리
- Identity Source: 내장 디렉토리 / 외부 SAML IdP / Active Directory
- Permission Set = 계정별 권한 집합 정의
- 장점: 각 계정에 별도 IAM User 생성 불필요

---

### 6-3. 외부 접근 감사

**IAM Access Analyzer:**

- S3 버킷, IAM Role, KMS Key, Lambda 등의 **외부 접근 가능 여부** 자동 탐지
- 의도치 않은 퍼블릭/크로스 계정 공개 즉시 발견
:::tip
"외부에서 접근 가능한 리소스 자동 탐지" → **IAM Access Analyzer** 
:::

---

## 7. 모니터링 & 감사

### 7-1. AWS CloudTrail

- **모든 AWS API 호출** 기록 (누가 / 언제 / 무엇을 / 어디서)
- S3 버킷에 로그 저장 → **Log File Validation** 활성화 (무결성 검증)
- IAM 보안 감사의 기본 기반
:::tip
"누가 리소스를 변경했는가" → **CloudTrail**
:::

---

### 7-2. 불필요한 자격증명 정기 정리

**정리 대상:**

- 미사용 IAM User, Role, Group
- 만료된 Policy 및 권한
- 90일 이상 미사용 Access Key
- 45일 이상 미사용 패스워드/Access Key (CIS 기준)

**도구:**

- **IAM Credential Report:** 계정 전체 자격증명 현황 CSV 다운로드
- **IAM Last Accessed:** 서비스/액션별 마지막 사용 시간
- **AWS Config Rules:** 자동 탐지 및 알림

:::tip
"미사용 권한 자동 탐지 + 최소 권한 정책 생성" → **IAM Access Analyzer + Last Accessed 정보 조합** 
:::

---

### 7-3. AWS Config

- **리소스 구성 변경 이력** 추적
- Compliance Rules:
    - `iam-user-no-policies-check`: User에 직접 Policy 부착 여부
    - `iam-root-access-key-check`: 루트 Access Key 존재 여부
    - `mfa-enabled-for-iam-console-access`: MFA 미설정 User 탐지
- **Auto-remediation:** 비준수 시 Lambda 통해 자동 수정 가능

---

## 8. 루트 계정 보호

### 루트 계정 보안 수칙

| 항목 | 권고 사항 |
| --- | --- |
| **일상 사용** | 절대 금지 — 루트 전용 작업 외 사용 금지 |
| **Access Key** | 생성하지 말 것. 기존 것 즉시 삭제 |
| **MFA** | Hardware MFA 필수 활성화 |
| **이메일** | 루트 계정 이메일 보안 강화 (2FA 적용) |
| **비밀번호** | 복잡한 비밀번호 + 안전한 저장소 보관 |

**루트 계정만 할 수 있는 작업:**

- 계정 해지
- 결제 정보 변경
- Support 플랜 변경
- IAM Identity Center 활성화 (조직 관리 계정)
- S3 버킷 정책이 모든 요청을 차단할 때 버킷 정책 삭제

### Centralized Root Access (Organizations)

- AWS Organizations 관리 계정에서 멤버 계정의 **루트 자격증명 중앙 제어**
- 멤버 계정의 장기 루트 자격증명 제거 및 복구 방지
- 필요 시 임시 특권 세션(Break Glass)으로만 루트 수준 작업 수행

:::tip
- 루트 Access Key = **즉시 삭제**
- "루트 계정 모니터링" → **CloudTrail + CloudWatch 알람** 연동
- AWS Config Rule: `iam-root-access-key-check`
:::

---

## 9. 시험 단골 패턴 요약

| 상황 | 정답 키워드 |
| --- | --- |
| 사내 AD로 AWS 접근 | IAM Identity Center + SAML 2.0 페더레이션 |
| GitHub Actions → AWS 접근 | OIDC + IAM Role AssumeRole |
| EC2에서 S3 접근 | IAM Role (Instance Profile) — Access Key 절대 X |
| 다중 계정 권한 중앙 관리 | IAM Identity Center (Permission Set) |
| OU 단위 서비스 사용 차단 | SCP (Service Control Policy) |
| 개발자 권한 위임 + 상승 방지 | Permission Boundary |
| 미사용 권한 자동 탐지 | IAM Access Analyzer + Last Accessed |
| 최소 권한 Policy 자동 생성 | IAM Access Analyzer Policy Generation (CloudTrail 기반) |
| 외부 접근 가능 리소스 탐지 | IAM Access Analyzer External Access Findings |
| MFA 없이 API 접근 차단 | `aws:MultiFactorAuthPresent: false` Deny Policy |
| API 호출 감사 로그 | CloudTrail |
| 리소스 구성 변경 자동 탐지 | AWS Config Rules |
| 루트 계정 Access Key | 즉시 삭제 (생성 자체 금지) |
| DB 자격증명 자동 로테이션 | AWS Secrets Manager |
| 단순 설정값 저장 | SSM Parameter Store (무료) |

---