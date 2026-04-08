---
title: 🗄️ AWS RDS & Aurora (관계형 데이터베이스)
published: 2026-03-18
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🗄️ AWS RDS & Aurora (관계형 데이터베이스)

> AWS 관리형 관계형 DB 서비스 — 시험 고빈도 영역
> 
> 
> RDS는 다양한 엔진을 지원하고, Aurora는 AWS 전용 고성능 DB입니다.
> 

---
## 목차
1. [RDS 개요](#1-rds-개요)
2. [RDS vs EC2에서 DB 직접 운영](#2-rds-vs-ec2에서-db-직접-운영)
3. [RDS Storage Auto Scaling](#3-rds-storage-auto-scaling)
4. [RDS Read Replicas (읽기 복제본)](#4-rds-read-replicas-읽기-복제본)
5. [RDS Multi-AZ (재해 복구)](#5-rds-multi-az-재해-복구)
6. [RDS Custom](#6-rds-custom)
7. [Amazon Aurora](#7-amazon-aurora)
8. [Aurora 고급 기능](#8-aurora-고급-기능)
9. [RDS & Aurora 백업](#9-rds-aurora-백업)
10. [RDS & Aurora 보안](#10-rds-aurora-보안)
11. [핵심 비교 요약](#11-핵심-비교-요약)
12. [참고 자료](#-참고-자료)
---

## 1. RDS 개요

### 🔑 RDS (Relational Database Service)란?

- SQL 기반 관계형 데이터베이스를 위한 **AWS 관리형 서비스 (Managed DB Service)**
- 사용자가 DB 엔진만 선택하면 나머지는 AWS가 운영

### 지원 DB 엔진

| 엔진 | 특징 |
| --- | --- |
| PostgreSQL | 오픈소스, 기능 풍부 |
| MySQL | 가장 널리 사용되는 오픈소스 |
| MariaDB | MySQL 포크, 오픈소스 |
| Oracle | 상용 엔터프라이즈 DB |
| Microsoft SQL Server | 상용 엔터프라이즈 DB |
| IBM DB2 | 상용 엔터프라이즈 DB |
| **Aurora** | AWS 독자 개발 (PostgreSQL/MySQL 호환) |

---

## 2. RDS vs EC2에서 DB 직접 운영

| 항목 | RDS (관리형) | EC2에서 직접 운영 |
| --- | --- | --- |
| 프로비저닝·OS 패치 | **AWS 자동** | 직접 관리 |
| 연속 자동 백업 (PITR) | ✅ | 직접 구성 |
| 모니터링 대시보드 | ✅ | 직접 구성 |
| 읽기 복제본 (Read Replica) | ✅ (간편 설정) | 직접 구성 |
| Multi-AZ 구성 | ✅ (간편 설정) | 직접 구성 |
| 유지보수 창 (Maintenance Window) | ✅ | 직접 관리 |
| 스토리지 자동 확장 | ✅ | 직접 관리 |
| SSH 접속 | ❌ (관리 불가) | ✅ |
| 스토리지 | EBS 기반 | 선택 가능 |

---

## 3. RDS Storage Auto Scaling

- RDS 인스턴스의 스토리지가 부족해질 때 **자동으로 용량 확장**
- 수동으로 DB 스토리지를 확장할 필요 없음
- **Maximum Storage Threshold** (최대 스토리지 임계값) 설정 필수

### 자동 확장 조건 (모두 충족 시)

```
1️⃣ 여유 스토리지 < 할당 스토리지의 10%
        AND
2️⃣ 저용량 상태가 5분 이상 지속
        AND
3️⃣ 마지막 수정으로부터 6시간 이상 경과
```

> 💡 **Use Case**: 예측 불가한 워크로드(Unpredictable Workloads)가 있는 애플리케이션. 모든 RDS 엔진 지원.
> 

---

## 4. RDS Read Replicas (읽기 복제본)

### 📖 개념

- 읽기 요청(SELECT)을 메인 DB에서 분산하여 **읽기 성능 향상**
- **비동기 복제 (ASYNC Replication)** → **최종 일관성 (Eventually Consistent)**
    - 복제 전에 읽으면 약간 오래된(Stale) 데이터가 조회될 수 있음

```
[Primary DB] ──ASYNC 복제──→ [Read Replica 1]
             ──ASYNC 복제──→ [Read Replica 2]  ← 읽기 전용 쿼리
             ──ASYNC 복제──→ [Read Replica N]
```

---

### 📋 주요 특성

| 항목 | 내용 |
| --- | --- |
| **최대 복제본 수** | **15개** |
| **복제본 위치** | 동일 AZ, 교차 AZ, **교차 리전** 모두 가능 |
| **복제 방식** | **비동기 (ASYNC)** |
| **승격 가능 여부** | ✅ 독립 DB로 승격 가능 (Promote to own DB) |
| **사용 가능 작업** | **SELECT만** (INSERT, UPDATE, DELETE 불가) |

:::warning
⚠️ **애플리케이션 수정 필요**: Read Replica를 사용하려면 연결 문자열(Connection String)을 복제본 엔드포인트로 업데이트해야 함.
:::
---

### 💰 Read Replica 네트워크 비용

| 복제본 위치 | 데이터 전송 비용 |
| --- | --- |
| **동일 리전** 내 (AZ가 달라도) | **무료** |
| **교차 리전 (Cross Region)** | **요금 부과** (비동기 복제 데이터 전송) |

---

### 💡 Read Replica 활용 시나리오

```
[상황]
운영 DB: 정상 트래픽 처리 중
팀 요청: 분석용 리포팅 애플리케이션 추가

[잘못된 방법]  운영 DB에서 직접 분석 쿼리 실행 → 성능 저하
[올바른 방법]  Read Replica 생성 → 분석 쿼리는 복제본에서 실행
              → 운영 DB 영향 없음
```

---

## 5. RDS Multi-AZ (재해 복구)

### 🔄 개념

- **동기 복제 (SYNC Replication)** — 메인 DB의 모든 변경사항을 **즉시** Standby에 동기화
- **단일 DNS 이름** — 장애 시 애플리케이션 수정 없이 **자동 페일오버 (Automatic Failover)**

```
[Primary DB] ──SYNC 복제──→ [Standby DB]
     │                           │
     └────── 단일 DNS ───────────┘
             (자동 전환)
```

---

### 📋 주요 특성

| 항목 | 내용 |
| --- | --- |
| **복제 방식** | **동기 (SYNC)** |
| **목적** | **재해 복구 (Disaster Recovery, DR)** |
| **엔드포인트** | **단일 DNS 이름** (페일오버 시 앱 수정 불필요) |
| **페일오버 트리거** | AZ 장애, 네트워크 장애, 인스턴스 장애, 스토리지 장애 |
| **스케일링 목적** | ❌ (읽기 분산에 사용 불가, 스케일링 아님) |

:::tip
📌 **Read Replica는 Multi-AZ로 설정 가능** → Standby 복제본을 이중 보호
:::

---

### 🔧 Single-AZ → Multi-AZ 전환

- **Zero Downtime** — DB 중지 없이 전환 가능
- AWS 콘솔에서 "Modify(수정)" 클릭만으로 완료

```
내부 동작:
1. 현재 DB 스냅샷 생성
2. 새 AZ에 스냅샷에서 Standby DB 복원
3. 두 DB 간 동기화 확립 완료
```

---

### 📊 Read Replica vs Multi-AZ 비교

| 항목 | Read Replica | Multi-AZ |
| --- | --- | --- |
| 복제 방식 | **비동기 (ASYNC)** | **동기 (SYNC)** |
| 주요 목적 | **읽기 성능 향상** | **재해 복구 (DR)** |
| 쓰기 가능 여부 | ❌ | ❌ (Standby는 쓰기 불가) |
| 엔드포인트 | **복제본별 별도** | **단일 DNS** |
| AZ 수 | 1개 이상 | **2개 (Primary + Standby)** |
| 수 | 최대 15개 | 1개 (Standby) |

---

## 6. RDS Custom

- **Oracle, Microsoft SQL Server**에만 지원
- 표준 RDS: AWS가 DB와 OS를 완전 관리 (SSH 불가)
- **RDS Custom**: OS와 DB에 대한 **관리자 수준 접근 권한** 제공

| 기능 | RDS | RDS Custom |
| --- | --- | --- |
| 설정 자동화 | ✅ | ✅ (기본) |
| OS 접근 | ❌ | ✅ |
| DB 접근 (SSH/SSM) | ❌ | ✅ |
| 패치 직접 설치 | ❌ | ✅ |
| 네이티브 기능 활성화 | ❌ | ✅ |

:::note
💡 **커스터마이징 전**: 반드시 DB 스냅샷 생성 후 Automation Mode 비활성화
:::
---

## 7. Amazon Aurora

### ✨ 개요

- AWS가 독자 개발한 **독점 기술 (Proprietary Technology)** (오픈소스 아님)
- **PostgreSQL, MySQL과 호환** (드라이버 교체만으로 마이그레이션 가능)
- **AWS 클라우드에 최적화**: MySQL 대비 **5배**, PostgreSQL 대비 **3배** 성능 향상 주장
- 스토리지 자동 증가: **10GB 단위, 최대 128TB** (이전 256TB에서 업데이트)
- 최대 **15개 복제본**, 복제 지연 **10ms 미만 (Sub 10ms replica lag)**
- **즉각적인 페일오버 (Instantaneous Failover)** — HA Native
- RDS 대비 **약 20% 비용 증가** (하지만 효율성으로 상쇄)

---

### 🏗️ Aurora 고가용성 & 스토리지 아키텍처

```
[Aurora Writer (Master)]  →  [Read Replica] × 최대 15개
         │
         ▼
[분산 스토리지 클러스터]
├── AZ 1: 복사본 2개
├── AZ 2: 복사본 2개
└── AZ 3: 복사본 2개
     → 총 6개 복사본 (3 AZ × 2)
```

| 항목 | 내용 |
| --- | --- |
| **총 데이터 복사본** | **6개** (3개 AZ에 각 2개씩) |
| **쓰기 성공 기준** | 6개 중 **4개** 이상 성공 시 |
| **읽기 성공 기준** | 6개 중 **3개** 이상 가능 |
| **자가 복구** | Peer-to-Peer 복제로 자동 복구 (Self-healing) |
| **스토리지 구성** | 수백 개 볼륨에 스트라이핑(Striped) |
| **Master 페일오버** | **30초 이내** 자동 전환 |

---

### 🔗 Aurora 엔드포인트

```
[Writer Endpoint] ──→ [Master (쓰기)]          ← 항상 마스터를 가리킴
[Reader Endpoint] ──→ [Read Replicas (읽기)]   ← 복제본 간 로드 밸런싱
[Custom Endpoint] ──→ [특정 복제본 서브셋]      ← 분석 쿼리 등 특수 목적
```

| 엔드포인트 유형 | 설명 |
| --- | --- |
| **Writer Endpoint** | 마스터 자동 추적 (페일오버 시 자동 전환) |
| **Reader Endpoint** | 모든 Read Replica에 연결 분산 |
| **Custom Endpoint** | 특정 인스턴스 서브셋 지정 (고성능 인스턴스에만 분석 쿼리 전달) |

---

### ⚙️ Aurora DB 클러스터 주요 기능

| 기능 | 설명 |
| --- | --- |
| **Automatic Failover** (자동 페일오버) | Master 장애 시 30초 내 자동 전환 |
| **Backup & Recovery** | 자동 백업, PITR 지원 |
| **Isolation & Security** | VPC, KMS 암호화, IAM 통합 |
| **Industry Compliance** | 각종 규정 준수 |
| **Push-button Scaling** | 버튼 클릭으로 용량 조정 |
| **Automated Patching** | **Zero Downtime** 자동 패치 |
| **Advanced Monitoring** | 세밀한 모니터링 |
| **Backtrack** | 백업 없이 **특정 시점으로 데이터 복원** (Restore data at any point in time without using backups) |

---

## 8. Aurora 고급 기능

### 🌐 Aurora Serverless

- 실제 사용량 기반 **자동 DB 인스턴스화 및 오토스케일링**
- **비정기적(Infrequent), 간헐적(Intermittent), 예측 불가한(Unpredictable) 워크로드**에 적합
- 용량 계획 불필요
- 초 단위 과금 (Pay per second) — 경우에 따라 비용 효율적

---

### 🌍 Global Aurora

| 유형 | 설명 |
| --- | --- |
| **Aurora Cross Region Read Replicas** | 재해 복구(DR)용, 간단히 구성 가능 |
| **Aurora Global DB** ⭐ | 전용 글로벌 아키텍처 |

**Aurora Global DB 특성:**

- **1개 Primary Region** (읽기/쓰기)
- 최대 **5개 Secondary(보조) 리전** (읽기 전용)
- 보조 리전당 최대 **16개 Read Replica**
- 리전 간 복제 지연 **1초 미만 (< 1 second)**
- 보조 리전 승격 시 **RTO(Recovery Time Objective) < 1분**
- 지역별 지연 시간 감소에 효과적

---

### 🤖 Aurora Machine Learning

- Aurora에서 SQL 쿼리로 **ML 기반 예측** 수행
- Aurora ↔ AWS ML 서비스 간 최적화된 보안 통합

| 지원 서비스 | 용도 |
| --- | --- |
| **Amazon SageMaker** | ML 모델 (범용) |
| **Amazon Comprehend** | 감성 분석 (Sentiment Analysis) |

**Use Case**: 사기 탐지(Fraud Detection), 광고 타겟팅, 감성 분석, 상품 추천

---

### 🔄 Babelfish for Aurora PostgreSQL

- Aurora PostgreSQL이 **Microsoft SQL Server의 T-SQL 명령을 이해**하도록 지원
- MS SQL Server 기반 애플리케이션을 **코드 수정 거의 없이** Aurora PostgreSQL로 마이그레이션 가능

---

## 9. RDS & Aurora 백업

### 🔄 자동 백업 (Automated Backups)

| 항목 | RDS | Aurora |
| --- | --- | --- |
| 백업 주기 | 매일 전체 백업 (지정 백업 창) | 지속 백업 |
| 트랜잭션 로그 | **5분마다** 백업 | 지속 |
| PITR (Point-in-Time Recovery) | 최근 5분 전까지 복원 가능 | 동일 |
| 보존 기간 | **1~35일** (0으로 설정 시 비활성화) | **1~35일** (**비활성화 불가**) |

---

### 📸 수동 스냅샷 (Manual DB Snapshots)

- 사용자가 직접 트리거
- **보존 기간 무제한** (자동 백업은 최대 35일)
- 오래 DB를 중지할 경우: 스냅샷 저장 후 삭제 → 나중에 복원 (스토리지 비용 절감)

---

### 🔃 복원 옵션 (Restore Options)

| 방법 | 설명 |
| --- | --- |
| 백업/스냅샷 복원 | **새 DB 인스턴스 생성** (기존 DB 덮어쓰기 불가) |
| **MySQL RDS ← S3** | 온프레미스 DB 백업을 S3에 저장 후 새 RDS에 복원 |
| **MySQL Aurora ← S3** | **Percona XtraBackup**으로 백업 → S3 → 새 Aurora 클러스터 복원 |

---

### 🔁 Aurora Database Cloning (클로닝)

- 기존 Aurora DB 클러스터에서 **새 클러스터 빠르게 생성**
- 스냅샷 & 복원보다 **빠름**
- **Copy-on-Write 프로토콜** 사용:

```
초기 상태: 원본과 클론이 동일한 스토리지 볼륨 공유 (복사 없음 → 빠름)
         │
         ▼
클론에 데이터 변경 발생 시: 변경된 데이터만 별도 스토리지에 분리 저장
```

**Use Case**: **프로덕션 DB에 영향 없이 스테이징 DB 즉시 생성** (Useful to create a staging database from a production database without impacting the production database)

---

## 10. RDS & Aurora 보안

### 🔐 보안 레이어

| 보안 항목 | 내용 |
| --- | --- |
| **저장 데이터 암호화 (At-rest Encryption)** | **AWS KMS** 사용, 생성 시 정의 필수 |
| **전송 데이터 암호화 (In-flight Encryption)** | **TLS 기본 활성화** (AWS TLS 루트 인증서 사용) |
| **IAM 인증** | 사용자명/비밀번호 대신 **IAM 역할**로 DB 접속 |
| **보안 그룹 (Security Groups)** | RDS/Aurora에 대한 네트워크 접근 제어 |
| **SSH 접속** | ❌ 불가 (RDS Custom 제외) |
| **감사 로그 (Audit Logs)** | 활성화 시 **CloudWatch Logs**로 전송 (장기 보존) |

---

### 🔑 암호화 관련 핵심 규칙

```
마스터 암호화 ← Read Replica 암호화 여부 결정
  │
  ├── 마스터 암호화 → 복제본도 암호화 가능
  └── 마스터 미암호화 → 복제본 암호화 불가 ❌

미암호화 DB → 암호화 변환 방법:
  1. DB 스냅샷 생성
  2. 스냅샷 복사 시 암호화 활성화
  3. 암호화된 스냅샷으로 새 DB 복원
```

---

## 11. 핵심 비교 요약

### RDS vs Aurora

| 항목 | RDS | Aurora |
| --- | --- | --- |
| 엔진 | 여러 엔진 선택 | PostgreSQL/MySQL 호환 |
| 성능 | 표준 | **MySQL 5배, PostgreSQL 3배** |
| 스토리지 | EBS (수동 확장) | **자동 확장 (최대 128TB)** |
| Read Replica 수 | 최대 15개 | **최대 15개** |
| 복제 지연 | 일반적 | **Sub 10ms** |
| 페일오버 | 수분 | **30초 이내** |
| 비용 | 기준 | **20% 높음** |
| 데이터 복사본 | 1~2개 | **6개 (3 AZ × 2)** |
| SSH 접속 | ❌ | ❌ |
| Backtrack | ❌ | ✅ |
| Serverless | ❌ | ✅ |

---

### 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| Read Replica 복제 | **비동기 (ASYNC)** → 최종 일관성 |
| Multi-AZ 복제 | **동기 (SYNC)** → 재해 복구 목적 |
| Multi-AZ 목적 | **DR (재해 복구)**, 스케일링 아님 |
| Read Replica 비용 | 동일 리전은 무료, 교차 리전은 유료 |
| Read Replica 용도 | **SELECT만** (쓰기 불가) |
| Single→Multi-AZ | **Zero Downtime** 전환 가능 |
| Aurora 복사본 수 | **6개** (3 AZ, 각 2개) |
| Aurora 쓰기 정족수 | 6개 중 **4개** |
| Aurora 읽기 정족수 | 6개 중 **3개** |
| Aurora 페일오버 | **30초 이내** |
| Aurora Global 복제 지연 | **1초 미만** |
| Aurora Global RTO | 리전 승격 시 **1분 미만** |
| Aurora Backtrack | 백업 없이 특정 시점 복원 |
| Aurora Cloning | **Copy-on-Write**, 스테이징 DB 생성에 활용 |
| RDS 자동 백업 보존 | **최대 35일** |
| Aurora 자동 백업 | **비활성화 불가** |
| MySQL Aurora S3 복원 | **Percona XtraBackup** 필요 |
| RDS 암호화 키 | **AWS KMS** |
| 암호화 DB 복제본 | 마스터 미암호화 시 복제본 암호화 불가 |
| RDS Custom 지원 엔진 | **Oracle, MS SQL Server만** |

---

## 📚 참고 자료

- [Amazon RDS 공식 문서](https://docs.aws.amazon.com/rds/latest/userguide/Welcome.html)
- [Amazon Aurora 공식 문서](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [RDS 백업 및 복원](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html)
- [Aurora Database Cloning](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Clone.html)