---
title: 🗄️ AWS Storage — Extended Services
published: 2026-03-21
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🗄️ AWS Storage — Extended Services

> Snow Family · FSx · Storage Gateway · Transfer Family · DataSync
> 
> 
> Hybrid Cloud, Data Migration, High-Performance File Systems의 핵심 서비스
---
## 목차 
1. [AWS Snow Family](#aws-snow-family)
2. [Amazon FSx](#amazon-fsx)
3. [AWS Storage Gateway](#aws-storage-gateway)
4. [AWS Transfer Family](#aws-transfer-family)
5. [AWS DataSync](#aws-datasync)
6. [전체 Storage 서비스 비교 (Storage Comparison)](#전체-storage-서비스-비교-storage-comparison)
7. [시험 자주 출제 포인트 총정리](#-시험-자주-출제-포인트-총정리)
8. [참고 자료](#-참고-자료)
---

## AWS Snow Family

### ⚠️ 중요: 단종(Discontinuation) 공지

> **Effective November 12, 2024**: 구세대 Snowball Edge 모델 3종 단종
> 
> - Snowball Edge Storage Optimized 80TB
> - Snowball Edge Compute Optimized (52 vCPUs)
> - Snowball Edge Compute Optimized with GPU
> 
> **대부분의 Data Migration 워크로드에서 AWS DataSync를 대안으로 사용 권장**
> 

---

### Snow Family 개요

인터넷/네트워크를 통한 전송이 어렵거나 느린 경우 **물리적 장치(Physical Device)를 통해 데이터를 오프라인으로 이전**하는 서비스.

**네트워크 전송의 문제점:**

```
Limited connectivity      → 오지, 선박, 원격지에서 인터넷 없음
Limited bandwidth         → 대용량 데이터 전송에 수주~수개월 소요
High network cost         → 대역폭 비용 과다
Shared bandwidth          → 업무 트래픽과 경합
Connection instability    → 전송 중 연결 끊김 위험
```

> 📌 **Rule of Thumb**: 네트워크로 전송 시 **1주 이상** 걸린다면 → Snow Family 검토
> 

---

### Snowball Edge

| 모델 | 특징 | 주요 Use Case |
| --- | --- | --- |
| **Storage Optimized** | 스토리지 중심, 최대 수백 TB | Petabyte 규모 Data Migration |
| **Compute Optimized** | EC2, Lambda 실행 가능 | Edge Computing + Data Migration |

---

### Edge Computing (엣지 컴퓨팅)

- 인터넷 연결이 제한되거나 없는 **현장(Edge Location)**에서 데이터 처리
- Snowball Edge를 현장에 배치하여 On-site Computing Power 제공

**Edge Computing Use Cases:**

```
현장 예시: 도로 위 트럭, 바다 위 선박, 지하 광산
사용 목적:
  - EC2 Instance 또는 Lambda 실행
  - 데이터 전처리 (Preprocessing)
  - Machine Learning Inference
  - 미디어 트랜스코딩 (Media Transcoding)
```

---

### Snowball → S3 → Glacier 아키텍처

> 📌 **시험 포인트**: Snowball로 Glacier에 **직접 import 불가**
> 

```
[Snowball 장치] → Import → [Amazon S3] → Lifecycle Policy → [Amazon Glacier]
                                          (자동 전환)
```

Snowball로 데이터를 S3로 먼저 import한 뒤, **S3 Lifecycle Policy**를 이용해 Glacier로 전환해야 함.

---

### Snow Family 서비스 선택 가이드

| 서비스 | 용량 | 형태 | Use Case |
| --- | --- | --- | --- |
| **Snowball Edge** | 수십~수백 TB | 가방 크기 장치 | Petabyte 이하 Migration, Edge Computing |
| **Snowmobile** | 최대 100 PB (Exabyte) | 45피트 트럭 컨테이너 | Data Center 전체 이전 |

> Snowmobile: 10 PB 이상의 대규모 Migration에 Snowball보다 효율적
> 

---

## Amazon FSx

- AWS에서 **3rd-party High-Performance File System**을 완전 관리형으로 제공
- NFS, SMB, Lustre 등 기존 프로토콜 그대로 사용 가능 → On-premises 워크로드를 클라우드로 Lift-and-Shift

---

### FSx for Windows File Server

- **Windows 환경을 위한 완전 관리형 공유 파일 시스템**
- 핵심 지원 기능:
    - **SMB Protocol** + **Windows NTFS**
    - **Microsoft Active Directory** 통합, ACL, 사용자 쿼터(User Quotas)
    - **DFS (Distributed File System) Namespaces**: 여러 파일 시스템을 하나로 묶어 접근

| 항목 | 내용 |
| --- | --- |
| **용량/성능** | 수십 GB/s, 수백만 IOPS, 수백 PB |
| **Linux 지원** | ✅ Linux EC2 Instance에서도 마운트 가능 |
| **On-premises 접근** | ✅ VPN 또는 Direct Connect 경유 |
| **High Availability** | Multi-AZ 구성 지원 |
| **백업** | 일별 S3 자동 백업 |

**Storage Options:**

| 옵션 | 특징 | Use Case |
| --- | --- | --- |
| **SSD** | 저지연, 고IOPS | DB, 미디어 처리, 데이터 분석 |
| **HDD** | 저비용, 높은 처리량 | 홈 디렉터리, CMS |

---

### FSx for Lustre

- **Linux 기반 병렬 분산 파일 시스템 (Parallel Distributed File System)**
- 이름 유래: **"Linux"** + **"cluster"** = **Lustre**
- *HPC(High Performance Computing)**를 위한 최고 성능 파일 시스템

| 항목 | 내용 |
| --- | --- |
| **성능** | 수백 GB/s, 수백만 IOPS, Sub-millisecond Latency |
| **S3 통합** | S3를 파일 시스템처럼 Read 가능, 연산 결과를 S3로 Write 가능 |
| **On-premises 접근** | ✅ VPN 또는 Direct Connect 경유 |

**Storage Options:**

| 옵션 | 특징 | Use Case |
| --- | --- | --- |
| **SSD** | 저지연, 랜덤 I/O | 소규모 랜덤 파일 작업 |
| **HDD** | 고처리량, 순차 I/O | 대규모 순차 파일 작업 |

**Primary Use Cases:** Machine Learning, HPC, Video Processing, Financial Modeling, Electronic Design Automation (EDA)

---

### FSx for Lustre — File System Deployment Options

> 📌 **시험 포인트**: Scratch vs Persistent 차이를 명확히 구분
> 

| 항목 | Scratch File System | Persistent File System |
| --- | --- | --- |
| **목적** | 임시 저장소 | 장기 저장소 |
| **데이터 복제** | ❌ 없음 (서버 장애 시 데이터 소실) | ✅ 동일 AZ 내 복제 |
| **장애 복구** | 없음 | 분 단위 자동 복구 |
| **성능** | **6x 더 빠름 (200 MBps/TiB Burst)** | 표준 |
| **Use Case** | 단기 처리, 비용 최적화 | 장기 처리, 민감 데이터 |

---

### FSx for NetApp ONTAP

- **NetApp ONTAP을 AWS에서 완전 관리형으로 제공**
- 기존 ONTAP 또는 NAS(Network-Attached Storage) 환경 워크로드를 AWS로 이전 시 최적

| 항목 | 내용 |
| --- | --- |
| **지원 프로토콜** | **NFS, SMB, iSCSI** (3종 동시 지원) |
| **OS 호환성** | Linux, Windows, macOS, VMware Cloud on AWS, EC2, ECS, EKS 등 폭넓게 지원 |
| **용량** | 스토리지 자동 축소/확장 |
| **기능** | Snapshots, Replication, 압축(Compression), 중복 제거(Deduplication) |
| **클로닝** | **Point-in-time Instantaneous Cloning** (새 워크로드 테스트에 유용) |

---

### FSx for OpenZFS

- **OpenZFS 파일 시스템을 AWS에서 완전 관리형으로 제공**
- 기존 ZFS 기반 워크로드를 AWS로 이전 시 최적

| 항목 | 내용 |
| --- | --- |
| **지원 프로토콜** | **NFS** (v3, v4, v4.1, v4.2) |
| **성능** | 최대 **1,000,000 IOPS**, **0.5ms 미만** Latency |
| **기능** | Snapshots, 압축(Compression), **Point-in-time Instantaneous Cloning** |

---

### FSx 선택 가이드

```
Windows 파일 서버 필요 (SMB, AD 연동)     → FSx for Windows File Server
Linux HPC, ML, 대규모 병렬 처리           → FSx for Lustre
NFS + SMB + iSCSI 동시 지원, ONTAP 이전  → FSx for NetApp ONTAP
ZFS 이전, 초고속 NFS (100만 IOPS)        → FSx for OpenZFS
```

---

## AWS Storage Gateway

### Hybrid Cloud for Storage

- **On-premises ↔ AWS 클라우드 간 브릿지(Bridge)** 역할
- S3는 독점 스토리지 기술 (NFS처럼 표준 프로토콜이 아님) → On-premises에서 직접 접근 불가 → **Storage Gateway가 해결**

**Use Cases:** Disaster Recovery, Backup & Restore, Tiered Storage, On-premises Cache & Low-latency File Access

---

### S3 File Gateway

- On-premises 애플리케이션이 **NFS / SMB 프로토콜**로 S3에 접근하도록 지원
- 최근 사용 데이터는 File Gateway에 **로컬 캐시** → 빠른 접근 속도

**지원 S3 Storage Classes:**

```
✅ S3 Standard
✅ S3 Standard-IA
✅ S3 One Zone-IA
✅ S3 Intelligent-Tiering
❌ S3 Glacier 직접 지원 안 함
   → Lifecycle Policy로 S3에서 Glacier로 전환
```

**SMB + Active Directory 통합:**

- SMB Protocol 사용 시 **Microsoft AD**로 사용자 인증 가능
- Bucket별로 **IAM Role** 기반 접근 제어

---

### Volume Gateway

- **iSCSI Protocol**을 통한 Block Storage 제공 (S3 백업)
- EBS Snapshot으로 On-premises 볼륨 복원 가능

| 모드 | 설명 | 데이터 위치 |
| --- | --- | --- |
| **Cached Volumes** (캐시 볼륨) | 최근 데이터만 로컬 캐시, 전체는 S3 | 주 데이터: S3 / 캐시: On-premises |
| **Stored Volumes** (저장 볼륨) | 전체 데이터셋 On-premises, S3에 주기적 백업 | 주 데이터: On-premises / 백업: S3 |

---

### Tape Gateway

- 물리 테이프 기반 백업 프로세스를 사용하는 기업을 위한 **Virtual Tape Library(VTL)**
- 기존 테이프 방식의 백업 소프트웨어(iSCSI Interface)를 그대로 사용하면서 클라우드로 이전
- 백업 데이터: **Amazon S3 + Glacier** 백업

```
[On-premises Backup Software] → iSCSI → [Tape Gateway] → [S3 / Glacier]
(기존 테이프 프로세스 유지)                 (가상 테이프 라이브러리)
```

**지원 백업 소프트웨어**: Veeam, Backup Exec, NetBackup 등 주요 벤더 호환

---

### Storage Gateway 유형 선택 가이드

```
NFS/SMB로 S3 접근 (파일 공유)           → S3 File Gateway
iSCSI Block Storage + S3 백업           → Volume Gateway
  ├── 전체 On-premises 저장 + 스케줄 백업   → Stored Volumes
  └── 최근 데이터 캐시 + 나머지는 S3       → Cached Volumes
기존 테이프 백업 소프트웨어 그대로 유지    → Tape Gateway
```

---

## AWS Transfer Family

- Amazon S3 또는 Amazon EFS로의 파일 전송을 위한 **완전 관리형 FTP 서비스**
- 기존 FTP 기반 파일 교환 인프라를 AWS로 이전할 때 사용

### 지원 프로토콜

| 프로토콜 | 설명 | 보안 |
| --- | --- | --- |
| **FTP** | File Transfer Protocol | ❌ 비암호화 |
| **FTPS** | FTP over SSL | ✅ SSL 암호화 |
| **SFTP** | Secure File Transfer Protocol (SSH 기반) | ✅ SSH 암호화 |

### 주요 특성

| 항목 | 내용 |
| --- | --- |
| **인프라** | 완전 관리형, Scalable, Reliable |
| **가용성** | Multi-AZ |
| **과금** | Provisioned Endpoint 시간당 요금 + 데이터 전송(GB) 요금 |
| **인증 통합** | Microsoft AD, LDAP, Okta, Amazon Cognito, Custom |
| **저장 대상** | Amazon S3 또는 Amazon EFS |

**Use Cases:** 파일 공유(File Sharing), 공개 데이터셋 배포, CRM, ERP 시스템 연동

---

## AWS DataSync

- **대용량 데이터를 자동화하여 빠르게 전송**하는 온라인 데이터 이전 서비스

### 전송 방향

| 방향 | Agent 필요 여부 |
| --- | --- |
| **On-premises → AWS** (NFS, SMB, HDFS, S3 API 등) | ✅ DataSync Agent 설치 필요 |
| **다른 Cloud → AWS** | ✅ Agent 필요 |
| **AWS → AWS** (Storage 서비스 간) | ❌ Agent 불필요 |

### 지원 대상 스토리지

**DataSync가 동기화할 수 있는 AWS 서비스:**

```
Amazon S3  (모든 Storage Class, Glacier 포함)
Amazon EFS
Amazon FSx for Windows File Server
Amazon FSx for Lustre
Amazon FSx for NetApp ONTAP
Amazon FSx for OpenZFS
```

### 핵심 특성

| 항목 | 내용 |
| --- | --- |
| **속도** | Agent 1개로 최대 **10 Gbps** 활용 (Bandwidth Throttle 설정 가능) |
| **스케줄링** | 시간별(Hourly), 일별(Daily), 주별(Weekly) 예약 실행 |
| **메타데이터 보존** | **File Permissions + Metadata 그대로 유지** (NFS POSIX, SMB 등) |
| **무결성** | 전송 중 및 완료 후 데이터 무결성 검증 |
| **재시도** | 실패한 전송 자동 재시도 |

---

### DataSync vs Storage Gateway — 실무 패턴

> 📌 **시험 핵심 패턴**: 두 서비스를 함께 쓰는 것이 Best Practice
> 

```
Phase 1 — 초기 Data Migration:
  [On-premises NFS] → [DataSync Agent] → [Amazon S3 / EFS]
  (대용량 데이터를 빠르게 일회성 또는 반복 이전)

Phase 2 — 지속적 On-premises 접근 유지:
  [On-premises App] → [S3 File Gateway] → [Amazon S3]
  (마이그레이션 후에도 On-premises 앱이 로컬처럼 접근)
```

| 항목 | DataSync | Storage Gateway |
| --- | --- | --- |
| **주요 목적** | **데이터 이전(Migration) 및 동기화** | **On-premises ↔ AWS 지속적 연동** |
| **사용 시점** | 초기 대량 이전, 정기 배치 동기화 | 이전 완료 후 지속적 Hybrid 접근 |
| **프로토콜** | NFS, SMB, HDFS, S3 API 등 | NFS, SMB, iSCSI |
| **캐시** | ❌ | ✅ 로컬 캐시 제공 |

---

## 전체 Storage 서비스 비교 (Storage Comparison)

| 서비스 | 분류 | 특징 | 핵심 키워드 |
| --- | --- | --- | --- |
| **S3** | Object Storage | 범용 객체 스토리지 | Buckets, Keys, 11 9's |
| **S3 Glacier** | Object Archival | 장기 아카이브, 저비용 | Vault, 최소 90~180일 |
| **EBS** | Block Storage | EC2 1:1 네트워크 드라이브 | AZ-bound, Snapshot |
| **Instance Store** | Block Storage | EC2 직결 물리 디스크, 임시 | Ephemeral, 최고 IOPS |
| **EFS** | File Storage | Linux 멀티 인스턴스 공유 NFS | POSIX, Multi-AZ |
| **FSx for Windows** | File Storage | Windows SMB, AD 통합 | NTFS, DFS |
| **FSx for Lustre** | File Storage | Linux HPC 병렬 파일 시스템 | Sub-ms, S3 통합 |
| **FSx for NetApp ONTAP** | File Storage | NFS+SMB+iSCSI 멀티 프로토콜 | 광범위 OS 호환 |
| **FSx for OpenZFS** | File Storage | ZFS 관리형, 초고속 NFS | 100만 IOPS, 0.5ms |
| **Storage Gateway** | Hybrid | On-premises ↔ Cloud 브릿지 | S3 File/Volume/Tape |
| **Transfer Family** | File Transfer | FTP/FTPS/SFTP 관리형 | FTP 프로토콜, S3/EFS |
| **DataSync** | Data Transfer | 자동화 온라인 데이터 이전 | Agent, 10Gbps, 스케줄링 |
| **Snow Family** | Physical Transfer | 오프라인 대용량 이전 | 오지/단절 환경, Petabyte |

---

## 📌 시험 자주 출제 포인트 총정리

| 포인트 | 내용 |
| --- | --- |
| Snowball → Glacier | **직접 import 불가** → S3 먼저 import → Lifecycle Policy로 Glacier 전환 |
| "1주 이상 걸리면" | **Snow Family** 검토 |
| Snow Family 대안 (2024~) | 대부분 **AWS DataSync** 권장 |
| Snowmobile 기준 | **10 PB 이상** 대규모 Migration |
| FSx for Windows 특징 | **SMB + NTFS + AD 통합**, Linux EC2에서도 마운트 가능 |
| FSx for Lustre 이름 | **Linux + cluster** |
| FSx Scratch vs Persistent | Scratch: 데이터 복제 없음, 6x 빠름 / Persistent: AZ 내 복제, 분 단위 복구 |
| FSx for NetApp ONTAP 프로토콜 | **NFS + SMB + iSCSI** 동시 지원 |
| FSx for OpenZFS 성능 | **1,000,000 IOPS, <0.5ms** |
| NetApp ONTAP & OpenZFS 공통 기능 | **Point-in-time Instantaneous Cloning** |
| S3 File Gateway Glacier | **직접 지원 안 함** → Lifecycle Policy 경유 |
| S3 File Gateway + SMB | **Active Directory 사용자 인증** 통합 |
| Volume Gateway Cached | 최근 데이터 로컬 캐시, 전체는 **S3** |
| Volume Gateway Stored | 전체 On-premises, 주기적 **S3 백업** |
| Tape Gateway | **Virtual Tape Library**, 기존 테이프 소프트웨어 그대로 사용 |
| DataSync Agent | On-premises → AWS 전송 시 **필수**, AWS → AWS는 불필요 |
| DataSync 속도 | Agent 1개당 최대 **10 Gbps** |
| DataSync 스케줄 | 시간별 / 일별 / **주별** |
| DataSync 메타데이터 | **File Permissions + Metadata 보존** |
| DataSync + File Gateway | 초기 이전은 DataSync, 이후 지속 접근은 **File Gateway** |
| Transfer Family 저장 대상 | **S3 또는 EFS** |
| Transfer Family 프로토콜 | **FTP, FTPS, SFTP** (3가지) |
| Transfer Family 인증 | AD, LDAP, Okta, Cognito, Custom |

---

## 📚 참고 자료

- [AWS DataSync 공식 문서](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html)
- [Amazon FSx 공식 문서](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [AWS Storage Gateway 공식 문서](https://docs.aws.amazon.com/storagegateway/latest/userguide/WhatIsStorageGateway.html)
- [AWS Transfer Family 공식 문서](https://docs.aws.amazon.com/transfer/latest/userguide/what-is-aws-transfer-family.html)
- [Snow Family 공식 문서](https://docs.aws.amazon.com/snowball/latest/developer-guide/whatisedge.html)