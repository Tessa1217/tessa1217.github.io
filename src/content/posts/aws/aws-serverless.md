---
title: ⚡ AWS Serverless
published: 2026-03-22
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# ⚡ AWS Serverless

> Lambda · API Gateway · DynamoDB · Step Functions · Cognito
> 
> 
> 서버를 프로비저닝하지 않고 코드와 함수만 배포하는 패러다임
---
## 목차
1. [Serverless 개요](#serverless-개요)
2. [AWS Lambda](#aws-lambda)
3. [CloudFront Functions vs. Lambda@Edge](#cloudfront-functions-vs-lambdaedge)
4. [Amazon DynamoDB](#amazon-dynamodb)
5. [Amazon API Gateway](#amazon-api-gateway)
6. [AWS Step Functions](#aws-step-functions)
7. [Amazon Cognito](#amazon-cognito)
8. [📌 시험 자주 출제 포인트](#-시험-자주-출제-포인트)
---

## Serverless 개요

**Serverless ≠ 서버가 없음** — 서버를 관리/프로비저닝하지 않는 것

### AWS Serverless 서비스

> AWS Lambda, DynamoDB, Amazon Cognito, API Gateway
>
> Amazon S3, SNS, SQS, Kinesis Data Firehose
>
> Aurora Serverless, Step Functions, Fargate

---

## AWS Lambda

### EC2 vs. Lambda

| 항목 | EC2 | Lambda |
| --- | --- | --- |
| **서버** | 직접 프로비저닝 | 관리 불필요 |
| **실행 방식** | 항상 실행 중 | **On-demand** (요청 시 실행) |
| **시간 제한** | 없음 | **최대 15분 (900초)** |
| **스케일링** | 수동 또는 ASG | **자동** |
| **과금** | 실행 여부 무관 | **요청 수 + 실행 시간** |

---

### Lambda 제한 (Limits) — per Region

> 📌 시험 빈출 수치
> 

**실행 (Execution):**

| 항목 | 제한 |
| --- | --- |
| Memory | **128 MB ~ 10 GB** (1 MB 단위) |
| Max Execution Time | **900초 (15분)** |
| Environment Variables | **4 KB** |
| /tmp 디스크 | **512 MB ~ 10 GB** |
| Concurrent Executions | **1,000** (증가 요청 가능) |

**배포 (Deployment):**

| 항목 | 제한 |
| --- | --- |
| 압축 .zip 크기 | **50 MB** |
| 비압축 코드 + 의존성 | **250 MB** |
| Environment Variables | **4 KB** |

> 💡 RAM을 늘리면 CPU와 네트워크 성능도 함께 향상됨.
> 

---

### Lambda 지원 런타임

- Node.js, Python, Java, C#/PowerShell, Ruby
- **Custom Runtime API**: Rust, Golang 등
- **Lambda Container Image**: Lambda Runtime API를 구현한 컨테이너 이미지 사용 가능
    - 임의의 Docker Image는 **ECS/Fargate 권장** (Lambda Container Image와 다름)

---

### Lambda 가격

| 과금 항목 | 내용 |
| --- | --- |
| **요청 수** | 첫 **1,000,000 req** 무료 / 이후 $0.20 / 100만 req |
| **실행 시간** | 월 **400,000 GB-seconds** 무료 / 이후 $1.00 / 600,000 GB-sec |

> 400,000 GB-sec = 1GB RAM → 400,000초 / 128MB RAM → 3,200,000초
> 

---

### Lambda Concurrency & Throttling (동시 실행 및 조절)

```
계정당 기본 동시 실행 한도: 1,000개
        │
        ▼
Throttle 발생 시:
  ├── Synchronous 호출 → 429 ThrottleError 즉시 반환
  └── Asynchronous 호출 → 자동 재시도 (최대 6시간, 지수 백오프)
                          → 최종 실패 시 DLQ(Dead Letter Queue)
```

**Reserved Concurrency (예약 동시 실행):**

- 특정 함수에 동시 실행 상한을 예약 설정
- 다른 함수가 한도를 다 쓰지 못하도록 보호
- 여러 Lambda가 같은 계정 한도를 공유하므로 **한 함수가 폭발적 요청 시 다른 함수 Throttle 가능**

---

### Cold Start & Provisioned Concurrency

**Cold Start 문제:**

```
새 인스턴스 시작 → 코드 로드 + Init 코드 실행 (handler 외부)
→ 첫 요청의 지연 시간 높음 (대형 패키지/SDK 사용 시 더 심각)
```

**Provisioned Concurrency:**

- 함수 호출 전에 미리 인스턴스를 워밍업 → Cold Start 없음
- Application Auto Scaling으로 스케줄 또는 목표 활용률 기반 관리

---

### Lambda SnapStart (Java / Python / .NET)

- **최대 10배 성능 향상** (추가 비용 없음)
- 새 버전 배포 시 Lambda가 함수를 초기화하고 메모리/디스크 **Snapshot 저장**
- 이후 호출은 저장된 Snapshot에서 즉시 시작 → Init 과정 생략

```
SnapStart 비활성화: invoke → [Init → invoke → shutdown]
SnapStart 활성화:   invoke → [snapshot에서 복원 → invoke → shutdown]
```

---

### Lambda Networking (VPC 연동)

**기본 동작 (Default):**

```
Lambda는 AWS 소유 VPC에서 실행
→ 고객 VPC의 RDS, ElastiCache, 내부 ELB에 접근 불가
```

**Lambda in VPC:**

```
Lambda → ENI(Elastic Network Interface) 생성 → 지정 Subnet에 연결
→ VPC 내 리소스(RDS, ElastiCache 등) 접근 가능
```

- VPC ID, Subnet, Security Group 지정 필요

**Lambda + RDS Proxy:**

```
[많은 Lambda 호출] → [RDS Proxy] → [RDS/Aurora]
(직접 연결 시 DB Connection 과다)  (Connection Pooling으로 보호)
```

- RDS Proxy는 Public 접근 불가 → **Lambda도 반드시 VPC 내 배포 필요**
- IAM 인증 강제 + Secrets Manager 연동

---

### Lambda에서 RDS/Aurora 호출 (Invoke Lambda from DB)

- RDS for PostgreSQL, Aurora MySQL에서 **DB 내 이벤트 기반 Lambda 호출** 가능
- 예: 신규 사용자 INSERT → Lambda → SES로 환영 이메일 발송
- **DB 인스턴스가 Lambda에 아웃바운드 접근 권한 필요** (Public / NAT GW / VPC Endpoint 중 하나)
- DB Instance에 Lambda 호출 권한 필요 (Lambda Resource-based Policy + IAM Policy)

---

### RDS Event Notifications

- DB 인스턴스 **자체의 이벤트** 알림 (데이터 변경 아님)
- 이벤트 카테고리: DB Instance, Snapshot, Parameter Group, Security Group, RDS Proxy, Custom Engine Version
- **Near real-time** (최대 5분 지연)
- SNS 또는 EventBridge로 알림 전송

---

## CloudFront Functions vs. Lambda@Edge

Edge에서 코드를 실행하여 CloudFront 요청/응답을 동적으로 처리.

| 항목 | CloudFront Functions | Lambda@Edge |
| --- | --- | --- |
| **런타임** | JavaScript만 | Node.js, Python |
| **처리량** | **수백만 req/s** | 수천 req/s |
| **최대 실행 시간** | **< 1ms** | **5~10초** |
| **메모리** | 2 MB | 128 MB ~ 10 GB |
| **패키지 크기** | 10 KB | 1 MB ~ 50 MB |
| **트리거** | Viewer Req/Res | Viewer/Origin Req/Res (4가지) |
| **네트워크/파일 접근** | ❌ | ✅ |
| **요청 Body 접근** | ❌ | ✅ |
| **가격** | 더 저렴 (Free Tier 있음) | Free Tier 없음 |
| **코드 작성 위치** | CloudFront 내 | us-east-1 리전 |

**CloudFront Functions Use Cases:** Cache Key 정규화, Header 조작, URL 리다이렉트, JWT 인증
**Lambda@Edge Use Cases:** 복잡한 로직, 외부 API 호출, 파일시스템 접근, 3rd Party 라이브러리

---

## Amazon DynamoDB

### 특징

- **NoSQL**, **Serverless**, **완전 관리형**, Multi-AZ 복제
- 초당 수백만 req, 수조 개 행, 수백 TB 스토리지
- **Single-digit millisecond** 지연 시간
- IAM 통합 보안, 자동 스케일링
- Standard / Infrequent Access (IA) Table Class

### 기본 구조

| 항목 | 내용 |
| --- | --- |
| **Primary Key** | 생성 시 결정 (변경 불가) |
| **Item 최대 크기** | **400 KB** |
| **Attribute** | 언제든 추가 가능, null 허용 → **Schema 유연성** |
| **지원 타입** | Scalar(String, Number, Binary, Boolean, Null), Document(List, Map), Set |

### Read/Write Capacity Modes

| 항목 | Provisioned Mode | On-Demand Mode |
| --- | --- | --- |
| **용량 설정** | RCU/WCU 미리 지정 | 자동 |
| **스케일링** | Auto-scaling 가능 | 자동 |
| **비용** | 프로비저닝 용량 과금 | 실제 사용량 과금 (더 비쌈) |
| **적합 워크로드** | 예측 가능한 트래픽 | **예측 불가, 급격한 Spike** |

### DynamoDB Accelerator (DAX)

- DynamoDB를 위한 **완전 관리형 인메모리 캐시**
- **Microsecond** 지연 시간 (캐시된 데이터)
- 기존 DynamoDB API와 호환 → **애플리케이션 코드 변경 불필요**
- 기본 TTL: **5분**

**DAX vs. ElastiCache:**

| DAX | ElastiCache |
| --- | --- |
| 개별 Object, Query/Scan 캐시 | **집계 결과(Aggregation) 캐시** |
| DynamoDB 전용 | 범용 캐시 |

### DynamoDB Streams

DynamoDB 테이블의 Item 변경(CUD) 이벤트 스트림.

| 항목 | DynamoDB Streams | Kinesis Data Streams |
| --- | --- | --- |
| **보존 기간** | **24시간** | **최대 365일** |
| **Consumer 수** | 제한적 | 많음 |
| **처리 방법** | Lambda Triggers, KCL Adapter | Lambda, Firehose, Analytics, EMR 등 |

**Use Cases:** 실시간 반응, 크로스 리전 복제, Lambda 트리거

### DynamoDB Global Tables

- **Active-Active** 복제 (모든 리전에서 Read/Write 가능)
- 다중 리전에서 **저지연** 접근
- **DynamoDB Streams 활성화 필수**

### DynamoDB TTL (Time To Live)

- 만료 Timestamp 기반 자동 Item 삭제
- Use Cases: 세션 데이터, 규정 준수, 오래된 데이터 정리

### DynamoDB 백업

| 방식 | 내용 |
| --- | --- |
| **PITR (자동 백업)** | 최대 **35일**, 복원 시 새 테이블 생성 |
| **On-Demand Backup** | 명시적 삭제 전까지 유지, 성능 영향 없음, AWS Backup으로 Cross-Region 가능 |

### DynamoDB ↔ S3 연동

| 항목 | Export to S3 | Import from S3 |
| --- | --- | --- |
| **전제 조건** | PITR 활성화 필수 | - |
| **포맷** | DynamoDB JSON / ION | CSV / DynamoDB JSON / ION |
| **용량 소비** | Read Capacity 미사용 | Write Capacity 미사용 |
| **결과** | 기존 테이블 유지 | 새 테이블 생성 |

---

## Amazon API Gateway

| 항목 | 내용 |
| --- | --- |
| **인프라 관리** | 불필요 (Lambda + API Gateway = 완전 Serverless) |
| **주요 기능** | API 버저닝, 환경 관리(dev/prod), 인증/인가, Rate Limiting, API Key, 캐싱, SDK 생성 |
| **지원 프로토콜** | REST, HTTP, WebSocket |

### API Gateway Endpoint Types

| 유형 | 설명 | 특징 |
| --- | --- | --- |
| **Edge-Optimized** (기본) | CloudFront Edge Location 경유 | 글로벌 클라이언트, API는 단일 리전 |
| **Regional** | 동일 리전 클라이언트용 | 수동으로 CloudFront 결합 가능 |
| **Private** | VPC 내부에서만 접근 | Interface VPC Endpoint (ENI) + Resource Policy 필요 |

### API Gateway 인증

| 방법 | Use Case |
| --- | --- |
| **IAM Roles** | 내부 애플리케이션 |
| **Cognito** | 외부 사용자 (모바일 앱 등) |
| **Custom Authorizer** | 자체 인증 로직 (Lambda 기반) |

**ACM (AWS Certificate Manager) 연동:**

- Edge-Optimized: 인증서는 **us-east-1**에 있어야 함
- Regional: 인증서는 **API Gateway와 같은 리전**에 있어야 함
- Route 53에 **CNAME 또는 A-alias 레코드** 설정 필요

---

## AWS Step Functions

- Lambda 함수들을 **시각적 워크플로우**로 오케스트레이션
- 기능: 순차/병렬 실행, 조건 분기, 타임아웃, 에러 핸들링
- EC2, ECS, 온프레미스, API Gateway, SQS, DynamoDB 등과 통합
- **Human Approval** (사람의 승인 단계) 구현 가능
- Use Cases: 주문 처리, 데이터 파이프라인, 웹 애플리케이션 워크플로우

---

## Amazon Cognito

### 목적

외부 사용자(웹/모바일 앱 사용자)에게 **AWS 리소스에 대한 Identity 부여**

> 📌 **Cognito vs. IAM**: 수백 명의 외부 사용자, 모바일 사용자, SAML 인증 → **Cognito**
> 

### Cognito User Pools (CUP)

- 앱 사용자를 위한 **Serverless 사용자 DB**
- 기능: 이메일/비밀번호 로그인, 비밀번호 재설정, 이메일/전화 인증, MFA
- **Federated Identity**: Facebook, Google, SAML 계정으로 로그인
- **API Gateway 및 ALB와 통합**

### Cognito Identity Pools (Federated Identities)

- 사용자에게 **임시 AWS 자격증명(Temporary AWS Credentials)** 발급
- AWS 서비스에 직접 접근 가능 (S3, DynamoDB 등)
- 사용자 소스: Cognito User Pools, 3rd party 로그인
- IAM Policy를 Cognito에서 정의 (user_id 기반 세밀한 권한 제어 가능)
- 인증/미인증 사용자별 기본 IAM Role 설정 가능

### CUP vs. Identity Pools

| 항목 | User Pools | Identity Pools |
| --- | --- | --- |
| **목적** | 로그인/인증 | AWS 리소스 접근 권한 부여 |
| **결과물** | JWT Token | **임시 AWS 자격증명** |
| **통합** | API Gateway, ALB | S3, DynamoDB 직접 접근 |

---

## 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| Lambda 최대 실행 시간 | **15분 (900초)** |
| Lambda 메모리 범위 | **128 MB ~ 10 GB** |
| Lambda 기본 동시 실행 한도 | **1,000개** |
| Throttle (Sync) | **429 ThrottleError** |
| Throttle (Async) | 자동 재시도 최대 **6시간**, 지수 백오프 |
| Cold Start 해결 | **Provisioned Concurrency** |
| SnapStart 지원 런타임 | **Java, Python, .NET** |
| Lambda + RDS Proxy | Lambda는 **VPC 내 배포 필수** |
| CloudFront Functions 실행 시간 | **< 1ms** |
| Lambda@Edge 실행 시간 | **5~10초** |
| CloudFront Functions 트리거 | Viewer **Req/Res만** |
| Lambda@Edge 트리거 | Viewer/Origin **Req/Res (4가지)** |
| DynamoDB Item 최대 크기 | **400 KB** |
| DAX 지연 시간 | **Microsecond** |
| DAX 기본 TTL | **5분** |
| DAX vs ElastiCache | DAX: 개별 Object / ElastiCache: **집계 결과** |
| DynamoDB Streams 보존 | **24시간** |
| Global Tables 전제 | **DynamoDB Streams 활성화** 필수 |
| Global Tables 방식 | **Active-Active** |
| DynamoDB PITR 최대 | **35일** |
| Export to S3 전제 | **PITR 활성화** 필수 |
| Import from S3 결과 | **새 테이블 생성** |
| API Gateway Edge-Optimized ACM | **us-east-1** 인증서 |
| Cognito User Pools 통합 | **API Gateway + ALB** |
| Cognito Identity Pools 결과물 | **임시 AWS 자격증명** |