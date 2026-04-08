---
title: 🖥️ AWS EC2 (Elastic Compute Cloud)
published: 2026-03-16
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🖥️ AWS EC2 (Elastic Compute Cloud)

> EC2 = **Infrastructure as a Service (IaaS)**
> 
> 
> 가상 서버를 온디맨드로 임대하여 컴퓨팅 리소스를 유연하게 사용하는 서비스
> 

---

## 목차

1. [EC2 개요](#1-ec2-개요)
2. [EC2 인스턴스 타입](#2-ec2-인스턴스-타입)
3. [Security Groups (보안 그룹)](#3-security-groups-보안-그룹)
4. [SSH 접속 방법](#4-ssh-접속-방법)
5. [구매 옵션 (Purchasing Options)](#5-구매-옵션-purchasing-options)
6. [Spot Instance 심화](#6-spot-instance-심화)
7. [IP 주소 & Elastic IP](#7-ip-주소-elastic-ip)
8. [배치 그룹 (Placement Groups)](#8-배치-그룹-placement-groups)
9. [탄력적 네트워크 인터페이스 (ENI)](#9-탄력적-네트워크-인터페이스-eni)
10. [EC2 최대 절전 모드 (Hibernate)](#10-ec2-최대-절전-모드-hibernate)
11. [Best Practices](#11-best-practices)
12. [핵심 요약](#12-핵심-요약)
13. [참고 자료](#-참고-자료)

---

## 1. EC2 개요

### ☁️ EC2가 제공하는 것

| 기능 | 서비스 | 설명 |
| --- | --- | --- |
| **가상 서버 임대** | EC2 | 다양한 OS/스펙의 가상 머신 |
| **가상 드라이브** | EBS | 네트워크 연결 블록 스토리지 |
| **로드 분산** | ELB | 여러 인스턴스에 트래픽 분산 |
| **자동 확장** | ASG | 수요에 따른 인스턴스 수 조절 |

---

### ⚙️ EC2 설정 옵션

```
EC2 인스턴스 구성 요소
├── OS: Linux / Windows / macOS
├── CPU: vCPU 수, 프로세서 아키텍처 (x86, ARM/Graviton)
├── RAM: 메모리 크기
├── Storage
│   ├── Network-attached: EBS (Elastic Block Store), EFS (Elastic File System)
│   └── Hardware: EC2 Instance Store (임시, 고속)
├── Network: 네트워크 카드 속도, 공인 IP 설정
├── Security Group: 방화벽 규칙
└── User Data: 최초 실행 시 부트스트랩 스크립트
```

---

### 🚀 EC2 User Data (부트스트랩)

- 인스턴스 **최초 시작 시 딱 한 번만** 실행
- **root 권한**으로 실행됨
- 자동화할 수 있는 작업 예시:

```bash
#!/bin/bash
# EC2 User Data 예시
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2 $(hostname -f)</h1>" > /var/www/html/index.html
```

> 💡 **활용 팁**: 소프트웨어 설치, 설정 파일 다운로드, 환경변수 설정 등 초기화 작업을 자동화할 때 사용. 복잡한 설정이 필요하면 CloudFormation 또는 Ansible 연계 권장.
> 

---

## 2. EC2 인스턴스 타입

### 🏷️ 명명 규칙

```
m  7  g  .  2xlarge
│  │  │       └── 크기 (nano / micro / small / medium / large / xlarge / 2xlarge ...)
│  │  └────────── 추가 속성 (g=Graviton, a=AMD, i=Intel, n=고속 네트워크)
│  └───────────── 세대 (숫자가 높을수록 최신)
└──────────────── 인스턴스 패밀리 (m=범용, c=컴퓨팅, r=메모리, i=스토리지 등)
```

---

### 📊 인스턴스 패밀리별 용도

| 패밀리 | 대표 타입 | 특징 | 주요 Use Case |
| --- | --- | --- | --- |
| **General Purpose** | m7i, m7g, t3 | CPU/메모리/네트워크 균형 | 웹 서버, 코드 저장소, 소규모 DB |
| **Compute Optimized** | c7i, c7g, c6a | 고성능 CPU 중심 | 배치 처리, 미디어 트랜스코딩, HPC, ML |
| **Memory Optimized** | r7i, r7g, x2 | 대용량 메모리 | 인메모리 DB, 실시간 빅데이터, SAP HANA |
| **Storage Optimized** | i4i, d3, h1 | 고속 로컬 스토리지 | OLTP, Redis 캐시, 데이터 웨어하우스 |
| **Accelerated Computing** | p4, g5, trn1, inf2 | GPU/NPU 탑재 | AI/ML 학습·추론, 게임 스트리밍 |

---

### ⚡ Graviton (ARM 기반) 인스턴스 — 2025 핵심 트렌드

AWS Graviton 프로세서 기반 EC2 인스턴스는 x86 기반 인스턴스 대비 최대 40% 향상된 가격 대비 성능을 제공합니다.

| 세대 | 인스턴스 예시 | 특징 |
| --- | --- | --- |
| Graviton2 | m6g, c6g, r6g | 1세대 ARM, 검증된 안정성 |
| Graviton3 | m7g, c7g, r7g | 20% 향상된 네트워크 대역폭 |
| Graviton4 | m8g (신규) | 최신 세대, 최고 효율 |

:::warning
ARM 아키텍처이므로 애플리케이션이 x86 바이너리에 의존하는 경우 멀티 아키텍처 빌드 테스트 필요. Docker 이미지도 `arm64` 지원 여부 확인 필수.
:::

---

### 🔀 버스터블 인스턴스 (T 시리즈)

- 기준 CPU 성능 + 필요 시 일시적으로 버스트 가능
- **CPU 크레딧** 방식: 평소에 크레딧 적립 → 부하 시 소진
- 개발/테스트, 트래픽 변동이 큰 소규모 앱에 적합

```
t3.micro:  기준 20% CPU, 필요 시 최대 100%까지 버스트
t3.large:  기준 30% CPU, 필요 시 최대 100%까지 버스트
```
:::warning
프로덕션 고부하 환경에서 크레딧 고갈 시 심각한 성능 저하 발생. `unlimited` 모드 설정 시 추가 비용 발생.
:::
---

## 3. Security Groups (보안 그룹)

### 🔥 개념

- EC2 인스턴스 앞단의 **가상 방화벽**
- **Allow 규칙만** 존재 (Deny 규칙 없음 — NACL과 차이)
- IP 또는 **다른 Security Group**을 참조해 규칙 설정 가능

```
인터넷
  │
  ▼
[Security Group]  ← 여기서 허용된 트래픽만 통과
  │
  ▼
[EC2 Instance]   ← 차단된 트래픽은 인스턴스가 아예 볼 수 없음
```

---

### 📋 Inbound / Outbound 기본값

| 트래픽 방향 | 기본값 | 의미 |
| --- | --- | --- |
| **Inbound** | **전체 차단** | 명시적으로 허용하지 않으면 외부 접근 불가 |
| **Outbound** | **전체 허용** | 인스턴스에서 외부로 나가는 트래픽 제한 없음 |

---

### 📌 주요 포트 번호

| 포트 | 프로토콜 | 용도 |
| --- | --- | --- |
| **22** | SSH | Linux 인스턴스 원격 접속 |
| **22** | SFTP | SSH 기반 파일 전송 |
| **21** | FTP | 파일 전송 (보안 취약, 비권장) |
| **80** | HTTP | 웹 서버 (비암호화) |
| **443** | HTTPS | 웹 서버 (암호화) |
| **3389** | RDP | Windows 인스턴스 원격 데스크톱 |
| **3306** | MySQL/Aurora | DB 접속 |
| **5432** | PostgreSQL | DB 접속 |
| **6379** | Redis | 캐시 서버 접속 |

---

### ✅ Security Group 활용 팁

**1. Security Group 간 참조 (IP 대신)**

```
[App Server SG]  →  Inbound: MySQL(3306) from [DB Server SG]
```

→ IP 변경 시에도 자동 적용, IP 관리 불필요

**2. 트러블슈팅 가이드**

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| 앱에 아예 접근 안 됨 (timeout) | Security Group 차단 | Inbound 규칙 확인 |
| 접근은 되는데 `Connection refused` | 앱이 안 떴거나 포트 불일치 | 앱 상태 확인 |
| 특정 IP만 안 됨 | Inbound IP 범위 설정 오류 | CIDR 확인 |

:::important[💡 **Best Practice**]
SSH용 Security Group을 별도로 분리하여 관리. 0.0.0.0/0 (전체 허용)은 절대 프로덕션에 사용 금지.
:::
---

## 4. SSH 접속 방법

### 플랫폼별 지원 현황

| 방법 | Mac | Linux | Windows < 10 | Windows ≥ 10 |
| --- | --- | --- | --- | --- |
| **SSH** | ✅ | ✅ | ❌ | ✅ |
| **PuTTY** | ❌ | ❌ | ✅ | ✅ |
| **EC2 Instance Connect** | ✅ | ✅ | ✅ | ✅ |
| **AWS Systems Manager (SSM)** | ✅ | ✅ | ✅ | ✅ |

---

### SSH 접속 예시

```bash
# 키 파일 권한 설정 (최초 1회)
chmod 400 my-key.pem

# 접속
ssh -i my-key.pem ec2-user@<공인IP>
# Amazon Linux: ec2-user
# Ubuntu: ubuntu
# CentOS: centos
```

---

### 🌟 EC2 Instance Connect vs SSM Session Manager

| 구분 | EC2 Instance Connect | SSM Session Manager |
| --- | --- | --- |
| SSH 키 필요 | ❌ (브라우저 기반) | ❌ |
| 공인 IP 필요 | ✅ | **❌ (프라이빗 서브넷 가능)** |
| 포트 22 오픈 필요 | ✅ | **❌** |
| 감사 로그 | 제한적 | **CloudTrail + S3 완전 기록** |
| 보안 수준 | 보통 | **최고 (Best Practice)** |

**AWS Systems Manager Session Manager**

- SSH 포트를 열거나 Bastion 호스트를 사용하지 않고 EC2 인스턴스, 온프레미스 서버, VM 등을 엑세스 할 수 있는 안전하고(secure), 감사가 가능하며(audited), 브라우저 기반(browser-based) 쉘
- Benefits
    - 보안 (Security) : SSH 포트나 RDP (3389) 포트를 열지 않아도 되어서 높은 보안 수준 유지 가능
    - 접근 제어 (Access Control) : IAM 정책을 사용하여 사용자의 접근 제어 가능
    - 감사 추적 가능성 (Auditability) : 모든 세션 명령어는 로깅 → CloudWatch Logs가 Amazon S3 등에서 로그 중앙 관리 지원
    - SSH Key 관리 ❌ : SSH 키 관리, 저장 및 로테이션이 불필요
    - 크로스 플랫폼 (Cross-Platform) : Windows와 Linux 인스턴스 둘다 지원 가능
- How to setup?
    - SSM Agent 설치
    - IAM Role Configuration : `AmazonSSMManagedInstanceCore` 정책 설정을 통해서 System Manager service와 통신 가능하도록 허용
    - Network Connectivity : 인스턴스는 HTTPS (443) 아웃바운드 정책 허용
:::important[💡 **2025 Best Practice**]
포트 22를 아예 열지 않고 **SSM Session Manager**로 접속하는 것이 보안상 가장 권장되는 방식입니다.
:::
---

## 5. 구매 옵션 (Purchasing Options)

### 🏨 호텔 비유로 이해하기

| 구매 옵션 | 호텔 비유 | 특징 |
| --- | --- | --- |
| **On-Demand** | 예약 없이 정가로 체크인 | 유연, 단기, 가장 비쌈 |
| **Reserved** | 1~3년 장기 계약 할인 | 최대 72% 할인, 안정적 워크로드 |
| **Savings Plans** | 일정 금액 선불, 어느 방이든 OK | 유연한 할인 약정 |
| **Spot** | 빈 방 경매, 언제든 퇴실 가능 | 최대 90% 할인, 중단 가능 |
| **Dedicated Host** | 건물 전체 임대 | 물리 서버 단독 사용, 라이선스 |
| **Dedicated Instance** | 같은 건물 내 남과 공유 없음 | 논리적 격리 |
| **Capacity Reservation** | 방 예약만 해두기 (안 써도 요금) | 용량 보장, 특정 AZ |

---

### 💰 비용 비교 및 사용 시나리오

```
비용 (높음 → 낮음)
On-Demand > Dedicated Host > Reserved > Savings Plans > Spot

유연성 (높음 → 낮음)
On-Demand > Spot > Savings Plans > Reserved > Dedicated Host
```

### On-Demand

- Linux/Windows: **초 단위** 과금 (최초 1분 이후)
- 기타 OS: **시간** 단위 과금
- 예측 불가한 단기 워크로드, 신규 앱 테스트에 적합

### Reserved Instances (RI)

- 최대 **72% 할인** (3년 전액 선불 시 최대)
- 고정 조건: 인스턴스 타입, 리전, OS, 테넌시

| 결제 방식 | 할인율 |
| --- | --- |
| No Upfront (선불 없음) | 낮음 |
| Partial Upfront (일부 선불) | 중간 |
| All Upfront (전액 선불) | **최대** |
- **Convertible RI**: 인스턴스 타입 변경 가능, 최대 **66% 할인**
- Reserved Instance Marketplace에서 남은 기간 판매 가능

### Savings Plans

- 시간당 사용 금액 약정 (`$X/시간` 형태)
- RI보다 유연: 인스턴스 크기, OS, 테넌시 변경 가능
- 약정 초과분은 On-Demand 요금 적용
- **취소 불가** (1년 또는 3년 약정)

> 대부분의 팀에게는 Compute Savings Plans가 절감액과 유연성의 가장 좋은 균형을 제공합니다. AWS는 새로운 약정에 대해 Savings Plans를 권장합니다.
> 

### Spot Instances

- 최대 **90% 할인** — AWS에서 가장 저렴
- 현재 Spot 가격이 설정한 최대 가격 초과 시 **2분 경고 후 인스턴스 종료**
- 중단에 견딜 수 있는 워크로드에만 적합:

```
✅ 적합: 배치 처리, 데이터 분석, 이미지 처리, ML 학습, CI/CD
❌ 부적합: 운영 DB, 결제 시스템, 실시간 서비스
```

### Dedicated Hosts vs Dedicated Instances

| 항목 | Dedicated Instance | Dedicated Host |
| --- | --- | --- |
| 물리 서버 전용 사용 | ❌ (계정 내 공유 가능) | ✅ |
| 소켓/코어 수 가시성 | ❌ | ✅ |
| 인스턴스 배치 제어 | ❌ | ✅ |
| 기존 라이선스 활용 (BYOL) | ❌ | ✅ |
| 비용 | 비쌈 | 가장 비쌈 |

> 💡 **Dedicated Host 사용 시나리오**: Oracle, Microsoft SQL Server 등 소켓/코어 기반 라이선스가 있는 소프트웨어를 클라우드로 이전(Lift & Shift)할 때.
> 

---

## 6. Spot Instance 심화

### 🎯 Spot Instance 작동 방식

```
사용자가 max price 설정
        │
        ▼
현재 Spot 가격 < max price → 인스턴스 실행 유지
        │
현재 Spot 가격 > max price → 2분 경고
        │
        ├── Stop: Spot 가격이 낮아지면 자동 재시작 (Persistent)
        └── Terminate: 완전 종료 (One-time)
```

---

### 📋 Spot Request 유형

| 유형 | 설명 | 종료 후 |
| --- | --- | --- |
| **One-time** | 한 번만 요청, 종료 시 끝 | 재시작 없음 |
| **Persistent** | 가격 조건 충족 시 자동 재시작 | 자동 재실행 |

**올바른 종료 순서:**

```
1️⃣ Spot Request 취소 (Cancel) — 먼저!
2️⃣ 연결된 인스턴스 종료 (Terminate)

⚠️ 인스턴스만 종료하면 Persistent 요청이 새 인스턴스를 다시 시작시킴
```

---

### 🚢 Spot Fleet — 더 스마트한 Spot 활용

Spot Fleet = **Spot 인스턴스 집합** + (선택) On-Demand 인스턴스

```
Spot Fleet
├── Launch Pool A: c5.xlarge, Linux, us-east-1a
├── Launch Pool B: c5.2xlarge, Linux, us-east-1b
└── Launch Pool C: c5a.xlarge, Linux, us-east-1c

→ 목표 용량에 맞춰 가장 유리한 풀에서 자동으로 인스턴스 요청
```

**Spot Fleet 할당 전략:**

| 전략 | 설명 | 적합한 상황 |
| --- | --- | --- |
| `lowestPrice` | 가장 저렴한 풀 우선 | 비용 최우선 단기 작업 |
| `diversified` | 여러 풀에 분산 | 가용성 중시, 장기 실행 |
| `capacityOptimized` | 용량 여유 있는 풀 우선 | 중단 최소화 |
| `priceCapacityOptimized` ⭐ | 용량 여유 풀 중 최저가 | **대부분의 워크로드에 권장** |

---

## 7. IP 주소 & Elastic IP

### 🌐 공인 IP (Public IP) vs 사설 IP (Private IP)

| 구분 | 공인 IP (Public IP) | 사설 IP (Private IP) |
| --- | --- | --- |
| 식별 범위 | 인터넷 전체 | 해당 사설 네트워크 내부만 |
| 유일성 | **전 세계에서 유일** | 같은 네트워크 내에서만 유일 |
| 중복 | 불가 | 서로 다른 회사 간에는 중복 가능 |
| 외부 접근 | 직접 가능 | NAT + 인터넷 게이트웨이 경유 필요 |
| 위치 추적 | 가능 (Geo-location) | 불가 |

```
[사설 네트워크 A]          [사설 네트워크 B]
  192.168.1.10               192.168.1.10   ← 사설 IP 중복 가능
       │                          │
   [NAT/IGW]                  [NAT/IGW]
       │                          │
       └──────── 인터넷 ──────────┘
           (공인 IP는 유일)
```
:::tip
EC2 인스턴스는 기본적으로 사설 IP는 고정이지만, **공인 IP는 인스턴스를 중지(Stop) 후 재시작(Start)하면 변경됩니다.**
:::
---

### 🔒 IPv4 vs IPv6

| 항목 | IPv4 | IPv6 |
| --- | --- | --- |
| 예시 | `1.160.10.240` | `3ffe:1900:4545:3:200:f8ff:fe21:67cf` |
| 주소 수 | 약 37억 개 (고갈 중) | 사실상 무제한 |
| 현재 상태 | 인터넷의 주류 방식 | IoT 등 신규 기기 확산으로 증가 중 |

---

### 📌 Elastic IP (탄력적 IP)

- EC2 인스턴스를 **중지(Stop) 후 시작(Start)하면 공인 IP가 바뀌는 문제**를 해결하는 고정 공인 IPv4 주소
- 삭제하지 않는 한 계속 소유 가능
- 한 번에 **하나의 인스턴스에만** 연결 가능
- 인스턴스 또는 네트워크 인터페이스 장애 시, 다른 인스턴스로 빠르게 재매핑(Remap)하여 장애를 마스킹(Masking)하는 용도로 활용 가능
- 계정당 기본 **5개 제한** (AWS에 증가 요청 가능)
- **연결되지 않은 Elastic IP에는 요금 부과** (낭비 방지용)

---

### ⚠️ Elastic IP 사용을 피해야 하는 이유

> Elastic IP를 사용하는 아키텍처는 종종 설계가 잘못된 신호입니다 (often reflect poor architectural decisions).
> 

**대신 권장하는 방법:**

| 상황 | 권장 방법 |
| --- | --- |
| 고정 주소가 필요한 경우 | **DNS 이름(Route 53)을 공인 IP에 연결** |
| 외부 트래픽 처리 | **로드 밸런서(ELB) 사용** → 인스턴스에 공인 IP 불필요 |
| 장애 복구 | **Auto Scaling + ELB** 조합으로 IP 의존성 제거 |

---

## 8. 배치 그룹 (Placement Groups)

- EC2 인스턴스가 AWS 인프라 내에 **어떻게 배치될지를 제어**하는 기능
- 세 가지 전략 중 하나를 선택하여 그룹 생성

---

### 📊 배치 전략 비교

| 전략 | 핵심 목표 | 제약 | 주요 Use Case |
| --- | --- | --- | --- |
| **Cluster** | 초저지연, 고대역폭 | 단일 AZ, 단일 랙 | HPC, 빅데이터, 초고속 통신 |
| **Spread** | 최대 가용성, 물리적 격리 | AZ당 최대 7 인스턴스 | 미션 크리티컬, 고가용성 앱 |
| **Partition** | 대규모 분산, 파티션 단위 격리 | AZ당 최대 7 파티션, 수백 인스턴스 | 분산 데이터 시스템 (Hadoop, Kafka) |

---

### 🚀 Cluster Placement Group (클러스터 배치 그룹)

```
┌─────────── Availability Zone ───────────┐
│  ┌─────────── 단일 랙 ─────────────┐   │
│  │  [EC2] [EC2] [EC2] [EC2] [EC2]  │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

- **장점**: Enhanced Networking (향상된 네트워킹) 활성화 시 인스턴스 간 10Gbps 대역폭
- **단점**: AZ 전체 장애 시 **모든 인스턴스가 동시에 중단**됨
- **Use Case**: 빠르게 완료해야 하는 빅데이터 작업 (Big Data job), 초저지연·고처리량이 필요한 애플리케이션

---

### 🛡️ Spread Placement Group (분산 배치 그룹)

```
┌── AZ 1 ──┐   ┌── AZ 2 ──┐   ┌── AZ 3 ──┐
│ [랙1]    │   │ [랙2]    │   │ [랙3]    │
│  [EC2]   │   │  [EC2]   │   │  [EC2]   │
└──────────┘   └──────────┘   └──────────┘
```

- **장점**: 각 인스턴스가 서로 다른 물리 하드웨어(랙)에 배치 → 동시 장애 위험 최소화, 여러 AZ에 걸쳐 배치 가능
- **단점**: **AZ당 최대 7개 인스턴스** 제한 (소규모 배포에만 적합)
- **Use Case**: 각 인스턴스가 서로 독립적으로 장애 격리(Isolated from failure)되어야 하는 고가용성 미션 크리티컬 애플리케이션

---

### 🗂️ Partition Placement Group (파티션 배치 그룹)

```
┌──────────────── AZ 1 ──────────────────┐
│  ┌─파티션1─┐  ┌─파티션2─┐  ┌─파티션3─┐ │
│  │ [EC2]  │  │ [EC2]  │  │ [EC2]  │ │
│  │ [EC2]  │  │ [EC2]  │  │ [EC2]  │ │
│  │ [EC2]  │  │ [EC2]  │  │ [EC2]  │ │
│  └────────┘  └────────┘  └────────┘ │
│   (서로 다른 랙, 파티션 간 랙 공유 없음)     │
└────────────────────────────────────────┘
```

- AZ당 최대 **7개 파티션**, 동일 리전 내 여러 AZ에 걸쳐 배치 가능
- 그룹당 수백 개의 EC2 인스턴스 지원
- **파티션 간 랙을 절대 공유하지 않음** → 한 파티션 장애가 다른 파티션에 영향 없음
- 인스턴스 메타데이터(Instance Metadata)로 자신이 속한 파티션 정보 접근 가능 → 애플리케이션이 토폴로지 인식(Topology-aware) 배포 가능
- **Use Case**: HDFS, HBase, Cassandra, Kafka 등 대규모 분산 데이터 시스템

---

## 9. 탄력적 네트워크 인터페이스 (ENI)

### 🔌 ENI란?

- **ENI (Elastic Network Interface, 탄력적 네트워크 인터페이스)**: VPC (Virtual Private Cloud, 가상 사설 클라우드) 내에서 가상 네트워크 카드를 나타내는 논리적 컴포넌트
- EBS 볼륨을 필요에 따라 EC2에 붙이듯, 네트워킹 계층도 인스턴스와 독립적으로 분리하여 별도로 관리 가능
- **특정 AZ (Availability Zone, 가용 영역)에 종속**됨 — AZ 간 이동 불가

> 💡 AWS 공식 블로그 핵심 인사이트: ENI 도입 이전에는 EC2 인스턴스가 특정 서브넷(Subnet)에 속하는 개념이었으나, ENI 도입 이후 **서브넷 귀속 단위는 인스턴스가 아닌 ENI** 로 바뀌었습니다. 따라서 하나의 인스턴스에 서로 다른 서브넷의 ENI 두 개를 붙여 듀얼 홈(Dual-homed) 환경을 구성할 수 있습니다.

---

### 📋 ENI 속성 (Attributes)

| 속성 | 설명 |
| --- | --- |
| **기본 사설 IPv4 (Primary Private IPv4)** | 반드시 1개, 서브넷 대역 내 할당 |
| **보조 사설 IPv4 (Secondary Private IPv4)** | 1개 이상 추가 가능 |
| **탄력적 IP (Elastic IP)** | 사설 IPv4당 최대 1개 연결 가능 |
| **공인 IPv4 (Public IPv4)** | 1개 |
| **보안 그룹 (Security Groups)** | 1개 이상 연결 가능 |
| **MAC 주소 (MAC Address)** | 고유 식별자, BYOL 라이선스에 활용 |
| **소스/목적지 확인 플래그 (Source/Destination Check Flag)** | NAT·프록시 서버 구성 시 비활성화 필요 |
| **종료 시 삭제 플래그 (Delete on Termination Flag)** | 기본 ENI는 인스턴스 종료 시 자동 삭제 |

---

### 🔄 ENI의 독립적 생명주기 (Independent Lifetime)

EBS 볼륨처럼 ENI도 **특정 EC2 인스턴스와 독립적인 생명주기**를 가집니다.

```
ENI 생성 (미리 만들어 놓기 가능)
    │
    ├── 인스턴스 시작 시 연결 (Launch-time attach)
    ├── 실행 중인 인스턴스에 즉시 연결 (Hot attach, 핫 어태치)
    └── 인스턴스 종료 후에도 ENI는 유지 (Delete on Termination = false 시)
```

---

### 🏗️ ENI 주요 활용 패턴

**1. 관리 네트워크 분리 (Management Network / Backnet)**

```
EC2 인스턴스
├── ENI 1 (공용 서브넷) ─→ 인터넷 게이트웨이 → 외부 트래픽 처리 (포트 80/443)
└── ENI 2 (사설 서브넷) ─→ VPN 게이트웨이 → SSH 접속·로그·관리 트래픽 (포트 22)
```

각 ENI에 다른 보안 그룹을 적용해 트래픽을 세밀하게 제어 가능.

**2. 멀티 인터페이스 애플리케이션 (Multi-Interface Applications)**

로드 밸런서(Load Balancer), 프록시 서버(Proxy Server), NAT 서버 구성 시 두 서브넷 사이에서 트래픽을 전달. 이 경우 **소스/목적지 확인(Source/Destination Check)을 비활성화**해야 함.

**3. MAC 주소 기반 라이선스 (MAC-Based Licensing)**

특정 MAC 주소에 묶인 상용 소프트웨어 라이선스를 사용할 때, 인스턴스가 교체·타입 변경이 필요한 경우에도 동일한 ENI(MAC 주소 유지)를 새 인스턴스에 재연결하여 라이선스 재발급 없이 계속 사용 가능.

**4. 저비용 고가용성 (Low-Budget High Availability)**

```
1️⃣ ENI를 인스턴스 A에 연결
2️⃣ 인스턴스 A 장애 발생
3️⃣ 새 인스턴스 B 시작
4️⃣ ENI를 인스턴스 B에 재연결 → 수 초 내 트래픽 복구
```

IP 주소와 보안 그룹 설정이 ENI에 종속되어 있으므로, 인스턴스가 바뀌어도 네트워크 구성이 그대로 유지됨.

---

### 🔁 장애 복구 시나리오 (Failover)

```
[ENI: 10.0.1.50]──→ [인스턴스 A: 정상]
                             │ 장애!
                             ▼
[ENI: 10.0.1.50]──→ [인스턴스 B: 교체됨]
  (동일 IP 유지)      (수 초 내 트래픽 전환)
```
:::tip
ENI는 특정 AZ에 종속(Bound to a specific AZ)됩니다. AZ 간 이동은 불가하므로, 멀티 AZ 고가용성이 필요한 경우 ELB(Elastic Load Balancer)를 사용해야 합니다.
:::
---

## 10. EC2 최대 절전 모드 (Hibernate)

### 💤 인스턴스 상태 전환 비교

| 상태 | EBS 데이터 | RAM 상태 | 다음 시작 시 |
| --- | --- | --- | --- |
| **Stop (중지)** | ✅ 유지 | ❌ 소멸 | OS 부팅 + 앱 초기화 필요 |
| **Terminate (종료)** | ❌ 소멸 (루트 볼륨 기본 삭제) | ❌ 소멸 | 새 인스턴스 시작 필요 |
| **Hibernate (최대 절전)** | ✅ 유지 | ✅ **EBS에 저장 후 복원** | 훨씬 빠른 재시작 |

---

### 🔧 Hibernate 작동 원리

```
Hibernate 실행
    │
    ▼
RAM 전체 내용을 루트 EBS 볼륨 파일에 덤프(Dump)
    │
    ▼
인스턴스 중지 (OS가 완전히 종료되지 않음)
    │
    ▼
재시작 시 → EBS에서 RAM 상태 복원 → 중단 지점부터 즉시 재개
```
:::important
OS 재부팅(OS booting)이 없으므로 시작 시간이 매우 빠름. 애플리케이션이 캐시 워밍(Cache Warming)을 다시 할 필요 없음.
:::
---

### ✅ 지원 조건 (Requirements)

| 항목 | 조건 |
| --- | --- |
| **인스턴스 패밀리 (Instance Family)** | C3, C4, C5, I3, M3, M4, R3, R4, T2, T3 등 |
| **RAM 크기 (RAM Size)** | **150 GB 미만** |
| **인스턴스 크기 (Instance Size)** | 베어 메탈(Bare Metal) 인스턴스 미지원 |
| **AMI** | Amazon Linux 2, Linux AMI, Ubuntu, Windows 등 |
| **루트 볼륨 (Root Volume)** | 반드시 **EBS**, **암호화(Encrypted)** 필수, Instance Store 불가, RAM을 담을 충분한 용량 필요 |
| **구매 옵션** | On-Demand, Reserved, Spot 인스턴스 모두 지원 |
| **최대 절전 유지 기간** | **60일을 초과할 수 없음 (cannot be hibernated more than 60 days)** |

---

### 💡 Hibernate 주요 Use Case

| Use Case | 설명 |
| --- | --- |
| **장시간 실행 프로세스 (Long-running processing)** | 재시작 없이 작업 상태 그대로 유지 |
| **RAM 상태 보존 (Saving the RAM state)** | 인메모리 데이터·캐시 유지 |
| **초기화가 오래 걸리는 서비스 (Services that take time to initialize)** | 부팅·캐시 워밍 비용 없이 즉시 재개 |

:::warning
⚠️ **루트 볼륨 암호화가 필수**인 이유: RAM 내용(민감한 데이터 포함 가능)이 EBS 디스크에 평문으로 쓰이는 것을 방지하기 위함.
:::

---

## 11. Best Practices

### 💡 인스턴스 선택

**1. 워크로드 특성 먼저 파악**

```
CPU 집약적  → Compute Optimized (c 계열)
메모리 집약적 → Memory Optimized (r, x 계열)
스토리지 I/O  → Storage Optimized (i, d 계열)
균형 잡힌 일반 → General Purpose (m, t 계열)
AI/ML 학습   → GPU (p, g 계열) 또는 Trn1
AI/ML 추론   → Inf2 (가장 저렴한 추론 전용)
```

**2. Graviton(ARM) 우선 검토**

Graviton(arm64)은 x86 대비 최대 40% 향상된 가격 대비 성능을 제공합니다. 멀티 아키텍처 빌드를 테스트하고 롤아웃 전에 p95/p99 성능을 측정하세요.

---

### 💰 비용 최적화 전략

**1. Right-Sizing (적정 규모 조정)**

2~4주 동안 활용도 데이터를 수집하고, AWS Compute Optimizer 또는 Cost Explorer를 통해 과소 활용 인스턴스를 파악한 뒤 적절한 크기로 조정하세요.

```bash
# AWS Compute Optimizer로 우선 분석
# → CloudWatch 지표 기반 권장 인스턴스 타입 제안
```

**2. 구매 옵션 혼합 (Blended Pricing)**

```
안정적 기본 부하  → Reserved Instances 또는 Savings Plans (72% 절감)
변동적 추가 부하  → Spot Instances (최대 90% 절감)
예측 불가 피크   → On-Demand (백업)
```

> 기준 부하에는 Savings Plans/Reserved Instance로 커버하고, 기준 초과의 변동 부하에는 Spot Instance를 사용하세요. Spot 용량 부족 시에는 중요 워크로드를 위해 On-Demand로 자동 폴백하도록 구성하세요.
> 

**3. 스케줄링으로 유휴 비용 절감**

```
개발/테스트 환경: 업무 시간(09:00-19:00)에만 실행
→ 태그 기반 자동 시작/종료 → 최대 65% 비용 절감

권장 태그 체계:
  Environment = dev | staging | prod
  Schedule    = OfficeHours | AlwaysOn
  Owner       = team-name
  CostCenter  = CC-1234
```

**4. EBS 볼륨 비용 최적화**

CloudWatch 또는 Compute Optimizer를 통해 일관되게 낮은 IOPS 또는 처리량을 보이는 볼륨을 식별하고, 더 저렴한 gp3 또는 st1 볼륨 타입으로 다운그레이드를 검토하세요.

| 볼륨 타입 | 특징 | 적합한 용도 |
| --- | --- | --- |
| `gp3` | 비용 효율 높음 ⭐ | 대부분의 워크로드 |
| `gp2` | 구형, gp3보다 비쌈 | 마이그레이션 고려 |
| `io2` | 고IOPS, 고비용 | 미션 크리티컬 DB |
| `st1` | 저비용, 순차 읽기 | 로그, 빅데이터 |
| `sc1` | 가장 저렴 | 콜드 데이터 |

:::tip
gp2 → gp3 전환 시 동일 성능에 약 20% 비용 절감 가능.
:::
---

### 🔐 EC2 보안 Best Practice

**1. 인스턴스 접근**

- 포트 22(SSH)를 0.0.0.0/0으로 여는 것 **절대 금지**
- 가능하면 **SSM Session Manager** 사용 (포트 22 불필요)
- 불가피한 경우 회사 IP 또는 VPN IP만 허용

**2. IAM Role 사용**

```python
# ❌ 절대 금지: Access Key를 EC2에 직접 저장
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."

# ✅ 권장: EC2 IAM Role 사용 (자동으로 임시 자격증명 관리)
# 인스턴스에 Role만 붙이면 코드에서 자격증명 불필요
import boto3
s3 = boto3.client('s3')  # Role이 자동으로 처리
```

**3. Auto Scaling 활용**

```
Auto Scaling Group
├── Minimum: 2 (최소 가용성 보장)
├── Desired: 4 (평소 운영 수량)
└── Maximum: 10 (트래픽 급증 대비)

→ CloudWatch 지표(CPU 70% 초과 시) 기반 자동 Scale Out
→ CloudWatch 지표(CPU 30% 미만 시) 기반 자동 Scale In
```

---

### 🚨 자주 하는 실수 (Anti-Patterns)

| 실수 | 문제 | 해결책 |
| --- | --- | --- |
| 인스턴스 타입 고려 없이 m5.xlarge 고정 | 비용 낭비 또는 성능 부족 | Compute Optimizer로 분석 후 선택 |
| 모든 워크로드에 On-Demand | 불필요한 비용 | RI + Savings Plans + Spot 혼합 |
| Security Group에서 0.0.0.0/0:22 허용 | 무차별 대입 공격 노출 | SSM 또는 특정 IP만 허용 |
| 개발 환경 24시간 가동 | 불필요한 비용 | 스케줄 기반 자동 중지 |
| Access Key를 EC2 내 파일로 저장 | 키 탈취 위험 | IAM Role로 대체 |
| 인스턴스 스토어에 중요 데이터 저장 | 종료 시 데이터 소멸 | EBS 또는 S3에 저장 |
| 구버전 인스턴스 타입 방치 | 가격 대비 성능 낮음 | 신세대로 마이그레이션 |

---

## 12. 핵심 요약

```
EC2 구성 핵심
├── 인스턴스 타입: 워크로드에 맞게 선택 (m=범용, c=컴퓨팅, r=메모리, i=스토리지)
├── User Data: 최초 1회 부트스트랩 (root 권한)
├── Security Group: 상태 기반 가상 방화벽 (Allow만, 기본 Inbound 차단)
└── 접속: SSH / EC2 Instance Connect / SSM Session Manager

IP 주소
├── Public IP: 인터넷에서 식별, 인스턴스 재시작 시 변경됨
├── Private IP: 내부 네트워크 전용, 재시작해도 고정
└── Elastic IP: 고정 공인 IP, 5개 제한, 미연결 시 과금

배치 그룹 (Placement Groups)
├── Cluster:   단일 AZ, 단일 랙 → 초저지연·고대역폭 (10Gbps), 동시 장애 위험
├── Spread:    서로 다른 랙 → 최대 가용성, AZ당 7개 제한
└── Partition: 파티션 단위 격리 → 분산 시스템 (Hadoop, Kafka, Cassandra)

ENI (Elastic Network Interface)
├── VPC 내 가상 네트워크 카드 (논리적 컴포넌트)
├── 특정 AZ에 종속, AZ 간 이동 불가
├── 인스턴스와 독립적 생명주기 (Hot attach 가능)
└── 활용: 관리망 분리, MAC 라이선스, 저비용 고가용성 Failover

Hibernate (최대 절전 모드)
├── RAM 상태를 암호화된 EBS에 저장 → 빠른 재시작
├── 루트 EBS 암호화(Encrypted) 필수
├── RAM 150GB 미만, 최대 60일 제한
└── 구매 옵션: On-Demand / Reserved / Spot 모두 지원

구매 옵션 선택 가이드
├── 단기·불규칙  → On-Demand
├── 안정적 장기  → Reserved Instances 또는 Savings Plans (72% 절감)
├── 중단 허용   → Spot Instances (90% 절감)
├── 라이선스 제약 → Dedicated Host
└── 용량 보장   → Capacity Reservation

비용 최적화 3단계
1️⃣ Right-Sizing: Compute Optimizer로 적정 규모 분석
2️⃣ 구매 혼합: 기본 부하 → RI/SP, 변동 부하 → Spot
3️⃣ 스케줄링: 개발 환경 업무시간만 가동
```

---

## 📚 참고 자료

- [EC2 인스턴스 타입 전체 목록](https://aws.amazon.com/ec2/instance-types/)
- [EC2 가격 계산기](https://calculator.aws/pricing/2/home)
- [AWS Compute Optimizer](https://aws.amazon.com/compute-optimizer/)
- [EC2 Spot Instance 공식 문서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html)
- [EC2 비용 및 용량 최적화](https://aws.amazon.com/ec2/cost-and-capacity/)
- [Savings Plans vs Reserved Instances 비교](https://aws.amazon.com/savingsplans/faq/)
- [ENI 공식 AWS 블로그 (Jeff Barr)](https://aws.amazon.com/blogs/aws/new-elastic-network-interfaces-in-the-virtual-private-cloud/)
- [Placement Groups 공식 문서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)
- [EC2 Hibernate 공식 문서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Hibernate.html)