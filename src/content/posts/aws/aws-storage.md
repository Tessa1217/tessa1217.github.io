---
title: 💾 AWS Storage
published: 2026-03-18
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 💾 AWS Storage

> EC2와 함께 사용하는 핵심 스토리지 서비스 정리
> 
> 
> EBS · EFS · EC2 Instance Store · AMI
---
## 목차
1. [1. 스토리지 유형 비교](#1-스토리지-유형-비교)
2. [2. EBS (Elastic Block Store)](#2-ebs-elastic-block-store)
3. [3. EBS 볼륨 타입](#3-ebs-볼륨-타입)
4. [4. EBS Snapshots (스냅샷)](#4-ebs-snapshots-스냅샷)
5. [5. EBS Encryption (암호화)](#5-ebs-encryption-암호화)
6. [6. EBS Multi-Attach](#6-ebs-multi-attach)
7. [7. EC2 Instance Store](#7-ec2-instance-store)
8. [8. AMI (Amazon Machine Image)](#8-ami-amazon-machine-image)
9. [9. EFS (Elastic File System)](#9-efs-elastic-file-system)
10. [10. 핵심 비교 요약](#10-핵심-비교-요약)
11. [📌 시험 자주 출제 포인트](#-시험-자주-출제-포인트)
12. [📚 참고 자료](#-참고-자료)
---

## 1. 스토리지 유형 비교

```
EC2 인스턴스가 사용할 수 있는 스토리지 종류

├── Block Storage (블록 스토리지)
│   ├── EBS (Elastic Block Store)  — 네트워크 연결, 영구, AZ 종속
│   └── EC2 Instance Store         — 하드웨어 직결, 임시(Ephemeral), 초고속
│
└── File Storage (파일 스토리지)
    └── EFS (Elastic File System)  — 네트워크 파일시스템, 멀티 AZ, 멀티 인스턴스
```

| 항목 | EBS | EC2 Instance Store | EFS |
| --- | --- | --- | --- |
| 연결 방식 | 네트워크 (Network-attached) | 하드웨어 직결 (Hardware) | 네트워크 파일시스템 |
| 영속성 | ✅ 영구 (인스턴스 종료 후에도 유지 가능) | ❌ 임시 (인스턴스 중지/종료 시 소멸) | ✅ 영구 |
| AZ 범위 | 특정 AZ에 종속 | 특정 인스턴스에 종속 | **멀티 AZ 공유 가능** |
| 성능 | 좋음 (설정에 따라 다름) | **최고 (직결 I/O)** | 좋음 (처리량 확장 가능) |
| 공유 가능 | 제한적 (Multi-Attach: io1/io2만) | ❌ | **✅ 수백 인스턴스 동시 마운트** |
| OS 지원 | Linux, Windows | Linux, Windows | **Linux 전용 (POSIX)** |
| 비용 | 프로비저닝 용량 기준 과금 | 인스턴스 비용에 포함 | 사용량 기준 과금 (gp2의 약 3배) |

---

## 2. EBS (Elastic Block Store)

### 📦 개념

- **"네트워크 USB 스틱 (Network USB Stick)"** — 인스턴스와 네트워크로 통신
- 실행 중인 인스턴스에 연결 가능한 **네트워크 드라이브 (Network Drive)**
- 인스턴스 종료(Terminate) 후에도 **데이터 영구 보존** 가능
- **특정 AZ에 종속** — AZ 간 이동 불가 (스냅샷을 통해 이전)

```
[EC2 Instance]  ←── 네트워크 ───→  [EBS Volume]
  us-east-1a                         us-east-1a (동일 AZ 필수)
```

---

### ⚙️ 주요 특성

| 속성 | 설명 |
| --- | --- |
| **용량 (Capacity)** | 생성 시 크기(GiB)와 IOPS를 미리 프로비저닝 |
| **과금 방식** | **프로비저닝한 전체 용량** 기준 (사용량 무관) |
| **용량 변경** | 운영 중 온라인으로 증가 가능 (감소 불가) |
| **연결 대상** | 기본적으로 **한 인스턴스에만** 연결 (Multi-Attach 예외) |

---

### 🗑️ Delete on Termination 속성

| 볼륨 유형 | 기본값 | 의미 |
| --- | --- | --- |
| **루트 EBS 볼륨 (Root Volume)** | **활성화 (Enabled)** | 인스턴스 종료 시 자동 삭제 |
| **추가 EBS 볼륨 (Additional Volume)** | **비활성화 (Disabled)** | 인스턴스 종료 후에도 볼륨 유지 |

> 💡 **실무 팁**: 데이터 보호가 필요한 경우 루트 볼륨의 Delete on Termination을 비활성화하거나, 별도 데이터 볼륨을 추가로 구성하세요.
> 

---

## 3. EBS 볼륨 타입

> **시험 핵심**: 부트 볼륨(Boot Volume)으로 사용 가능한 타입은 **gp2, gp3, io1, io2 Block Express** 뿐. HDD 계열(st1, sc1)은 부트 볼륨 불가.
> 

---

### 📊 전체 타입 비교

| 타입 | 분류 | 크기 범위 | 최대 IOPS | 최대 처리량 | 부트 볼륨 | 주요 용도 |
| --- | --- | --- | --- | --- | --- | --- |
| **gp3** | SSD 범용 | 1GiB~16TiB | 16,000 | 1,000 MiB/s | ✅ | 대부분의 워크로드 ⭐ |
| **gp2** | SSD 범용 | 1GiB~16TiB | 16,000 | 250 MiB/s | ✅ | gp3 전환 권장 |
| **io1** | SSD 고성능 | 4GiB~16TiB | 64,000 (Nitro) | 1,000 MiB/s | ✅ | 고성능 DB |
| **io2 Block Express** | SSD 고성능 | 4GiB~64TiB | **256,000** | 4,000 MiB/s | ✅ | 미션 크리티컬 DB |
| **st1** | HDD 처리량 최적화 | 125GiB~16TiB | 500 | 500 MiB/s | ❌ | 빅데이터, 로그 |
| **sc1** | HDD 저비용 | 125GiB~16TiB | 250 | 250 MiB/s | ❌ | 콜드 데이터, 최저가 |

---

### 🔵 General Purpose SSD (gp3 / gp2)

**gp3** — 현재 권장 범용 타입 ⭐

- 기본 IOPS 3,000, 처리량 125 MiB/s 제공
- IOPS(최대 16,000)와 처리량(최대 1,000 MiB/s)을 **독립적으로** 설정 가능
- 적합: 시스템 부트 볼륨, 가상 데스크톱, 개발/테스트 환경

**gp2** — 구세대 (마이그레이션 권장)

- IOPS와 볼륨 크기가 **연동** (3 IOPS/GiB), 크기 늘려야 IOPS 증가
- 5,334 GiB에서 최대 IOPS(16,000) 도달
- 소규모 볼륨은 3,000 IOPS까지 버스트 가능

> 💡 **gp2 → gp3 전환**: 동일 성능 기준 약 20% 비용 절감. 독립 IOPS 설정으로 유연성도 향상.
> 

---

### 🟠 Provisioned IOPS SSD (io1 / io2 Block Express)

- **지속적인 고IOPS**가 필요한 미션 크리티컬 비즈니스 애플리케이션
- DB 워크로드처럼 **스토리지 성능과 일관성에 민감한** 애플리케이션에 최적
- **16,000 IOPS 초과**가 필요한 경우 필수

| 항목 | io1 | io2 Block Express |
| --- | --- | --- |
| 크기 | 4GiB ~ 16TiB | 4GiB ~ **64TiB** |
| 최대 PIOPS | 64,000 (Nitro EC2) / 32,000 (기타) | **256,000** |
| IOPS:GiB 비율 | 50:1 | **1,000:1** |
| 지연 시간 | 낮음 | **서브 밀리초 (Sub-millisecond)** |
| EBS Multi-Attach | ✅ | ✅ |

---

### 🟤 Hard Disk Drives (st1 / sc1)

- **부트 볼륨으로 사용 불가**
- 크기 범위: 125GiB ~ 16TiB

| 타입 | 별칭 | 최대 처리량 | 최대 IOPS | 적합한 용도 |
| --- | --- | --- | --- | --- |
| **st1** | Throughput Optimized HDD (처리량 최적화 HDD) | 500 MiB/s | 500 | 빅데이터, 데이터 웨어하우스, 로그 처리 |
| **sc1** | Cold HDD (콜드 HDD) | 250 MiB/s | 250 | 비정기 접근 데이터, 최저 비용 |

---

## 4. EBS Snapshots (스냅샷)

### 📸 개념

- 특정 시점의 EBS 볼륨 백업 (Point-in-time Backup)
- 스냅샷 시 볼륨을 꼭 분리(Detach)할 필요는 없으나 **권장**
- 스냅샷은 **AZ 또는 리전 간 복사** 가능 → AZ 간 볼륨 이전 수단

```
us-east-1a [EBS Volume] → 스냅샷 생성 → us-east-1b [새 EBS Volume] 복원
                             (S3에 저장됨)
```

---

### ✨ 스냅샷 주요 기능

| 기능 | 설명 | 비용/속도 |
| --- | --- | --- |
| **EBS Snapshot Archive** | 스냅샷을 '아카이브 티어(Archive Tier)'로 이동 | **75% 저렴**, 복원에 24~72시간 소요 |
| **Recycle Bin (휴지통)** | 삭제된 스냅샷을 지정 기간 동안 보존 | 보존 기간: 1일~1년 설정 가능 |
| **Fast Snapshot Restore (FSR)** | 스냅샷의 전체 초기화를 강제하여 첫 사용 시 지연 없음 | 비용 높음 ($$$), 즉시 사용 필요 시 |

> 💡 **실무 팁**: 자주 접근하지 않는 스냅샷은 Archive Tier로 이동하고, 실수 삭제 방지를 위해 Recycle Bin 규칙을 설정하세요.
> 

---

## 5. EBS Encryption (암호화)

### 🔒 암호화된 EBS 볼륨의 특성

- 볼륨 내 저장 데이터 (Data at rest) 암호화
- 인스턴스 ↔ 볼륨 간 전송 데이터 (Data in flight) 암호화
- 해당 볼륨으로 생성된 모든 스냅샷 암호화
- 암호화된 스냅샷으로 생성한 모든 볼륨 암호화
- **암호화/복호화는 AWS가 투명하게 처리** (사용자가 별도 조작 불필요)
- 지연 시간에 미치는 영향 최소화
- **AWS KMS (AES-256)** 키 활용

---

### 🔄 미암호화 볼륨 → 암호화 변환 절차

```
1️⃣ 기존 미암호화 EBS 볼륨의 스냅샷 생성
        │
        ▼
2️⃣ 스냅샷 복사 시 암호화 옵션 활성화 (Encrypt the EBS snapshot using copy)
        │
        ▼
3️⃣ 암호화된 스냅샷으로 새 EBS 볼륨 생성 (볼륨도 자동으로 암호화됨)
        │
        ▼
4️⃣ 원본 인스턴스에 새 암호화 볼륨 연결
```

> 📌 **시험 포인트**: 미암호화 스냅샷을 복사(Copy)할 때 암호화를 활성화하면 암호화된 스냅샷으로 변환 가능. 암호화된 볼륨의 스냅샷은 항상 암호화됨.
> 

---

## 6. EBS Multi-Attach

- **동일 AZ 내** 여러 EC2 인스턴스에 **같은 EBS 볼륨**을 동시에 연결
- 각 인스턴스는 볼륨에 대한 **전체 읽기/쓰기 권한** 보유
- **io1 / io2 Block Express 타입만** 지원

| 항목 | 내용 |
| --- | --- |
| **최대 연결 인스턴스 수** | **16개** |
| **사용 가능 파일시스템** | 클러스터 인식 파일시스템 필수 (XFS, EXT4 등 일반 파일시스템 사용 불가) |
| **주요 Use Case** | 클러스터형 Linux 애플리케이션의 고가용성 확보 (ex: Teradata) |

:::warning
일반 XFS, EXT4 파일시스템은 Multi-Attach 환경에서 데이터 충돌 발생. OCFS2, GFS2 같은 클러스터 인식 파일시스템 사용 필요.
:::
---

## 7. EC2 Instance Store

### ⚡ 개념

- EC2 인스턴스에 **물리적으로 직결된 하드웨어 디스크**
- 네트워크 경유 없이 직접 I/O → **EBS보다 훨씬 높은 I/O 성능**

### ⚠️ 핵심 제약: 임시 스토리지 (Ephemeral Storage)

```
인스턴스 중지(Stop) 또는 종료(Terminate) → 데이터 완전 소멸 ❌
인스턴스 재부팅(Reboot) → 데이터 유지 ✅
```

| 항목 | 내용 |
| --- | --- |
| **성능** | 매우 높음 (수백만 IOPS 가능) |
| **영속성** | ❌ 임시 (인스턴스 중지/종료 시 소멸) |
| **하드웨어 장애 시** | 데이터 손실 위험 |
| **백업/복제** | 사용자 책임 |

**적합한 Use Case:**

- 버퍼 (Buffer)
- 캐시 (Cache)
- 임시 데이터 (Scratch data / Temporary content)
- 빠른 읽기/쓰기가 필요한 임시 연산

> ❌ **데이터베이스, 영구 보관 데이터에는 절대 사용 금지**
> 

---

## 8. AMI (Amazon Machine Image)

### 🖼️ 개념

- **AMI (Amazon Machine Image, 아마존 머신 이미지)**: EC2 인스턴스의 커스터마이징 템플릿
- OS, 소프트웨어, 설정, 모니터링 에이전트 등을 미리 패키징
- **빠른 부팅/설정**: 필요한 소프트웨어가 이미 포함되어 있어 User Data 스크립트 최소화
- **특정 리전에 귀속**, 리전 간 복사 가능

---

### 📂 AMI 소스 유형

| 유형 | 설명 | 예시 |
| --- | --- | --- |
| **Public AMI** | AWS가 제공하는 공식 AMI | Amazon Linux 2, Ubuntu, Windows |
| **Own AMI** | 직접 생성·관리하는 커스텀 AMI | 사내 표준 이미지 |
| **AWS Marketplace AMI** | 벤더가 제공하는 상용 AMI | 방화벽, DB, 보안 솔루션 등 |

---

### 🛠️ AMI 생성 프로세스

```
1️⃣ EC2 인스턴스 시작 후 소프트웨어/설정 커스터마이징
        │
        ▼
2️⃣ 인스턴스 중지 (데이터 무결성 보장을 위해 Stop 권장)
        │
        ▼
3️⃣ AMI 빌드 (내부적으로 EBS 스냅샷도 함께 생성)
        │
        ▼
4️⃣ 다른 인스턴스를 해당 AMI로 신속하게 실행
```

> 💡 **골든 AMI (Golden AMI) 패턴**: 표준화된 기업 환경 이미지를 AMI로 만들어 두고, Auto Scaling Group에서 사용. 배포 속도와 일관성 향상.
> 

---

## 9. EFS (Elastic File System)

### 📁 개념

- **관리형 NFS (Network File System)** — 여러 EC2에 동시에 마운트 가능
- **멀티 AZ**에서 동시 접근 가능
- 용량 자동 확장 (페타바이트 규모까지), 사용량 기반 과금 (Pay-per-use)
- Linux 기반 AMI 전용 (**Windows 미지원**, POSIX 파일시스템)
- **NFSv4.1 프로토콜** 사용
- 보안 그룹(Security Group)으로 접근 제어
- KMS 기반 저장 데이터 암호화

```
           [EFS]
          /  |  \
    [EC2]  [EC2]  [EC2]
    AZ-1   AZ-2   AZ-3
    (동시 마운트 가능, 파일 공유)
```

**주요 Use Case**: 콘텐츠 관리(Content Management), 웹 서빙(Web Serving), 데이터 공유, WordPress

---

### ⚡ 성능 모드 (Performance Mode)

> EFS 생성 시 설정, **이후 변경 불가**
> 

| 모드 | 특징 | 적합한 Use Case |
| --- | --- | --- |
| **General Purpose** (기본값) | 저지연, 범용 | 웹 서버, CMS, 일반 파일 공유 |
| **Max I/O** | 높은 지연 허용, 높은 처리량, 고도 병렬 처리 | 빅데이터, 미디어 처리, HPC |

---

### 🔄 처리량 모드 (Throughput Mode)

| 모드 | 설명 | 특징 |
| --- | --- | --- |
| **Bursting** | 저장 용량 기반 처리량 (1TB = 50MiB/s + 최대 100MiB/s 버스트) | 예측 가능한 소규모 워크로드 |
| **Provisioned** | 저장 용량과 무관하게 원하는 처리량 설정 | ex: 1TiB 용량에 1GiB/s 처리량 |
| **Elastic** ⭐ | 워크로드에 따라 처리량 자동 확장/축소 | 읽기 최대 3GiB/s, 쓰기 최대 1GiB/s, **예측 불가한 워크로드에 권장** |

---

### 🗂️ 스토리지 클래스 (Storage Classes)

> 라이프사이클 정책(Lifecycle Policy)으로 N일 후 자동 계층 이동
> 

| 계층 | 접근 빈도 | 특징 |
| --- | --- | --- |
| **Standard** | 자주 접근 | 기본 계층, 빠른 응답 |
| **Infrequent Access (EFS-IA)** | 비정기 접근 | 저장 비용 저렴, 조회 시 추가 요금 |
| **Archive** | 연 몇 차례 | 50% 더 저렴 |

> 💡 **비용 절감**: 라이프사이클 정책으로 **90% 이상 비용 절감** 가능
> 

---

### 🌐 가용성 및 내구성 (Availability & Durability)

| 유형 | 특징 | 적합한 환경 |
| --- | --- | --- |
| **Standard (Multi-AZ)** | 여러 AZ에 분산 저장 | 프로덕션 |
| **One Zone** | 단일 AZ | 개발/테스트, 비용 절감 필요 시 |
| **One Zone-IA** | 단일 AZ + 비정기 접근 계층 | 최대 비용 절감 |

---

## 10. 핵심 비교 요약

### EBS vs EFS vs Instance Store

```
데이터 영속성
  영구 ──────────────────────────── 임시
  EFS          EBS          EC2 Instance Store

공유 범위
  멀티 인스턴스/멀티 AZ ──────── 단일 인스턴스
  EFS              EBS (기본)    Instance Store

성능
  중간           중간~높음       최고 (직결)
  EFS             EBS            Instance Store
```

| 상황 | 선택 |
| --- | --- |
| OS 루트 볼륨 | **EBS gp3** |
| 고성능 DB (높은 IOPS) | **EBS io2 Block Express** |
| 여러 인스턴스 파일 공유 | **EFS** |
| 임시 캐시·버퍼 (최고속도) | **EC2 Instance Store** |
| 자주 쓰는 이미지 배포 표준화 | **AMI** |
| 콜드 데이터, 최저 비용 | **EBS sc1** or **EFS Archive** |

---

## 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| 부트 볼륨 가능 타입 | gp2, gp3, io1, io2 Block Express (HDD는 불가) |
| EBS AZ 이전 방법 | 스냅샷 생성 → 다른 AZ에서 복원 |
| EBS 암호화 키 | **AWS KMS (AES-256)** |
| 미암호화 → 암호화 | 스냅샷 복사 시 암호화 옵션 활성화 |
| EBS Multi-Attach | io1/io2만, 동일 AZ 내, 최대 16개 인스턴스 |
| Instance Store 데이터 유지 | 재부팅(Reboot)만 유지, Stop/Terminate 시 소멸 |
| EFS OS 지원 | Linux 전용 (POSIX), Windows 미지원 |
| EFS 처리량 모드 권장 | 예측 불가 워크로드 → **Elastic** |
| EFS 비용 절감 | 라이프사이클 정책 → 최대 90% 절감 |

---

## 📚 참고 자료

- [EBS 볼륨 타입 공식 문서](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
- [EBS 스냅샷 공식 문서](https://docs.aws.amazon.com/ebs/latest/userguide/EBSSnapshots.html)
- [EFS 공식 문서](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)
- [EC2 Instance Store 공식 문서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)