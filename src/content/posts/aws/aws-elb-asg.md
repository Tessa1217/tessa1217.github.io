---
title: ⚖️ AWS ELB (Elastic Load Balancer) & ASG (Auto Scaling Group)
published: 2026-03-18
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# ⚖️ AWS ELB (Elastic Load Balancer) & ASG (Auto Scaling Group)

> 고가용성(High Availability)과 확장성(Scalability)의 핵심 서비스
> 
> 
> ELB는 트래픽을 분산하고, ASG는 인스턴스 수를 자동으로 조절합니다.
---
## 목차
1. [확장성 & 고가용성 개념](#1-확장성--고가용성-개념)
2. [ELB 개요](#2-elb-개요)
3. [로드 밸런서 유형 비교](#3-로드-밸런서-유형-비교)
4. [ALB (Application Load Balancer)](#4-alb-application-load-balancer)
5. [NLB (Network Load Balancer)](#5-nlb-network-load-balancer)
6. [GWLB (Gateway Load Balancer)](#6-gwlb-gateway-load-balancer)
7. [ELB 주요 기능](#7-elb-주요-기능)
8. [ASG (Auto Scaling Group)](#8-asg-auto-scaling-group)
9. [ASG 스케일링 정책](#9-asg-스케일링-정책)
10. [핵심 요약](#10-핵심-요약)
11. [📌 시험 자주 출제 포인트](#-시험-자주-출제-포인트)
12. [📚 참고 자료](#-참고-자료)
---

## 1. 확장성 & 고가용성 개념

### 📈 스케일링 방향

| 방향 | 영어 표현 | 방법 | 예시 |
| --- | --- | --- | --- |
| **수직 확장 (Scale Up/Down)** | Vertical Scalability | 인스턴스 크기 증가 | t2.micro → t2.large |
| **수평 확장 (Scale Out/In)** | Horizontal Scalability (= Elasticity) | 인스턴스 수 증가/감소 | ASG, 로드 밸런서 |

```
수직 확장 (Scale Up):   [작은 서버]  →  [큰 서버]        ← 한계 있음 (하드웨어 상한)
수평 확장 (Scale Out):  [서버1]  →  [서버1][서버2][서버3]  ← 이론상 무제한
```

- **수직 확장**: RDS, ElastiCache 같은 **비분산 시스템(Non-distributed Systems)**에 일반적
- **수평 확장**: 웹 애플리케이션, 현대 분산 시스템에 일반적

---

### 🏢 고가용성 (High Availability)

- 애플리케이션을 **최소 2개의 데이터 센터(= AZ)**에서 운영
- 목표: 하나의 데이터 센터 장애 시에도 서비스 지속 (Survive a data center loss)
- **수평 확장과 함께 설계**되는 것이 일반적

```
EC2 HA 구성 예시:
    [Load Balancer]
    /             \
[EC2 - AZ1]  [EC2 - AZ2]   ← 한 AZ 장애 시 다른 AZ가 트래픽 처리
```

---

## 2. ELB 개요

### 🔀 로드 밸런서란?

- 여러 다운스트림 서버(EC2 등)에 **트래픽을 분산**하는 서버
- 애플리케이션에 **단일 접근 지점(Single Point of Access, DNS)** 제공

### 왜 ELB(관리형)를 쓰는가?

| 직접 구성 | AWS ELB (관리형) |
| --- | --- |
| 초기 비용 낮음 | 비용이 있음 |
| 유지보수·패치 직접 | **AWS가 관리** (업그레이드, 유지보수, HA 보장) |
| 설정 복잡 | 설정 간단 (Configuration knobs 제공) |
| AWS 서비스 통합 어려움 | **EC2, ASG, ECS, ACM, CloudWatch, Route 53, WAF 등과 통합** |

### ELB의 주요 역할

- 다운스트림 인스턴스 장애 시 seamless하게 처리 (Health Check)
- SSL 종료(SSL Termination, HTTPS 처리)
- 쿠키 기반 고정성(Stickiness)
- AZ 간 고가용성
- 퍼블릭/프라이빗 트래픽 분리

---

### ❤️ Health Check (헬스 체크)

- LB가 **인스턴스 정상 여부를 지속적으로 확인**하는 기능
- 특정 포트와 경로(Route)에 요청 → 200 OK면 정상, 아니면 비정상
- 비정상 인스턴스로는 **트래픽을 전송하지 않음**

```
LB → GET /health (포트 80)
       ↓
   200 OK → 정상 → 트래픽 전달
   기타   → 비정상 → 트래픽 차단
```

---

## 3. 로드 밸런서 유형 비교

| 유형 | 출시 | 레이어 | 프로토콜 | 주요 특징 |
| --- | --- | --- | --- | --- |
| **CLB** (Classic) | 2009 | L4/L7 | HTTP, HTTPS, TCP, SSL | 구세대, Deprecated |
| **ALB** (Application) | 2016 | **L7** | HTTP, HTTPS, WebSocket | 경로/호스트/쿼리 기반 라우팅 |
| **NLB** (Network) | 2017 | **L4** | TCP, TLS, UDP | 초고성능, 정적 IP |
| **GWLB** (Gateway) | 2020 | **L3** | IP (GENEVE 6081) | 3rd Party 가상 어플라이언스 |

:::tip
💡 **권장**: 항상 신세대(ALB, NLB, GWLB) 사용. CLB는 Deprecated.
:::

---

## 4. ALB (Application Load Balancer)

### 🌐 특징

- **레이어 7 (Layer 7, HTTP 레이어)** 전용
- HTTP/2, WebSocket 지원
- HTTP → HTTPS 리다이렉트 지원
- **고정 호스트명(Fixed Hostname)** 제공
- 클라이언트 IP는 인스턴스에 직접 전달되지 않음 → **X-Forwarded-For 헤더**로 확인

```
클라이언트 IP: 1.2.3.4
  │
  ▼
[ALB]  ──→  EC2: X-Forwarded-For: 1.2.3.4
               X-Forwarded-Port: 443
               X-Forwarded-Proto: https
```

---

### 🗺️ 라우팅 규칙 (Routing Rules)

| 라우팅 기준 | 예시 |
| --- | --- |
| **URL 경로 (Path)** | `/users` → 서비스 A, `/posts` → 서비스 B |
| **호스트명 (Hostname)** | `api.example.com` → API 서버, `web.example.com` → 웹 서버 |
| **쿼리스트링 (Query String)** | `?platform=mobile` → 모바일 서버 |
| **HTTP 헤더 (Headers)** | 특정 헤더 값에 따라 분기 |

> 💡 **마이크로서비스와 컨테이너에 최적**: ECS와 연동 시 동적 포트 매핑(Dynamic Port Mapping) 지원.
> 

---

### 🎯 Target Groups (대상 그룹)

| 대상 유형 | 설명 |
| --- | --- |
| **EC2 Instances** | Auto Scaling Group으로 관리 가능 |
| **ECS Tasks** | ECS 자체 관리 |
| **Lambda Functions** | HTTP 요청 → JSON 이벤트로 변환 |
| **IP Addresses** | 반드시 프라이빗 IP |
- Health Check는 **Target Group 레벨**에서 수행
- ALB는 **여러 Target Group**으로 동시 라우팅 가능

---

## 5. NLB (Network Load Balancer)

### ⚡ 특징

- **레이어 4 (Layer 4, 전송 계층)** — TCP & UDP 트래픽 처리
- **초고성능**: 초당 수백만 요청 처리, 초저지연(Ultra-low Latency)
- **AZ당 정적 IP(Static IP) 1개** 보유 → Elastic IP 할당 가능 (특정 IP 화이트리스팅에 유리)
- 극한 성능, TCP/UDP 트래픽 처리에 사용

### 🎯 Target Groups

- EC2 Instances
- IP Addresses (반드시 프라이빗 IP)
- **Application Load Balancer** (NLB 뒤에 ALB 배치 가능)
- Health Check: **TCP, HTTP, HTTPS** 프로토콜 지원

```
[정적 IP: 1.2.3.4]        ← 클라이언트가 고정 IP로 접근 (방화벽 화이트리스팅)
     [NLB]
       │
   [ALB / EC2]
```

---

## 6. GWLB (Gateway Load Balancer)

### 🛡️ 특징

- **레이어 3 (Layer 3, 네트워크 계층)** — IP 패킷 수준 처리
- 3rd Party 네트워크 가상 어플라이언스 배포·확장·관리
- **GENEVE 프로토콜 (포트 6081)** 사용

### 두 가지 역할

| 역할 | 설명 |
| --- | --- |
| **Transparent Network Gateway** (투명 네트워크 게이트웨이) | 모든 트래픽의 단일 진입/출구점 |
| **Load Balancer** | 가상 어플라이언스들에 트래픽 분산 |

```
인터넷 트래픽
     │
     ▼
  [GWLB]  ← 모든 트래픽이 먼저 여기를 통과
     │
[방화벽/IDS/IPS 어플라이언스 Fleet]
     │
     ▼
  [EC2 앱]
```

**주요 Use Case**: 방화벽(Firewalls), 침입탐지/방지 시스템(IDS/IPS), 딥 패킷 검사(Deep Packet Inspection), 페이로드 조작

---

## 7. ELB 주요 기능

### 🍪 Sticky Sessions (세션 고정, Session Affinity)

- 동일 클라이언트가 **항상 동일한 인스턴스**로 라우팅되도록 설정
- **CLB, ALB, NLB** 지원 (NLB는 쿠키 없이 동작)
- **Use Case**: 사용자 세션 데이터(로그인 정보 등)를 인스턴스 로컬에 저장하는 앱

:::warning
⚠️ **단점**: 트래픽 불균형 발생 가능 (일부 인스턴스에 부하 집중)
:::

**쿠키 유형:**

| 쿠키 종류 | 생성 주체 | 쿠키 이름 | 특징 |
| --- | --- | --- | --- |
| Custom Cookie | 애플리케이션(Target) | 직접 지정 | 커스텀 속성 포함 가능, AWSALB/AWSALBAPP/AWSALBTG 이름 사용 불가 |
| Application Cookie | LB | `AWSALBAPP` | ALB 생성 |
| Duration-based Cookie | LB | `AWSALB` (ALB), `AWSELB` (CLB) | 만료 시간 기반 |

---

### 🌍 Cross-Zone Load Balancing (교차 AZ 로드 밸런싱)

|  | 활성화 시 | 비활성화 시 |
| --- | --- | --- |
| **동작** | 모든 AZ의 인스턴스에 **균등 분산** | 같은 AZ의 인스턴스에만 분산 |

```
활성화:  AZ1(10대) + AZ2(2대) → 전체 12대에 균등 분배 (각 8.3%)
비활성화: AZ1(10대) → AZ1 10대만, AZ2(2대) → AZ2 2대만 (각 AZ 50%)
```

| LB 유형 | 기본값 | 교차 AZ 데이터 비용 |
| --- | --- | --- |
| **ALB** | **활성화 (기본)** | 없음 (No charges) |
| **NLB / GWLB** | **비활성화 (기본)** | 활성화 시 데이터 요금 부과 |
| **CLB** | 비활성화 (기본) | 없음 |

---

### 🔒 SSL/TLS 인증서

- **SSL (Secure Sockets Layer)** / **TLS (Transport Layer Security)**: 클라이언트 ↔ LB 간 전송 암호화 (In-flight Encryption)
- TLS는 SSL의 최신 버전 (현재 대부분 TLS 사용)
- 인증서는 **CA(Certificate Authority)**에서 발급 (Comodo, DigiCert, GoDaddy 등)
- LB는 **X.509 인증서** 사용
- *ACM (AWS Certificate Manager)**으로 인증서 관리 권장

---

### 📛 SNI (Server Name Indication)

- **하나의 웹 서버(LB)에 여러 SSL 인증서를 로드**하여 여러 도메인을 서비스하는 기술
- 클라이언트가 SSL 핸드셰이크 시 접속할 **호스트명을 먼저 전달** → 서버가 올바른 인증서 선택
- **ALB, NLB, CloudFront만 지원** — CLB는 미지원

| LB 유형 | SSL 인증서 | SNI 지원 |
| --- | --- | --- |
| **CLB** | **단 1개** | ❌ |
| **ALB** | **여러 개** (리스너당) | ✅ |
| **NLB** | **여러 개** (리스너당) | ✅ |

---

### ⏳ Connection Draining (연결 드레이닝)

| LB 유형 | 명칭 |
| --- | --- |
| CLB | Connection Draining |
| ALB / NLB | **Deregistration Delay (등록 해제 지연)** |
- 인스턴스가 비정상이거나 등록 해제 중일 때 **진행 중인 요청(In-flight Requests)을 완료할 시간** 부여
- 해제 중인 인스턴스에 **새 요청 전달 중단**
- 설정 범위: **1~3600초** (기본값: 300초)
- 0으로 설정 시 비활성화 (즉시 연결 종료)

> 💡 **짧은 요청이 많은 서비스**: 낮은 값 설정 (ex: 30초). 긴 처리가 필요한 서비스: 높은 값 설정.
> 

---

## 8. ASG (Auto Scaling Group)

### 📐 개념

```
[Load Balancer]
      │
  [ASG]
  ├── EC2 (최소: min)
  ├── EC2 (현재: desired)
  └── EC2 (최대: max)

트래픽 증가 → Scale Out (인스턴스 추가)
트래픽 감소 → Scale In  (인스턴스 제거)
```

- **비용 없음**: ASG 자체는 무료, 실행되는 EC2 인스턴스 비용만 지불
- 비정상 인스턴스 감지 시 **자동으로 새 인스턴스 생성 (Re-create)**
- 새 인스턴스를 자동으로 **LB에 등록**

---

### ⚙️ ASG 구성 요소

**Launch Template (시작 템플릿)** — 인스턴스 설정의 청사진

| 항목 | 내용 |
| --- | --- |
| AMI + Instance Type | 이미지와 인스턴스 사양 |
| EC2 User Data | 부트스트랩 스크립트 |
| EBS Volumes | 스토리지 설정 |
| Security Groups | 방화벽 규칙 |
| SSH Key Pair | 접속 키 |
| IAM Roles | 인스턴스에 부여할 권한 |
| Network + Subnets | 배치할 네트워크 설정 |
| Load Balancer | 연결할 LB |

:::important
구세대 **Launch Configurations**는 Deprecated. 반드시 **Launch Template** 사용.
:::

**크기 설정:**

- **Min Size**: 최소 실행 인스턴스 수 (가용성 하한)
- **Max Size**: 최대 실행 인스턴스 수 (비용 상한)
- **Desired Capacity**: 현재 목표 인스턴스 수

---

### 📊 CloudWatch와 ASG 연동

- CloudWatch 알람(Alarm)이 트리거되면 Scale Out/In 정책 실행
- 메트릭 예시: **평균 CPU 사용률 (Average CPU)** — ASG 전체 인스턴스의 평균값

```
CPU > 70% → CloudWatch Alarm → ASG Scale Out (+2 인스턴스)
CPU < 30% → CloudWatch Alarm → ASG Scale In  (-1 인스턴스)
```

---

## 9. ASG 스케일링 정책

### 🔄 Dynamic Scaling (동적 스케일링)

| 정책 유형 | 설명 | 예시 |
| --- | --- | --- |
| **Target Tracking Scaling** (목표 추적) | 특정 지표를 목표값으로 유지 | 평균 CPU를 40%로 유지 |
| **Simple / Step Scaling** (단순/단계) | 알람 트리거 시 고정 수량 추가/제거 | CPU > 70% → +2, CPU < 30% → -1 |
| **Scheduled Scaling** (예약) | 알려진 사용 패턴에 맞춰 사전 설정 | 매주 월요일 09:00에 5대로 확장 |

---

### 🔮 Predictive Scaling (예측 스케일링)

- ML로 **미래 부하를 예측**하여 사전에 스케일링 예약
- 반복적인 패턴이 있는 워크로드에 효과적

---

### 📏 스케일링에 좋은 지표

| 지표 | 설명 | 적합한 상황 |
| --- | --- | --- |
| **CPU Utilization** (CPU 사용률) | ASG 전체 평균 CPU | 컴퓨팅 집약적 앱 |
| **RequestCountPerTarget** | 인스턴스당 요청 수 | 웹 서버, API 서버 |
| **Average Network In/Out** | 평균 네트워크 입출력 | 네트워크 집약적 앱 |
| **Custom Metric** | CloudWatch에 직접 푸시한 지표 | 비즈니스 로직 기반 |

---

### ⏸️ Scaling Cooldown (스케일링 쿨다운)

- 스케일링 활동 후 **기본 300초(5분) 대기** — 지표가 안정될 때까지 추가 스케일링 중단
- **권장**: 미리 구성된 AMI (Ready-to-use AMI) 사용 → 인스턴스 준비 시간 단축 → 쿨다운 기간 축소 가능

---

### 🔐 LB Security Groups 구성 패턴

```
인터넷 (0.0.0.0/0)
        │ HTTP/HTTPS
        ▼
[Load Balancer Security Group]   ← 0.0.0.0/0:80, 443 허용
        │ 포트 80
        ▼
[EC2 Security Group]             ← LB Security Group에서 온 트래픽만 허용
        (인터넷에서 직접 접근 불가)
```

---

## 10. 핵심 요약

```
로드 밸런서 선택 가이드
├── HTTP/HTTPS, 경로·호스트 기반 라우팅   → ALB (Layer 7)
├── TCP/UDP, 초고성능, 정적 IP 필요       → NLB (Layer 4)
├── 3rd Party 보안 어플라이언스 삽입      → GWLB (Layer 3)
└── (구세대)                              → CLB (사용 지양)

SNI 지원: ALB, NLB, CloudFront (CLB 미지원)
Cross-Zone LB: ALB 기본 활성화, NLB/GWLB 기본 비활성화

ASG 스케일링 정책
├── Target Tracking    → 목표 지표 자동 유지 (가장 간단)
├── Simple/Step        → 알람 기반 고정 수량 조정
├── Scheduled          → 사전 예약 (알려진 패턴)
└── Predictive         → ML 기반 사전 예측

핵심 수치
├── Connection Draining 범위: 1~3600초 (기본 300초)
├── ASG Cooldown 기본: 300초
└── ELB Health Check: 포트 + 경로(/health 일반적)
```

---

### 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| 클라이언트 실제 IP 확인 | ALB → **X-Forwarded-For** 헤더 |
| NLB 정적 IP | AZ당 1개 정적 IP, Elastic IP 할당 가능 |
| GWLB 프로토콜 | **GENEVE (포트 6081)** |
| SNI 지원 LB | **ALB, NLB** (CLB 미지원) |
| CLB SSL 제한 | SSL 인증서 **1개**만, 다중 도메인은 CLB 여러 개 필요 |
| Cross-Zone 과금 | NLB/GWLB는 활성화 시 데이터 요금 부과 |
| ASG 쿨다운 단축 | **미리 구성된 AMI** 사용 → 부팅 시간 단축 |
| Launch Configurations | Deprecated → **Launch Template** 사용 |

---

## 📚 참고 자료

- [ALB 공식 문서](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [NLB 공식 문서](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)
- [GWLB 공식 문서](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html)
- [ASG 공식 문서](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- [ASG 스케일링 정책](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html)