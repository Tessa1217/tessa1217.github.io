---
title: 🚨 AWS Disaster Recovery — Whitepaper Summary
published: 2026-03-23
tags: [AWS, Cloud, Certificates]
category: AWS
draft: true
---
# 🚨 AWS Disaster Recovery — Whitepaper Summary

> 출처: [Disaster Recovery of Workloads on AWS: Recovery in the Cloud](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
> 
> 
> Publication: February 12, 2021 | AWS Well-Architected Framework
> 
---
## 목차
1. [핵심 개념 정의](#1-핵심-개념-정의)
2. [Business Continuity Plan (BCP)](#2-business-continuity-plan-bcp)
3. [Data Plane vs. Control Plane](#3-data-plane-vs-control-plane)
4. [DR 전략 4가지](#4-dr-전략-4가지)
5. [Active/Passive vs. Active/Active](#5-activepassive-vs-activeactive)
6. [Failover 트래픽 라우팅 서비스](#6-failover-트래픽-라우팅-서비스)
7. [AWS 핵심 DR 서비스](#7-aws-핵심-dr-서비스)
8. [DR 테스팅 (Testing Disaster Recovery)](#8-dr-테스팅-testing-disaster-recovery)
9. [전략 선택 가이드](#9-전략-선택-가이드)
10. [비용 최적화 관점](#10-비용-최적화-관점)
11. [📌 시험 자주 출제 포인트 총정리](#-시험-자주-출제-포인트-총정리)
---

## 1. 핵심 개념 정의

### Disaster (재해)란?

워크로드 또는 시스템이 **Primary 배포 위치에서 비즈니스 목표를 달성하지 못하도록 방해하는 이벤트**.

자연재해, 기술 장애, 인위적 실수 모두 포함.

---

### RTO & RPO

| 지표 | 정의 |
| --- | --- |
| **RTO** (Recovery Time Objective) | 서비스 중단 후 **복구까지 허용 가능한 최대 시간** ("얼마나 빨리 돌아올 수 있는가?") |
| **RPO** (Recovery Point Objective) | **마지막 데이터 복구 시점부터 허용 가능한 최대 시간** ("얼마만큼의 데이터 손실을 허용하는가?") |

```
재해 발생
    │
    ├─── RPO ───→ (마지막 백업 시점)
    │                   데이터 손실 범위
    │
    └─── RTO ───→ (서비스 복구 시점)
                    서비스 다운타임 범위
```

> 📌 **RTO, RPO가 낮을수록 비용과 복잡도 증가** — 비즈니스 요구사항에 맞는 수준 선택 필수
> 

---

### Resiliency vs. DR vs. High Availability

| 개념 | 정의 | 범위 |
| --- | --- | --- |
| **Resiliency** | 인프라/서비스 장애 복구 능력 | 광범위 |
| **High Availability** | **단일 구성요소 장애** 시에도 서비스 지속 | AZ 레벨 단일 장애 |
| **Disaster Recovery** | **일회성 재난 이벤트** 발생 시 복구 | 데이터 센터/리전 레벨 |

> DR은 HA와 별개이며, HA가 완벽해도 DR 전략은 별도로 필요함.
> 

---

## 2. Business Continuity Plan (BCP)

- DR 전략은 **BCP(비즈니스 연속성 계획)의 하위 집합**이어야 함 (독립적인 문서 아님)
- 비즈니스 영향 분석 (Business Impact Analysis)으로 워크로드 중단의 비즈니스 영향 정량화
- DR 목표는 비즈니스 요구사항, 우선순위, 맥락에 기반해야 함

**예시**: 지진으로 물류가 차단되면 eCommerce 앱 DR이 완벽해도 BCP가 물류를 해결 못하면 비즈니스 목표 달성 불가.

---

## 3. Data Plane vs. Control Plane

DR 전략 설계 시 핵심 구분:

| 구분 | 역할 | 가용성 목표 | Failover 시 사용 |
| --- | --- | --- | --- |
| **Data Plane** | **실시간 서비스 제공** | **높음** | ✅ 권장 |
| **Control Plane** | 환경 설정/관리 | 낮음 | ❌ 지양 |

:::important
Failover 작업에는 **Data Plane 작업만 사용** — Control Plane 작업(예: AWS Backup 복원)은 재해 시 접근 불가능할 수 있음.
:::
:::tip
백업에서 데이터 복원은 Control Plane 작업 → **정기적 주기 복원(Scheduled Periodic Restore)을 미리 설정**하여 복원된 데이터스토어를 항상 보유해야 함.
:::
---

## 4. DR 전략 4가지

### 전략 비교 한눈에 보기

| 전략 | RPO | RTO | 비용 | 복잡도 | 트래픽 유형 |
| --- | --- | --- | --- | --- | --- |
| **Backup & Restore** | 시간 단위 | 시간 단위 | 최저 💲 | 최저 | Active/Passive |
| **Pilot Light** | 분 단위 | 수십 분 | 중간 💲💲 | 중간 | Active/Passive |
| **Warm Standby** | 초 단위 | 분 단위 | 높음 💲💲💲 | 높음 | Active/Passive |
| **Multi-Site Active/Active** | Near-zero | Near-zero | 최고 💲💲💲💲 | 최고 | Active/Active |

```
비용/복잡도 (낮음 → 높음):
Backup & Restore → Pilot Light → Warm Standby → Multi-Site Active/Active

RTO/RPO (높음 → 낮음, 즉 복구 시간 길어짐):
Multi-Site Active/Active → Warm Standby → Pilot Light → Backup & Restore
```

---

### 전략 1: Backup & Restore (백업 및 복원)

**개념**: 데이터를 정기적으로 백업 → 재해 발생 시 복원

```
[Primary Region]                    [Recovery Region]
EC2 / RDS / EBS 등
    │
    ├── 스냅샷 생성 (같은 리전)
    └── 스냅샷 복사 ─────────────────→ S3 / Glacier 저장
                                         (재해 시 복원)
                                         인프라도 IaC로 재배포
```

**특성:**

- 가장 단순하고 가장 저렴한 전략
- **Recovery 시 인프라 배포 + 코드 배포 + 데이터 복원 모두 필요** → 높은 RTO
- PITR(Point-in-Time Recovery) 활용 시 RPO를 약 **5분**까지 낮출 수 있음
- 모든 워크로드에 기본 적용 (다른 전략과 함께 사용)

**AWS 서비스:**

- **AWS Backup**: EC2, EBS, RDS, DynamoDB, EFS, FSx 등 중앙 집중 백업 관리
- **S3 + S3 Glacier**: 백업 저장 (Glacier: 수시간 복원)
- **CloudFormation / CDK**: 인프라 코드(IaC)로 복원 시간 단축
- **Amazon EventBridge + Lambda**: 감지 자동화 → RTO 단축

**적합한 경우:**

- 비즈니스 크리티컬하지 않은 워크로드
- 단일 AZ/데이터 센터 장애 수준의 재해만 고려할 때
- 비용 절감이 최우선일 때 (`cost-effective`)

---

### 전략 2: Pilot Light (파일럿 라이트)

**개념**: 데이터는 항상 복제 유지 → 핵심 인프라만 Recovery Region에 배포 → **컴퓨팅(EC2 등)은 꺼진(Shut-off) 상태**

```
[Primary Region]               [Recovery Region]
앱 서버 (Active)               앱 서버 = 0개 (배포 안 됨)
RDS Primary ─── Async 복제 ──→ RDS Replica (항상 실행)
ELB / ASG                      ELB / ASG (배포됨, 트래픽 없음)
    │                               │
    │       재해 발생               │
    └───────────────────────────────→
                    AMI에서 EC2 인스턴스 배포 + 스케일 아웃
                    DNS/Global Accelerator로 트래픽 전환
```

**특성:**

- 데이터는 항상 라이브 복제 (DB는 항상 켜짐)
- **컴퓨팅 리소스는 재해 시 비로소 배포** → 배포 시간 포함 = RTO 수십 분
- Warm Standby 대비 비용 절감 (컴퓨팅 비용 없음)
- **Pilot Light vs Warm Standby 핵심 차이**: Pilot Light는 컴퓨팅 없음 → 재해 시 "Turn on"(배포) 필요

**AWS Elastic Disaster Recovery (DRS):**

- **Pilot Light 전략**을 기반으로 구현
- 에이전트 기반 **블록 레벨 연속 복제** (Block-level replication)
- On-premises/EC2 → AWS로 복제 가능 (단, RDS 제외 — EC2 기반 DB만)
- **RPO: 초 단위 / RTO: 분 단위** 달성 가능
- Staging Area에 복제본 유지 (저비용 스토리지 + 최소 컴퓨팅)
- 비파괴적(Non-disruptive) 테스트 드릴 지원
- Failover/Failback 모두 지원

**AWS 서비스:**

- Aurora Global Database (복제)
- Amazon S3 + CloudFormation (백업 + IaC)
- Route 53 / Global Accelerator (트래픽 전환)
- AWS Elastic Disaster Recovery

---

### 전략 3: Warm Standby (웜 스탠바이)

**개념**: Recovery Region에 **축소된(Scaled-down) 완전한 기능의 환경** 항상 실행 → 재해 시 **스케일 업(Scale up)만** 하면 됨

```
[Primary Region]               [Recovery Region]
앱 서버 × N개 (Full)          앱 서버 × 1개 (Minimum) ← 항상 실행
RDS Primary ─── 복제 ───────→ RDS Replica (항상 실행)
ELB / ASG (Full)               ELB / ASG (Minimum)
    │                               │
    │       재해 발생               │
    └───────────────────────────────→
                    Scale Out만 하면 됨 (배포 불필요)
                    DNS/Global Accelerator로 트래픽 전환
```

**특성:**

- 항상 최소한의 프로덕션 환경이 실행 중 → **즉시 일부 트래픽 처리 가능**
- 재해 시 스케일 업만 필요 → Pilot Light보다 **낮은 RTO**
- Pilot Light와 비교: **코드와 인프라가 이미 실행 중**인 것이 차이
- Full Scale = "Hot Standby"라고도 불림

**Pilot Light vs Warm Standby 차이 요약:**

| 항목 | Pilot Light | Warm Standby |
| --- | --- | --- |
| 컴퓨팅 상태 | 배포 안 됨 (꺼짐) | 최소 규모로 실행 중 |
| 재해 시 행동 | 배포 + 스케일 아웃 | 스케일 아웃만 |
| RTO | 더 높음 | **더 낮음** |
| 비용 | 더 낮음 | 더 높음 |
| 즉시 트래픽 처리 | ❌ | ✅ (축소된 용량) |

**AWS 서비스:**

- Aurora Global Database / RDS Multi-Region Replica
- EC2 Auto Scaling (스케일 업)
- Route 53 / Global Accelerator (Failover 라우팅)

---

### 전략 4: Multi-Site Active/Active (멀티 사이트 액티브/액티브)

**개념**: 2개 이상의 AWS Region에서 **동시에 트래픽 처리** → 한 리전 장애 시 트래픽만 재라우팅

```
[Region A: Active]             [Region B: Active]
앱 서버 (Full)                 앱 서버 (Full)
RDS ←──── 양방향 복제 ─────→  RDS (Aurora Global)
    ↑                              ↑
    └──── Route 53 / Global Accelerator ────┘
              (두 리전 모두 트래픽 수신)

재해 발생 시: 트래픽을 장애 리전에서 빼는 것만으로 복구
```

**특성:**

- **RPO: Near-zero / RTO: Near-zero 또는 Zero**
- 비동기 데이터 복제로 Near-zero RPO 달성
- 가장 높은 비용과 복잡도
- **쓰기 충돌(Write Conflict)** 관리 필요 (두 리전에 동시 Write 가능)
- 데이터 복제는 일부 재해로부터만 보호 → **PITR(Point-in-Time Recovery)도 함께 필요**

**트래픽 라우팅:**

- **Route 53**: 지역별 라우팅 정책 (Geoproximity, Latency 등), 비율 기반 가중치
- **Global Accelerator**: AWS Edge Network 활용 → 낮은 지연 시간, DNS 캐시 문제 없음, Traffic Dial로 리전별 비율 설정

---

## 5. Active/Passive vs. Active/Active

| 항목 | Active/Passive | Active/Active |
| --- | --- | --- |
| **해당 전략** | Backup & Restore, Pilot Light, Warm Standby | Multi-Site Active/Active |
| **동작** | Primary에서 트래픽 처리, DR은 Standby | 모든 리전에서 트래픽 처리 |
| **Failover 방법** | DNS/Global Accelerator로 트래픽 전환 | 장애 리전에서 트래픽 빼기만 |
| **데이터 충돌** | 없음 | 쓰기 충돌 관리 필요 |

---

## 6. Failover 트래픽 라우팅 서비스

| 서비스 | 특징 | Active/Active 지원 | DNS 캐시 문제 |
| --- | --- | --- | --- |
| **Amazon Route 53** | DNS 기반, 다양한 라우팅 정책 지원 | ✅ | 있음 |
| **AWS Global Accelerator** | Anycast IP, AWS Edge 네트워크 직접 진입 | ✅ | ❌ 없음 |
| **Amazon CloudFront** | Origin Failover (요청 단위), 이후 요청은 Primary로 | 제한적 | - |

:::tip
**Global Accelerator**: DNS 캐시 문제 없음, 낮은 지연 시간 → Active/Active, Pilot Light, Warm Standby 모두 적합
:::
---

## 7. AWS 핵심 DR 서비스

| 서비스 | 역할 | 주요 전략 |
| --- | --- | --- |
| **AWS Backup** | EC2, EBS, RDS, DynamoDB 등 중앙 집중 백업 | Backup & Restore |
| **AWS Elastic Disaster Recovery (DRS)** | 블록 레벨 연속 복제, Failover/Failback | Pilot Light |
| **Aurora Global Database** | 리전 간 복제 < 1초, 최대 5개 Secondary | Pilot Light, Warm Standby |
| **RDS Read Replica (Cross-Region)** | 비동기 복제 | Pilot Light, Warm Standby |
| **S3 Cross-Region Replication** | 객체 복제 | 모든 전략 |
| **CloudFormation / CDK** | IaC로 DR 리전 인프라 신속 배포 | Backup & Restore |
| **AWS Resilience Hub** | RTO/RPO 목표 달성 여부 지속 검증 및 추적 | 모든 전략 |
| **Route 53 / Global Accelerator** | Failover 트래픽 라우팅 | 모든 전략 |
| **Amazon EventBridge + Lambda** | 재해 감지 자동화, RTO 단축 | 모든 전략 |

---

## 8. DR 테스팅 (Testing Disaster Recovery)
:::warning
⚠️ **DR 전략은 정기적으로 테스트하지 않으면 실제 재해 시 작동 보장 불가**
:::

### 핵심 원칙

- **DR 구현을 검증**하고 **DR 리전으로의 Failover를 정기 테스트**하여 RTO/RPO 달성 여부 확인
- **"거의 실행되지 않는 복구 경로" 패턴 피하기**: 드물게 테스트된 경로는 실제 장애 시 실패 위험

**테스트해야 하는 이유 (실제 사례):**

```
가정: Secondary DB가 Read-only 쿼리를 담당
     Primary 장애 시 Secondary로 Write 가능하다고 생각

현실: 오랫동안 Failover 테스트를 안 했다면
     → Secondary 용량이 부족하거나
     → 서비스 쿼터가 충족 안 될 수 있음
```

### DR 테스트 방법

| 방법 | 설명 |
| --- | --- |
| **Backup 테스트** | 백업 복원 정기 실행 (단순히 백업 생성으로 충분 아님) |
| **Failover 드릴** | 실제 DR 리전으로 Failover 시연 |
| **Chaos Engineering** | 의도적 장애 주입으로 복구 능력 검증 |
| **DR 드릴** | Isolated Subnet에서 실행 → 프로덕션 미간섭 |

:::note
💡 **AWS Resilience Hub**: RTO/RPO 목표 달성 여부를 지속적으로 검증하는 서비스
:::

---

## 9. 전략 선택 가이드

```
단순한 데이터 센터 장애 수준 + 비용 최소화
                    ↓
            Backup & Restore

데이터 센터 장애 수준 + RPO/RTO 수십 분 + 비용 절감
                    ↓
               Pilot Light
               (또는 AWS Elastic Disaster Recovery)

리전 레벨 장애 + RPO/RTO 분 단위 + 비즈니스 크리티컬
                    ↓
             Warm Standby

리전 레벨 장애 + Near-zero RPO/RTO + 미션 크리티컬
                    ↓
         Multi-Site Active/Active
```

**규제 요건:** 데이터 레지던시 요구사항이 있어 단일 리전만 사용 가능한 경우 → **AZ를 Region 대신 DR 사이트로 활용** 가능

---

## 10. 비용 최적화 관점

- AWS는 물리적 Backup Data Center의 **고정 자본 비용(CapEx)** → **실제 사용량 기반 운영 비용(OpEx)**으로 전환
- 온프레미스 DR은 항상 2번째 데이터 센터 풀 운영 비용 → AWS에서는 Pilot Light/Warm Standby로 최소 비용만 유지
- **Glacier / Glacier Deep Archive**: 백업 저장 비용 대폭 절감 (아카이브 데이터)
- 필요 이상으로 엄격한 전략 선택 금지 → 불필요한 비용 발생

---

## 📌 시험 자주 출제 포인트 총정리

| 포인트 | 내용 |
| --- | --- |
| RTO 정의 | 서비스 중단 후 **복구까지 허용 최대 시간** |
| RPO 정의 | **마지막 데이터 복구 시점 이후 허용 최대 시간** (데이터 손실 허용 범위) |
| RTO/RPO 낮을수록 | **비용과 복잡도 증가** |
| Data Plane vs Control Plane | Failover에는 **Data Plane 작업만** 사용 권장 |
| Backup & Restore RTO/RPO | **가장 높음** (시간 단위) / 비용 최저 |
| Pilot Light 특징 | 데이터 라이브 복제, **컴퓨팅 배포 없음** |
| Pilot Light vs Warm Standby | Pilot: 재해 시 "Turn on"(배포) / Warm: 재해 시 "Scale up"만 |
| Warm Standby 특징 | **최소 규모 환경 항상 실행**, 즉시 일부 트래픽 처리 가능 |
| Multi-Site Active/Active | **Near-zero RPO/RTO**, 모든 리전 Active, 쓰기 충돌 관리 필요 |
| Active/Passive 해당 전략 | Backup & Restore, Pilot Light, Warm Standby |
| Active/Active 해당 전략 | **Multi-Site Active/Active** |
| AWS Elastic DRS 전략 | **Pilot Light** 기반, 블록 레벨 연속 복제 |
| AWS Elastic DRS 대상 | On-premises 또는 **EC2 기반 앱/DB** (RDS 제외) |
| Global Accelerator 장점 | DNS 캐시 문제 없음, Edge 네트워크 활용 |
| DR 테스팅 필수 이유 | 드물게 실행되는 복구 경로는 **실제 장애 시 실패 가능** |
| Backup에서 자동 복원 | **AWS SDK로 AWS Backup API 호출** (자동 복원 기본 미지원) |
| DR 목표 지속 추적 | **AWS Resilience Hub** |
| 단일 리전 규제 환경 DR | **AZ를 Recovery Site로 활용** |