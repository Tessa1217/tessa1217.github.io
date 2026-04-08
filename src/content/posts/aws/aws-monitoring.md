---
title: 🔍 AWS Monitoring, Troubleshooting & Audit
published: 2026-03-22
tags: [AWS, Cloud, Certificates]
category: AWS
draft: true
---
# 🔍 AWS Monitoring, Troubleshooting & Audit

> CloudWatch · EventBridge · CloudTrail · AWS Config
> 
> 
> 리소스 상태 모니터링, API 감사, 규정 준수 관리의 3대 축
---
## 목차
1. [세 서비스 한눈에 비교](#세-서비스-한눈에-비교)
2. [Amazon CloudWatch](#amazon-cloudwatch)
3. [Amazon EventBridge (구: CloudWatch Events)](#amazon-eventbridge-구-cloudwatch-events)
4. [AWS CloudTrail](#aws-cloudtrail)
5. [AWS Config](#aws-config)
6. [CloudWatch vs. CloudTrail vs. Config 비교](#cloudwatch-vs-cloudtrail-vs-config-비교)
7. [📌 시험 자주 출제 포인트](#-시험-자주-출제-포인트)
---

## 세 서비스 한눈에 비교

| 서비스 | 핵심 역할 | 주요 질문 |
| --- | --- | --- |
| **CloudWatch** | 성능 모니터링 + 로그 + 알람 | "지금 CPU가 얼마나 되나?" |
| **CloudTrail** | API 호출 감사 기록 | "누가 언제 어떤 API를 호출했나?" |
| **AWS Config** | 설정 변경 기록 + 규정 준수 평가 | "리소스 설정이 어떻게 바뀌었나? 규정 준수하나?" |

> 📌 **리소스 삭제 원인 조사 → CloudTrail 먼저 확인**
> 

---

## Amazon CloudWatch

### CloudWatch Metrics

- AWS 모든 서비스에서 지표(Metric) 제공
- **Namespace** 단위로 지표 그룹화
- **Dimension**: 지표의 속성 (Instance ID, 환경 등), 지표당 최대 **30개 Dimension**
- 지표에 Timestamp 포함
- **Custom Metrics** 생성 가능 (예: RAM 사용률 — 기본 제공 안 됨)

### CloudWatch Metric Streams

- CloudWatch Metrics를 **Near real-time**으로 대상에 지속 스트리밍
- 대상: **Kinesis Data Firehose** (→ S3, Redshift 등)
- 3rd party: Datadog, Dynatrace, New Relic, Splunk, Sumo Logic

---

### CloudWatch Logs

| 항목 | 내용 |
| --- | --- |
| **Log Group** | 임의 이름, 보통 애플리케이션 단위 |
| **Log Stream** | 인스턴스/로그파일/컨테이너 단위 |
| **만료 정책** | 영구 보존 ~ 1일~10년 설정 가능 |
| **기본 암호화** | 활성화됨 (KMS 커스텀 키 설정 가능) |

**CloudWatch Logs 전송 대상:**

- Amazon S3 (Export)
- Kinesis Data Streams
- Kinesis Data Firehose
- AWS Lambda
- OpenSearch

**주요 Log Sources:**

- SDK, CloudWatch Logs Agent, CloudWatch Unified Agent
- Elastic Beanstalk, ECS, Lambda, VPC Flow Logs, API Gateway, CloudTrail, Route 53

### CloudWatch Logs Agent vs. Unified Agent

| 항목 | CloudWatch Logs Agent | CloudWatch Unified Agent |
| --- | --- | --- |
| **버전** | 구버전 | **신버전 (권장)** |
| **로그 전송** | CloudWatch Logs만 | CloudWatch Logs |
| **시스템 지표** | ❌ | ✅ (RAM, Process 등 추가 수집) |
| **설정 관리** | - | **SSM Parameter Store** 중앙 관리 |

**Unified Agent 수집 지표:**
CPU, Disk metrics/IO, RAM, Netstat, Processes, Swap Space
→ EC2 기본 Out-of-the-box Metrics(Disk, CPU, Network)보다 **상세한 수준**

### CloudWatch Logs Insights

- CloudWatch Logs에서 **SQL 유사 쿼리로 로그 분석**
- AWS 서비스 및 JSON 로그에서 필드 자동 검색
- 여러 Log Group, 여러 계정 동시 쿼리 가능
- 쿼리 저장 및 Dashboard 추가 가능
- ⚠️ **Query Engine** — 실시간 엔진 아님 (과거 데이터만 조회)

### CloudWatch Logs S3 Export

- 로그 데이터를 S3로 내보내기 (API: `CreateExportTask`)
- 데이터 가용까지 **최대 12시간** 소요
- **Near real-time 또는 실시간 아님** → 실시간이 필요하면 **Subscriptions Filter** 사용

### CloudWatch Logs Subscriptions

- 실시간 로그 이벤트 처리 및 분석용
- 대상: **Kinesis Data Streams, Kinesis Data Firehose, Lambda**
- **Subscription Filter**: 특정 조건의 로그만 대상으로 전달
- **Cross-Account Subscription**: 다른 계정의 KDS/KDF로 전달 가능

---

### CloudWatch Alarms

**Alarm States:**

- `OK` / `INSUFFICIENT_DATA` / `ALARM`

**Alarm Targets:**

- EC2 Instance: 중지/종료/재부팅/복구
- Auto Scaling Action 트리거
- SNS Topic 알림 → 이를 통해 사실상 모든 것 가능

**Composite Alarms:**

- 여러 알람 상태를 AND/OR 조건으로 조합
- "알람 노이즈(Alarm Noise)" 감소에 효과적

**EC2 Instance Recovery:**

- `StatusCheckFailed_System` Alarm → 복구 실행
- 복구 후: **Private/Public/Elastic IP, Metadata, Placement Group 동일 유지**

**알람 테스트 (CLI):**

```bash
aws cloudwatch set-alarm-state \
  --alarm-name "myalarm" \
  --state-value ALARM \
  --state-reason "testing purposes"
```

---

### CloudWatch 특화 Insights 서비스

| 서비스 | 대상 | 기능 |
| --- | --- | --- |
| **Container Insights** | ECS, EKS, EC2 Kubernetes, Fargate | 컨테이너 메트릭 + 로그 수집 (EKS: 컨테이너화된 CW Agent 필요) |
| **Lambda Insights** | Lambda (Lambda Layer 형태) | Cold Start, 시스템 메트릭, 진단 정보 |
| **Contributor Insights** | VPC Flow Logs, DNS 등 모든 CW Logs | Top-N 기여자 파악 (예: 상위 IP, 최다 오류 URL) |
| **Application Insights** | EC2 기반 앱 (Java, .NET 등) + AWS 서비스 | 문제 자동 감지 대시보드, SageMaker 기반 |

---

### CloudWatch Network Synthetic Monitor

- On-premises ↔ AWS 애플리케이션 간 **네트워크 성능 문제 감지**
- Agent 설치 불필요
- ICMP/TCP 트래픽 테스트 (Direct Connect 또는 Site-to-Site VPN 경유)
- 패킷 손실, 지연, Jitter 측정 → CloudWatch Metrics로 발행

---

## Amazon EventBridge (구: CloudWatch Events)

### 핵심 기능

| 기능 | 설명 |
| --- | --- |
| **Cron Jobs** | 특정 시간/주기 기반 스케줄 실행 |
| **Event Pattern** | 특정 이벤트 발생 시 규칙 트리거 |
| **대상** | Lambda, SQS, SNS, KDS, Step Functions, ECS Task, API Gateway, Batch 등 |

### Event Bus 유형

| 유형 | 설명 |
| --- | --- |
| **Default Event Bus** | AWS 서비스 이벤트 수신 |
| **Partner Event Bus** | SaaS 파트너 이벤트 (Zendesk, Datadog 등) |
| **Custom Event Bus** | 커스텀 앱 이벤트 |
- Event Bus는 **Resource-based Policy**로 다른 계정에 공유 가능
- 이벤트 **아카이브(Archive)** + **Replay** 가능

### Schema Registry

- EventBridge가 Event Bus의 이벤트 스키마를 자동 분석/추론
- 스키마 기반 애플리케이션 코드 자동 생성 (이벤트 구조 사전 파악)
- 스키마 버저닝 지원

### EventBridge Resource-based Policy

- 특정 Event Bus의 권한 관리
- 다른 계정/리전의 이벤트 허용/거부
- **Use Case**: AWS Organization 전체 이벤트를 단일 계정/리전으로 집계

### EventBridge 보안 (Target 별 권한)

| Target 유형 | 권한 방식 |
| --- | --- |
| Lambda, SNS, SQS, S3, API Gateway | **Resource-based Policy** |
| Kinesis Streams, EC2 ASG, SSM Run Command, ECS Task | **IAM Role** |

---

## AWS CloudTrail

- AWS 계정 내 **모든 API 호출 기록** (기본 활성화)
- 콘솔, SDK, CLI, AWS 서비스를 통한 호출 포함
- 로그 대상: CloudWatch Logs 또는 S3
- Trail은 **모든 리전** 또는 **단일 리전** 적용 가능

### CloudTrail 이벤트 유형

| 유형 | 기본 기록 | 설명 |
| --- | --- | --- |
| **Management Events** | ✅ | 리소스 작업 (IAM, EC2 서브넷 생성, 로깅 설정 등) |
| **Data Events** | ❌ (고볼륨) | S3 Object 레벨 작업 (GetObject, DeleteObject 등), Lambda 실행 |
| **CloudTrail Insights Events** | 별도 활성화 | 비정상적인 활동 자동 감지 |

### CloudTrail Insights

- 비정상 활동 감지:
    - 부정확한 리소스 프로비저닝
    - 서비스 한도 도달
    - IAM 작업 급증
    - 주기적 유지보수 누락
- 정상 Management Event로 **Baseline 생성** → Write Event 지속 분석
- 이상 탐지 결과: CloudTrail 콘솔 + S3 이벤트 + **EventBridge 이벤트** 생성

### CloudTrail 이벤트 보존

- 기본 보존: **90일**
- 90일 이상 보존: **S3 로그 저장 + Athena 분석** 조합

### CloudTrail + EventBridge 패턴

```
[사용자 API 호출]
      │
      ▼
[CloudTrail 기록]
      │
      ▼
[EventBridge 이벤트]
      │
      ▼
[SNS 알림 또는 자동화]
```

예시:

- `DeleteTable` (DynamoDB) → CloudTrail → EventBridge → SNS 경보
- `AssumeRole` (IAM) → CloudTrail → EventBridge → SNS 경보
- `AuthorizeSecurityGroupIngress` (EC2) → CloudTrail → EventBridge → SNS 경보

---

## AWS Config

- 리소스 설정 변경 기록 및 **규정 준수(Compliance) 평가**
- 리전별 서비스 (리전 간 집계 가능)
- 설정 데이터를 S3에 저장 → Athena로 분석 가능

**답할 수 있는 질문들:**

- SSH가 제한 없이 열려 있는 Security Group이 있나?
- Public Access가 열린 S3 Bucket이 있나?
- ALB 설정이 시간에 따라 어떻게 바뀌었나?

### Config Rules

- **AWS Managed Rules**: 75개 이상 기본 제공
- **Custom Rules**: Lambda 기반 커스텀 평가 로직
- 트리거: **설정 변경 시** 또는 **주기적 평가**
- ⚠️ Config Rules는 **예방(Deny) 기능 없음** — 규정 준수 평가만

**가격**: Free Tier 없음, 리전당 설정 항목 $0.003, 규칙 평가 $0.001

### Config 리소스 뷰

- 특정 리소스의 **규정 준수 변화 타임라인**
- 특정 리소스의 **설정 변경 타임라인**
- 특정 리소스 관련 **CloudTrail API 호출 타임라인**

### Config Rules — Remediations (자동 수정)

- 비준수 리소스를 **SSM Automation Document**로 자동 수정
- AWS Managed Automation Document 또는 Custom Document (Lambda 호출 가능)
- **Remediation Retry** 설정 가능 (자동 수정 후에도 비준수 시)

예시:

```
IAM Access Key 만료 (NON_COMPLIANT)
    → Auto Remediation: AWSConfigRemediation-RevokeUnusedIAMUserCredentials
```

### Config Rules — Notifications

- **EventBridge**: 비준수 리소스 발생 시 트리거 → 자동화
- **SNS**: 설정 변경 + 규정 준수 상태 알림 (SNS Filtering 또는 클라이언트 측 필터 활용)

---

## CloudWatch vs. CloudTrail vs. Config 비교

| 항목 | CloudWatch | CloudTrail | Config |
| --- | --- | --- | --- |
| **목적** | 성능 모니터링 + 알람 + 로그 | API 호출 감사 기록 | 설정 변경 기록 + 규정 준수 |
| **질문** | "CPU가 높은가?" | "누가 API를 호출했는가?" | "설정이 바뀌었는가? 규정에 맞는가?" |
| **범위** | 리전 (글로벌 집계 가능) | 글로벌 서비스 | 리전 (집계 가능) |

### ELB 관점에서 세 서비스 역할

| 서비스 | ELB에서의 역할 |
| --- | --- |
| **CloudWatch** | Incoming Connection 모니터링, 에러 코드 비율 시각화, 성능 대시보드 |
| **Config** | Security Group 규칙 추적, 설정 변경 이력, SSL 인증서 상시 부착 여부 규정 준수 |
| **CloudTrail** | API 호출 추적 (누가 LB 설정을 변경했는가?) |

---

## 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| EC2 기본 미제공 지표 | **RAM** (Unified Agent로 추가 수집 필요) |
| Custom Metric | RAM, 프로세스 등 **직접 푸시** 필요 |
| CW Logs → S3 Export | API: CreateExportTask, **최대 12시간 소요** (Near real-time 아님) |
| CW Logs 실시간 전송 | **Subscription Filter → KDS/KDF/Lambda** |
| Cross-Account Logs | **Cross-Account Subscription** 사용 |
| CW Alarm 테스트 | `set-alarm-state` CLI 명령으로 강제 ALARM |
| Composite Alarm | AND/OR 조건 조합, 알람 노이즈 감소 |
| EC2 Recovery | 동일 **Private/Public/Elastic IP, Metadata, Placement Group** 유지 |
| Container Insights + EKS | **컨테이너화된 CW Agent** 필요 |
| Lambda Insights 배포 | **Lambda Layer** 형태 |
| Contributor Insights | **Top-N** 기여자 파악 |
| EventBridge Target 권한 | Lambda/SNS/SQS: **Resource-based Policy** / Kinesis/ECS: **IAM Role** |
| EventBridge Replay | ✅ 아카이브된 이벤트 재생 가능 |
| CloudTrail 기본 보존 | **90일** |
| 90일 이상 CloudTrail 보존 | **S3 + Athena** |
| CloudTrail Data Events | **기본 비활성화** (고볼륨) |
| CloudTrail Insights | 비정상 활동 → CloudTrail 콘솔 + **S3 + EventBridge** |
| 리소스 삭제 원인 조사 | **CloudTrail 먼저** |
| Config Rules 기능 | 규정 준수 **평가만** (Deny/차단 불가) |
| Config Auto Remediation | **SSM Automation Document** |
| Config 서비스 범위 | **리전별** (리전 간 집계 가능) |