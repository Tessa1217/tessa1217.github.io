---
title: 📊 AWS Database & Data Analytics
published: 2026-03-22
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 📊 AWS Database & Data Analytics

> RDS · Aurora · ElastiCache · DynamoDB · DocumentDB · Neptune · Keyspaces · Timestream
> 
> 
> Athena · Redshift · OpenSearch · EMR · QuickSight · Glue · Lake Formation · Flink · MSK
> 
---
## 목차
1. [DB 유형 전체 비교](#db-유형-전체-비교)
2. [RDS (요약)](#rds-요약)
3. [Aurora (요약)](#aurora-요약)
4. [ElastiCache (요약)](#elasticache-요약)
5. [DocumentDB](#documentdb)
6. [Amazon Neptune](#amazon-neptune)
7. [Amazon Keyspaces (for Apache Cassandra)](#amazon-keyspaces-for-apache-cassandra)
8. [Amazon Timestream](#amazon-timestream)
9. [Amazon Athena](#amazon-athena)
10. [Amazon Redshift](#amazon-redshift)
11. [Amazon OpenSearch Service](#amazon-opensearch-service)
12. [Amazon EMR (Elastic MapReduce)](#amazon-emr-elastic-mapreduce)
13. [Amazon QuickSight](#amazon-quicksight)
14. [AWS Glue](#aws-glue)
15. [AWS Lake Formation](#aws-lake-formation)
16. [Amazon Managed Service for Apache Flink](#amazon-managed-service-for-apache-flink)
17. [Amazon MSK (Managed Streaming for Apache Kafka)](#amazon-msk-managed-streaming-for-apache-kafka)
18. [빅데이터 수집 파이프라인 아키텍처](#빅데이터-수집-파이프라인-아키텍처)
19. [📌 시험 자주 출제 포인트](#-시험-자주-출제-포인트)
---

## DB 유형 전체 비교

| 유형 | 서비스 | 특징 |
| --- | --- | --- |
| **RDBMS (OLTP)** | RDS, Aurora | SQL, JOIN 가능 |
| **NoSQL** | DynamoDB, ElastiCache, Neptune, DocumentDB, Keyspaces | JOIN/SQL 없음 |
| **Object Store** | S3, S3 Glacier | 대용량 객체, 아카이브 |
| **Data Warehouse (OLAP)** | Redshift, Athena, EMR | SQL 분석, BI |
| **Search** | OpenSearch | 전체 텍스트 검색, 비정형 |
| **Graph** | Neptune | 관계 데이터 시각화 |
| **Ledger** | QLDB (Quantum Ledger DB) | 변경 불가 트랜잭션 이력 |
| **Time Series** | Timestream | 시계열 데이터 |

---

## RDS (요약)

- Managed: PostgreSQL / MySQL / Oracle / SQL Server / DB2 / MariaDB / Custom
- Provisioned Instance Size + EBS Volume
- Read Replicas, Multi-AZ, Storage Auto Scaling
- IAM, Security Groups, KMS(rest), SSL(transit)
- PITR 최대 35일, Manual Snapshot 무제한
- RDS Custom: Oracle/SQL Server 전용, OS 및 DB 직접 접근 가능
- **Use Case**: RDBMS/OLTP, SQL 쿼리, 트랜잭션

---

## Aurora (요약)

- PostgreSQL/MySQL API 호환, **스토리지와 컴퓨팅 분리**
- 스토리지: **3 AZ × 2 = 6개 복사본**, Self-healing, Auto Scaling
- 컴퓨팅: Multi-AZ DB Cluster, Read Replica Auto Scaling
- Aurora Serverless: 예측 불가/간헐적 워크로드
- Aurora Global: 리전당 최대 16 Read Instance, 리전 간 복제 **< 1초**
- Aurora ML: SageMaker/Comprehend 통합
- Aurora Cloning: 기존 클러스터에서 빠르게 새 클러스터 생성 (스테이징 DB)
- **Use Case**: RDS와 동일 + 더 높은 성능, 가용성, 유연성

---

## ElastiCache (요약)

- Managed Redis / Memcached, **Sub-millisecond** 지연
- Redis: Multi-AZ, Read Replicas, Backup, AOF 영속성
- Memcached: Sharding, 비영속, 멀티스레드
- 보안: IAM, Security Groups, KMS, Redis AUTH
- **애플리케이션 코드 변경 필요**
- **Use Case**: Key/Value 캐시, 세션 스토어, DB 쿼리 결과 캐시

---

## DocumentDB

- Aurora의 AWS 구현처럼, DocumentDB는 **MongoDB**의 AWS 구현
- JSON 데이터 저장/쿼리/인덱싱
- Aurora와 유사한 배포 개념: Fully Managed, 3 AZ 고가용성
- 스토리지 자동 증가 (10 GB 단위)
- 초당 수백만 req로 자동 확장
- **Use Case**: MongoDB 워크로드를 AWS로 이전

---

## Amazon Neptune

- **완전 관리형 Graph Database**
- 수십억 개 관계 저장, **Millisecond** 단위 쿼리
- 3 AZ 고가용성, 최대 15 Read Replicas
- **Use Cases**: 소셜 네트워크, 지식 그래프(Wikipedia), 사기 탐지, 추천 엔진

### Neptune Streams

- Graph 데이터 변경의 **실시간 순서 보장 스트림**
- HTTP REST API로 접근
- Use Cases: 변경 알림, OpenSearch/ElastiCache 동기화, 리전 간 복제

---

## Amazon Keyspaces (for Apache Cassandra)

- **Apache Cassandra** 호환 완전 관리형 DB
- Serverless, Scalable, Multi-AZ (3 AZ 복제)
- **Cassandra Query Language (CQL)** 사용
- **Single-digit millisecond** 지연, 초당 수천 req
- On-Demand 또는 Provisioned with Auto-scaling
- PITR 최대 35일
- **Use Cases**: IoT 디바이스 데이터, 시계열 데이터

---

## Amazon Timestream

- 완전 관리형 **Time Series Database**, Serverless
- 하루 수조 개 이벤트 처리, 관계형 DB 대비 **100배 빠름, 1/10 비용**
- 스토리지 티어링: 최근 데이터 → 메모리 / 과거 데이터 → 저비용 스토리지
- **내장 시계열 분석 함수** (Near real-time 패턴 식별)
- SQL 호환, 암호화(transit/rest)
- **Use Cases**: IoT, 운영 모니터링, 실시간 분석

---

## Amazon Athena

- **S3에 저장된 데이터를 Serverless SQL로 분석**
- Presto 기반, Standard SQL 사용
- 지원 포맷: CSV, JSON, ORC, Avro, Parquet
- **가격: 스캔된 데이터 TB당 $5.00**
- Amazon QuickSight와 연동하여 BI 대시보드 구성

**성능 최적화 (비용 절감):**

| 방법 | 효과 |
| --- | --- |
| **Columnar 포맷** (Parquet, ORC) | 스캔 데이터 감소 → **비용 대폭 절감** |
| Glue로 변환 | CSV → Parquet/ORC 자동 변환 |
| 데이터 압축 | bzip2, gzip, snappy 등 |
| S3 Partitioning | 가상 컬럼 기반 파티셔닝으로 스캔 범위 축소 |
| 큰 파일 사용 | **> 128 MB** 권장 (소형 파일 오버헤드 제거) |

**Federated Query:**

- 관계형/비관계형/S3 등 다양한 소스를 SQL로 통합 쿼리
- **Lambda 기반 Data Source Connector** 사용 (CloudWatch Logs, DynamoDB, RDS 등)

> 📌 **시험 Tip**: "S3 데이터를 Serverless SQL로 분석" → **Athena**
> 

---

## Amazon Redshift

- PostgreSQL 기반이지만 **OLAP (데이터 웨어하우스)**, OLTP 아님
- **Columnar Storage + 병렬 쿼리 엔진** → Petabyte 규모 분석
- Athena보다 **Index 덕분에 더 빠른 Join/집계**
- BI 도구 (QuickSight, Tableau) 통합

### 클러스터 구조

```
[Leader Node]   : 쿼리 계획 + 결과 집계
[Compute Nodes] : 실제 쿼리 실행 (N개)
```

두 가지 모드: **Provisioned Cluster** / **Serverless Cluster**

### Snapshots & DR

- 일부 클러스터에 **Multi-AZ** 모드 지원
- Snapshot = 클러스터의 PITR 백업 (S3 내부 저장, Incremental)
- 자동 백업: 8시간마다 / 5 GB마다 / 스케줄 기반
- 수동 백업: 명시적 삭제 전까지 유지
- **다른 리전으로 Snapshot 자동 복사** 설정 가능

### 데이터 로딩 방법

| 방법 | 설명 |
| --- | --- |
| **Kinesis Data Firehose** | Firehose → S3 → COPY로 Redshift 적재 |
| **S3 COPY 명령** | IAM Role로 S3에서 직접 COPY (VPC 라우팅 가능) |
| **EC2 + JDBC** | 배치로 데이터 전송 (대량 Insert가 유리) |

### Redshift Spectrum

- **S3 데이터를 Redshift에 로딩하지 않고 직접 쿼리**
- Redshift Cluster가 있어야 함 (쿼리는 수천 개 Spectrum 노드에서 처리)

---

## Amazon OpenSearch Service

- Amazon ElasticSearch의 후속 서비스
- **어떤 필드든 전체 텍스트 검색 가능** (DynamoDB는 Primary Key/Index만)
- 다른 DB의 **보완재**로 일반적으로 사용
- Managed Cluster 또는 Serverless 모드
- 기본 SQL 미지원 (플러그인으로 활성화 가능)
- 데이터 수집: Kinesis Data Firehose, IoT, CloudWatch Logs
- 보안: Cognito, IAM, KMS, TLS
- **OpenSearch Dashboards** (시각화) 포함

### OpenSearch 통합 패턴

**DynamoDB + OpenSearch:**

```
[앱 CRUD] → [DynamoDB] → [DynamoDB Stream] → [Lambda] → [OpenSearch]
[앱 검색] → OpenSearch API → 검색 결과 → DynamoDB에서 상세 조회
```

**CloudWatch Logs:**

```
CloudWatch Logs → Subscription Filter → Lambda → OpenSearch (Real-time)
CloudWatch Logs → Subscription Filter → Firehose → OpenSearch (Near real-time)
```

**Kinesis:**

```
KDS → Firehose → OpenSearch (Near real-time)
KDS → Lambda   → OpenSearch (Real-time)
```

---

## Amazon EMR (Elastic MapReduce)

- **Hadoop 클러스터** 기반 빅데이터 분석 플랫폼
- 수백 개의 EC2 인스턴스로 구성된 클러스터
- Apache Spark, HBase, Presto, Flink 번들 포함
- 프로비저닝/설정 자동화, Auto Scaling, Spot Instance 통합

### 노드 유형 및 구매 옵션

| 노드 유형 | 역할 | 특성 |
| --- | --- | --- |
| **Master Node** | 클러스터 관리, 상태 조율 | Long-running |
| **Core Node** | 작업 실행 + 데이터 저장 | Long-running |
| **Task Node** | 작업 실행만 | 일반적으로 **Spot Instance** |

| 구매 방식 | 특징 |
| --- | --- |
| **On-Demand** | 안정적, 종료 없음 |
| **Reserved** | 비용 절감 (가용 시 EMR 자동 사용) |
| **Spot** | 저렴하지만 종료 가능, 덜 안정적 |

---

## Amazon QuickSight

- **Serverless ML 기반 BI(Business Intelligence) 서비스**
- 인터랙티브 대시보드 생성, 자동 확장, Session 단위 과금
- **SPICE 엔진**: 데이터 Import 시 In-memory 계산
- Enterprise 버전: **Column-Level Security (CLS)**

**통합 데이터 소스:**

- RDS, Aurora, Redshift, Athena, S3, OpenSearch, Timestream
- Salesforce, Jira, Teradata, 온프레미스 DB (JDBC)
- 파일: xlsx, csv, json, tsv, elf/clf

**대시보드 공유:**

- Users (Standard) / Groups (Enterprise) — QuickSight 내부 개념 (IAM과 별개)
- 대시보드 공유 전 반드시 **Publish** 필요
- 대시보드를 보는 사용자는 **기저 데이터도 볼 수 있음**

---

## AWS Glue

- **완전 관리형 ETL (Extract, Transform, Load) 서비스**, Serverless
- S3 또는 RDS 데이터 → Glue ETL(변환) → Redshift 로드

### Glue Data Catalog

- S3, RDS, DynamoDB, JDBC → **Glue Data Crawler** → **Glue Data Catalog** (메타데이터)
- Athena, Redshift Spectrum, EMR이 Catalog를 통해 데이터 검색

### Glue 주요 기능

| 기능 | 설명 |
| --- | --- |
| **Job Bookmarks** | 이미 처리한 데이터 재처리 방지 |
| **DataBrew** | 코드 없는 사전 빌드 변환으로 데이터 정제 |
| **Glue Studio** | ETL Job 생성/실행/모니터링 GUI |
| **Streaming ETL** | Kinesis Data Streams, Kafka, MSK 기반 스트리밍 ETL (Spark Structured Streaming) |

### CSV → Parquet 변환 패턴

```
[S3 Input] ─(Put)─→ Glue ETL (CSV → Parquet 변환) → [S3 Output]
                              ↑ Lambda/EventBridge 트리거 가능
→ Athena가 Parquet 파일로 훨씬 적은 비용으로 쿼리 가능
```

---

## AWS Lake Formation

- **Data Lake** = 분석용 중앙 데이터 저장소
- 복잡한 수동 단계(수집, 정제, 이동, 카탈로그) 자동화
- 정형/비정형 데이터 결합
- Source Blueprints: S3, RDS, 관계형/NoSQL DB
- **Row-level / Column-level Fine-grained Access Control**
- **AWS Glue 위에 구축**

### 중앙 권한 관리

```
[Athena] ─→ [QuickSight]
[Lake Formation] ← Column-level 접근 제어 설정
→ Athena, QuickSight가 Lake Formation의 권한을 따름
→ 각 서비스별로 별도 접근 제어 불필요 → 중앙 집중식 보안
```

---

## Amazon Managed Service for Apache Flink

- 이전 이름: **Kinesis Data Analytics for Apache Flink**
- Java, Scala, SQL로 데이터 스트림 처리
- 소스: **Kinesis Data Streams** 또는 **Amazon MSK** (Kafka)
- 완전 관리형: 프로비저닝, 병렬 처리, 자동 스케일링, 백업(Checkpoint/Snapshot)
- ⚠️ **Amazon Data Firehose에서 직접 읽기 불가** (Data Streams에서만)

---

## Amazon MSK (Managed Streaming for Apache Kafka)

- **Kinesis의 대안**: AWS에서 완전 관리형 Kafka
- Kafka Broker Node + Zookeeper 노드 자동 관리
- VPC 내 배포, Multi-AZ (최대 3 AZ)
- 데이터를 **EBS에 원하는 기간만큼 저장**
- **MSK Serverless**: 용량 관리 없이 Kafka 실행

### Kinesis Data Streams vs. Amazon MSK

| 항목 | Kinesis Data Streams | Amazon MSK |
| --- | --- | --- |
| **메시지 크기** | **1 MB 제한** | 기본 1 MB, **최대 10 MB** 설정 가능 |
| **구조** | Shards | **Kafka Topics with Partitions** |
| **확장** | Shard 분할/병합 가능 | 파티션 **추가만** 가능 |
| **암호화 (in-flight)** | TLS | PLAINTEXT 또는 TLS |
| **암호화 (at-rest)** | KMS | KMS |

**MSK Consumers**: Apache Flink, Glue (Streaming ETL), Lambda, EC2/ECS/EKS 앱

---

## 빅데이터 수집 파이프라인 아키텍처

```
[IoT Devices]
      │
      ▼
[Kinesis Data Streams]  ← 실시간 수집
      │
      ▼
[Kinesis Data Firehose] ← Near real-time 전달
      │
      ▼
[S3 (Ingestion Bucket)] ← Raw 데이터 저장
      │         │
      │         ▼ (선택)
      │    [SQS → Lambda] ← 이벤트 기반 처리
      │
      ▼
[Amazon Athena]         ← Serverless SQL 분석
      │
      ▼
[S3 (Reporting Bucket)] ← 분석 결과 저장
      │
      ├──→ [QuickSight]  ← BI 대시보드
      └──→ [Redshift]    ← 데이터 웨어하우스
```

---

## 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| DocumentDB | **MongoDB** 호환 AWS 구현 |
| Neptune Use Case | 소셜 그래프, 사기 탐지, 추천 엔진 |
| Keyspaces | **Apache Cassandra** 호환 |
| Timestream | **시계열 DB**, IoT/운영 모니터링 |
| Athena 가격 | **스캔된 TB당 $5** |
| Athena 비용 절감 | **Parquet/ORC** 포맷 + Partitioning |
| Athena Federated Query | **Lambda Data Source Connector** 사용 |
| Athena vs Redshift | 간단 S3 쿼리 → Athena / 복잡 Join/집계 → **Redshift** |
| Redshift Spectrum | S3 데이터를 **Redshift에 로딩 없이** 쿼리 |
| Redshift 리전 간 Snapshot | **자동 복사** 설정 가능 |
| OpenSearch 특징 | **어떤 필드든** 검색 가능 (DynamoDB와 차이) |
| Glue Catalog | Athena/Redshift Spectrum/EMR이 사용하는 메타데이터 저장소 |
| Lake Formation 특징 | **Row/Column 레벨** 세밀한 접근 제어 |
| Flink 소스 | **Kinesis Data Streams 또는 MSK** (Firehose 아님) |
| MSK vs Kinesis | MSK: 메시지 크기 최대 **10 MB**, 파티션 추가만 가능 |
| QuickSight SPICE | In-memory 계산 엔진 |
| QuickSight Users/Groups | QuickSight 내부 개념 (**IAM과 별개**) |
| EMR Task Node | 주로 **Spot Instance** 사용 |