---
title: 📘 AWS DB 백서 (Whitepaper Summary)
published: 2026-03-27
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 📘 AWS DB 백서 (Whitepaper Summary)

> **출처:** Building Modern Applications Using AWS Databases (AWS 공식 백서, 2024)
**목적:** SAA-C03 시험 대비 핵심 개념 정리
**최종 업데이트:** 2025
> 
---
## 목차
1. [Purpose-Built Database 선택 전략](#1-purpose-built-database-선택-전략)
2. [스케일링 전략](#2-스케일링-전략)
3. [In-Memory: ElastiCache vs MemoryDB](#3-in-memory-elasticache-vs-memorydb)
4. [고가용성 & 운영 안정성](#4-고가용성--운영-안정성)
5. [Multi-Region 배포 전략](#5-multi-region-배포-전략)
6. [Zero-ETL](#6-zero-etl)
7. [Vector Search & Generative AI](#7-vector-search--generative-ai)
8. [데이터 마이그레이션](#8-데이터-마이그레이션)
9. [DB 서비스 선택 매트릭스](#9-db-서비스-선택-매트릭스)
10. [🎯 시험 단골 패턴 요약](#-시험-단골-패턴-요약)
---

## 1. Purpose-Built Database 선택 전략

### 핵심 원칙

> **단일 관계형 DB로는 현대 애플리케이션의 모든 요구사항을 충족할 수 없다.**
> 
- 소셜, IoT, 모바일, 글로벌 서비스는 각기 다른 데이터 모델이 필요
- 마이크로서비스 아키텍처에서 각 팀은 **워크로드에 최적화된 DB를 독립적으로 선택**해야 함
- AWS는 15종 이상의 Purpose-built DB 제공

### AWS DB 포트폴리오

| 유형 | 서비스 |
| --- | --- |
| **Relational** | Amazon Aurora, Amazon RDS |
| **Key-Value** | Amazon DynamoDB |
| **Document** | Amazon DocumentDB |
| **In-Memory** | Amazon ElastiCache, Amazon MemoryDB |
| **Graph** | Amazon Neptune |
| **Wide-Column** | Amazon Keyspaces |
| **Time-Series** | Amazon Timestream |
| **Search** | Amazon OpenSearch Service |

### 🛒 E-Commerce 아키텍처 예시 (시험 단골 출제)

| 기능 | 선택 DB | 이유 |
| --- | --- | --- |
| 상품 검색 | Amazon OpenSearch | 인덱스 최적화 검색 |
| 장바구니 / 결제 | Amazon Aurora | 트랜잭션 무결성 필요 |
| 고객 리뷰 (별점) | Amazon DynamoDB | 단순 Key-Value 구조 |
| 상품 추천 | Amazon Neptune | 그래프 기반 관계 탐색 |

### Managed vs Self-Managed

**AWS 완전 관리형이 처리하는 것:**

- 프로비저닝, HA 구성, 자동 패치/업그레이드
- 자동 백업, 장애조치(Failover)
- Self-healing 스토리지, 자동 스케일링

**개발자/DBA가 집중할 수 있는 것:**

- 스키마 설계, 쿼리 최적화, 접근 제어, 신규 기능 개발

:::tip
"관리 부담 최소화", "운영 오버헤드 감소" → **완전 관리형(Managed) 서비스 선택**이 정답 패턴
:::
---

## 2. 스케일링 전략

### 2-1. Serverless 자동 스케일링

| 구현 방식 | 지원 서비스 |
| --- | --- |
| **Vertical Auto Scaling** | Aurora, Neptune, Timestream for LiveAnalytics |
| **완전 서버리스** (compute + storage 모두 자동) | Aurora DSQL, DynamoDB, ElastiCache, Keyspaces |

:::tip
피크 기준 프로비저닝 대비 **최대 90% 비용 절감**, 사용량 기반 과금
→ "가변 트래픽 + 비용 최적화 + 관리 Zero" = **Serverless 계열**
:::
---

### 2-2. Vertical vs Horizontal Scaling

```
Vertical Scaling   → 인스턴스 크기 증가 (CPU / RAM / Network)
                     운영 중단 없이 실시간 동적 조정 가능

Horizontal Scaling → 노드 추가로 분산 처리
                     방법: 읽기 레플리카 추가 / 파티셔닝 / 샤딩
```

:::tip
- **읽기 성능 개선** → Read Replica 추가 (Horizontal)
- **단일 인스턴스 성능 향상** → Vertical Scaling
:::
---

### 2-3. Data Partitioning

- 테이블을 컬럼 값 기준으로 **서브셋(파티션)** 으로 분할
- 파티션 방식: **Range / List / Hash**
- 샤딩과의 차이: 파티션은 **같은 DB 인스턴스 내** 분할 (다른 노드 불필요)
- 앱에 투명하게(Transparent) 동작 — 코드 변경 없음
- **Aurora DSQL**이 자동 지원

:::tip
파티셔닝 ≠ 샤딩. 파티셔닝은 단일 인스턴스 내 분할
:::
---

### 2-4. Sharding (Aurora PostgreSQL Limitless)

- 데이터를 **여러 독립 노드**에 분산 (Shared-Nothing 아키텍처)
- 각 샤드는 독립적인 데이터 서브셋 처리
- **Aurora PostgreSQL Limitless Database:**
    - 완전 관리형 샤딩 — 수동 라우팅 로직 불필요
    - 초당 수백만 트랜잭션 커밋 지원
    - 페타바이트급 확장
    - 샤드 추가/제거 시 **자동 리밸런싱**

:::tip
초대규모 OLTP 쓰기 확장 → **Aurora PostgreSQL Limitless**
직접 샤딩 구현 없이 AWS가 라우팅 관리 = "개발 오버헤드 최소화"
:::
---

## 3. In-Memory: ElastiCache vs MemoryDB

### 비교표

| 항목 | ElastiCache | MemoryDB |
| --- | --- | --- |
| **분류** | 캐시 (Non-durable) | 완전 내구성 In-Memory DB |
| **엔진** | Valkey, Memcached, Redis OSS | Valkey, Redis OSS |
| **읽기 레이턴시** | 마이크로초 | 마이크로초 |
| **쓰기 레이턴시** | 마이크로초 | 수 밀리초 (Multi-AZ 동기 쓰기) |
| **데이터 내구성** | 소량 손실 가능 | Zero 손실 |
| **HA** | 99.99% | 99.999% |
| **멀티리전** | - | Active-Active 지원 |

### ElastiCache 상세

- 단일 노드: **초당 100만 req 처리**
- 클러스터: **초당 5억 req 처리**
- Primary 장애 시 복제본 자동 승격 → 미복제 데이터만 손실
- **Valkey** = Redis OSS 대체 (Redis가 2024년 3월 BSD 3-Clause 라이선스 변경 후 Linux Foundation 주관)

### MemoryDB 상세

- 쓰기 시 **Multi-AZ 트랜잭션 로그에 동기 기록** 후 응답
- 읽기는 마이크로초 유지
- 가용성 **99.999%**

### 선택 기준

```
데이터 손실 허용 가능 + 원본 DB 별도 존재 → ElastiCache (캐시 레이어)
데이터 손실 절대 불허 + DB 자체가 In-Memory  → MemoryDB
```

:::tip
- **"캐시 vs 내구성 인메모리 DB" 선택 문제** 자주 출제
- 분기점 = **데이터 손실 허용 여부**
- 마이크로서비스에서 Network Latency 시 상쇄용 캐시 사용 = **Best Practice**
:::

---

## 4. 고가용성 & 운영 안정성

### 4-1. Multi-AZ 구성 패턴

**Aurora Multi-AZ:**

- 3개 AZ에 데이터 자동 복제, 비용은 1개분만 과금
- Writer 장애 시 → Reader로 **자동 Failover**
- Read Replica를 다른 AZ에 분산 배치

**RDS Multi-AZ:**

- Standby에 **동기 복제** → Failover 자동
- Standby는 읽기 불가 (Single Standby 기준)
- Multi-AZ with 2 Readable Standbys: 읽기 가능 + 1초 미만 다운타임

**DynamoDB Multi-AZ:**

- 3개 AZ에 자동 파티셔닝·복제·동기화
- **Failover 불필요** — Multi-active 구성

---

### 4-2. 가용성 SLA 비교

| 가용성 | 서비스 |
| --- | --- |
| **99.999%** (5 nines) | Aurora DSQL, DynamoDB, Keyspaces, MemoryDB |
| **99.99%** (4 nines) | RDS, ElastiCache, DocumentDB, Timestream |

:::tip
미션 크리티컬 + 최고 가용성 요구 → **Aurora DSQL 또는 DynamoDB**
:::
---

### 4-3. Blue/Green Deployments

- **대상:** Amazon RDS 및 Aurora 모두 지원
- **원리:**
    - Blue = 현재 프로덕션
    - Green = AWS가 자동 생성하는 완전 관리형 스테이징
- **장점:**
    - 전환 소요 시간: **1분 미만**
    - 데이터 손실: **Zero**
    - 메이저 버전 업그레이드, 스키마 변경 시 다운타임 없음
- **RDS Multi-AZ (2 Readable Standbys) + RDS Proxy:**
    - 마이너 버전 업그레이드 다운타임: **1초 미만**

:::tip
- "다운타임 없는 메이저 업그레이드" → **Blue/Green Deployments**
- "1초 미만 다운타임" → **RDS Proxy** 함께 언급
:::
---

## 5. Multi-Region 배포 전략

### 3가지 전략 비교

| 전략 | 쓰기 가능 리전 | 일관성 | 주요 서비스 |
| --- | --- | --- | --- |
| **Read Replica** | Primary 1개 | 비동기(Eventually) | RDS, Aurora Global DB, DocumentDB, Neptune |
| **Active-Active + Eventual** | 모든 리전 | Eventual (보통 1초 이내) | DynamoDB Global Tables, MemoryDB Multi-Region, Keyspaces |
| **Active-Active + Strong** | 모든 리전 | **Strong Consistency** | Aurora DSQL, DynamoDB Global Tables (강한 일관성 모드) |

---

### 5-1. Multi-Region 읽기 복제본

- 변경사항 **비동기 복제** → 읽기를 로컬 리전에서 처리
- 쓰기는 Primary 리전으로만
- 용도: 글로벌 읽기 성능 향상 + DR(재해복구)

:::tip
글로벌 읽기 성능 + 쓰기는 단일 리전 허용 → **Multi-Region Read Replica**
:::
---

### 5-2. Active-Active + Eventual Consistency

**DynamoDB Global Tables:**

- 모든 리전에서 읽기 + 쓰기 동시 가능
- 비동기 복제 (보통 1초 이내)
- 충돌 해결: **Last Writer Wins**
- 리전 장애 시 → 다른 리전으로 즉시 전환, **DB Failover 불필요**

**MemoryDB Multi-Region:**

- 비동기 복제, 마이크로초 읽기, 99.999%

**Keyspaces:**

- Apache Cassandra 호환, Active-Active, 99.999%

:::tip
- 전 세계 사용자 쓰기 + 약간의 일관성 지연 허용 → **DynamoDB Global Tables (Eventual)**
- 충돌 해결 = **Last Writer Wins**
:::
---

### 5-3. Active-Active + Strong Consistency ⭐

**Aurora DSQL:**

- 서버리스 분산 SQL (PostgreSQL 호환)
- 멀티리전에서 **강한 일관성** 보장
- Failover 없이 항상 최신 데이터 접근
- 인프라 관리 Zero, 읽기·쓰기 독립 스케일링
- SLA: **99.999%** (멀티리전)
- 쓰기 처리량: 분산 SQL DB 중 가장 빠름

**DynamoDB Global Tables (Strong Consistency 신규):**

- 최소 **2개 리전에 동기 쓰기** 후 응답
- 강한 일관성 + Multi-active + 무한 확장
- 리전 장애 시 다른 리전으로 트래픽 전환 → 항상 최신 데이터

:::tip
- 멀티리전 + **Strong Consistency** = **Aurora DSQL** 또는 **DynamoDB Global Tables (강한 일관성)**
- "리전 장애 시 Failover 없이 자동 전환" = 이 두 서비스의 핵심 강점
- Aurora DSQL ≠ Aurora. 별개 서비스임을 명심
:::
---

## 6. Zero-ETL

### 기존 ETL의 문제점

- 직접 코드 작성·테스트·유지보수 필요
- 테이블/필드 변경 시 파이프라인 전체 수정
- **복잡하고 취약하며 확장 한계** 존재
- 데이터 이동 지연(lag) → 실시간 분석 불가
- 적합하지 않은 케이스: 사기 탐지, 광고 최적화, 공급망 추적

---

### AWS Zero-ETL 통합 구성

```
Sources (운영 DB)                   Targets (분석/검색)
─────────────────                   ───────────────────
Aurora                      →→→     Amazon Redshift
DocumentDB                  →→→     OpenSearch Service
RDS for MySQL               →→→     SageMaker Lakehouse
DynamoDB                    →→→     S3, Glue
RDS, CloudWatch             →→→
```

**특징:**

- 코드 없이 클릭 몇 번으로 설정 완료
- 페타바이트 규모 운영 데이터 **Near-Real-Time** 분석/검색
- 여러 Aurora 인스턴스 → 단일 Redshift로 통합 → 전사 인사이트

:::tip
- "ETL 파이프라인 없이 + 실시간 분석" → **Zero-ETL**
- **여러 Aurora 인스턴스 → 단일 Redshift 통합 패턴** 암기
- Redshift의 Materialized Views, Data Sharing, Federated Access 함께 활용 가능
:::
---

## 7. Vector Search & Generative AI

### FM/LLM의 두 가지 한계

1. **Knowledge Cut-off** — 훈련 데이터 기준일 이후 정보 없음
2. **도메인 특화 지식 부재** — 사내 데이터, 전용 지식 없음

→ **해결책: RAG (Retrieval Augmented Generation)**

---

### RAG 워크플로우

```
[Ingestion 단계]
소스 데이터 (S3 등)
    ↓ 청킹 (Context Window 크기로 분할)
임베딩 모델 (Embedding Model)
    ↓ 벡터 생성
벡터 DB 저장

[Agency 단계]
사용자 쿼리
    ↓ 임베딩 변환
벡터 유사도 검색 (ANN 알고리즘: HNSW, IVFFlat)
    ↓ 관련 청크 검색
FM에 컨텍스트로 주입
    ↓
응답 생성
```

:::tip
- ANN 알고리즘: **HNSW**, **IVFFlat** 이름 기억
- **Bedrock Knowledge Bases** = Ingestion + Agency 워크플로우 **완전 자동화**
:::
---

### 벡터 지원 AWS DB (기존 DB에 벡터 내장)

| 서비스 | 벡터 지원 방식 |
| --- | --- |
| Aurora PostgreSQL | **pgvector** 익스텐션 |
| RDS for PostgreSQL | **pgvector** 익스텐션 |
| DynamoDB | Zero-ETL → OpenSearch 연동 |
| Amazon Neptune | GraphRAG 지원 |
| MemoryDB | 인메모리 벡터 검색 |
| DocumentDB | 벡터 검색 내장 |
| OpenSearch Service | Vector Engine (Serverless 포함) |

**핵심 이점:** 기존 운영 DB에 벡터 기능 내장
→ 별도 전용 벡터 DB 불필요 = **데이터 마이그레이션·중복·추가 비용·운영 복잡도 모두 제거**

---

### GraphRAG (Amazon Neptune 기반)

- Neptune이 여러 소스(텍스트·이미지·영상·오디오 등 **비정형** 포함) 자동 그래프화
- 검색 시 그래프 탐색 → 더 포괄적·정확한 LLM 응답
- **조직 데이터의 80% 이상이 비정형** → GraphRAG의 강점
- **Bedrock Knowledge Bases에 Neptune 통합 내장**

:::tip
- 비정형 데이터 + 관계 중심 검색 + RAG = **GraphRAG + Neptune**
- S3 비정형 데이터 자동 그래프 생성 가능
:::
---

## 8. 데이터 마이그레이션

### AWS DMS (Database Migration Service)

- 온프레미스 / 타 클라우드 DB → AWS DB 마이그레이션
- 지원 유형:
    - **Homogeneous (동종):** MySQL → Aurora MySQL
    - **Heterogeneous (이종):** Oracle → Aurora PostgreSQL (SCT 필요)
- **SCT (Schema Conversion Tool):** 이기종 DB 스키마 자동 변환

:::tip
마이그레이션 + 운영 중단 최소화 → **DMS**. 이기종 변환 → **SCT + DMS** 조합
:::
---

### 마이그레이션 지원 프로그램

| 프로그램 | 내용 |
| --- | --- |
| **AWS Professional Services** | 전문가 마이그레이션 지원 |
| **Database Migration Accelerator (DMA)** | 고정 비용, DB+앱 변환 AWS 팀 대행 |
| **Database Freedom** | 자격 고객 대상 전문 조언 + 마이그레이션 지원 |
| **AWS DMS Partners** | 파트너 생태계 활용 |

:::tip
레거시 상용 DB(Oracle 등) 탈피 + 비용 절감 = **Database Freedom** 프로그램
:::
---

## 9. DB 서비스 선택 매트릭스

| 서비스 | 유형 | 주요 Use Case | HA SLA | 스케일링 |
| --- | --- | --- | --- | --- |
| **Amazon Aurora** | Relational (MySQL/PG) | OLTP, 금융, 이커머스 | 99.99% | Read Replica + Serverless v2 |
| **Aurora DSQL** | Distributed SQL (PG) | 멀티리전 OLTP + Strong Consistency | **99.999%** | 완전 서버리스, 자동 수평 |
| **Amazon RDS** | Relational (6개 엔진) | 일반 관계형 | 99.99% | Read Replica + Multi-AZ |
| **DynamoDB** | Key-Value / Document | 고처리량 OLTP, 세션, 카탈로그 | **99.999%** | 완전 서버리스, Global Tables |
| **ElastiCache** | In-Memory Cache | DB 캐시, 세션 스토어 | 99.99% | 클러스터 5억 req/s |
| **MemoryDB** | In-Memory DB (Durable) | 인메모리 성능 + 내구성 | **99.999%** | 멀티리전 Active-Active |
| **Neptune** | Graph DB | 소셜 그래프, 추천, GraphRAG | 99.99% | Serverless 지원 |
| **OpenSearch** | Search / Analytics | 전문 검색, 로그 분석, 벡터 | - | Serverless Vector Engine |
| **Keyspaces** | Wide-Column (Cassandra) | IoT, 타임스탬프 데이터 | **99.999%** | 완전 서버리스, 멀티리전 |
| **DocumentDB** | Document (MongoDB) | JSON 문서, 콘텐츠 관리 | 99.99% | 읽기 복제본 |
| **Timestream** | Time-Series | IoT 센서, 모니터링 메트릭 | 99.99% | Serverless (LiveAnalytics) |

---

## 🎯 시험 단골 패턴 요약

| 상황 | 정답 |
| --- | --- |
| 가변 트래픽 + 비용 최적화 (unpredictable traffic/pattern + cost-effective) | Serverless 계열 |
| 읽기 성능 향상 (read heavy) | Read Replica 추가 |
| 데이터 손실 허용 캐시 (data loss allowed) | ElastiCache |
| 데이터 손실 불허 인메모리 (data loss not allowed) | MemoryDB |
| 멀티리전 Eventually Consistent | DynamoDB Global Tables |
| 멀티리전 Strong Consistency | Aurora DSQL 또는 DynamoDB Global Tables (Strong) |
| 다운타임 없는 메이저 업그레이드 (major upgrade without downtime) | Blue/Green Deployments |
| ETL 없는 실시간 분석 | Zero-ETL (→ Redshift / OpenSearch) |
| RAG 자동화 | Bedrock Knowledge Bases |
| 그래프 기반 추천/관계 탐색 | Amazon Neptune |
| 이기종 DB 마이그레이션 | DMS + SCT |
| 초대규모 OLTP 쓰기 확장 | Aurora PostgreSQL Limitless |