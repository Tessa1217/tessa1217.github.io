---
title: 🌐 AWS Networking — VPC
published: 2026-03-23
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🌐 AWS Networking — VPC

> CIDR · VPC · Subnet · IGW · NAT · NACL · Security Group
> 
> 
> VPC Peering · Endpoints · VPN · Direct Connect · Transit Gateway · Network Firewall
---
## 목차
1. [CIDR — IPv4 주소 체계](#cidr--ipv4-주소-체계)
2. [VPC (Virtual Private Cloud)](#vpc-virtual-private-cloud)
3. [Subnet (서브넷)](#subnet-서브넷)
4. [Internet Gateway (IGW)](#internet-gateway-igw)
5. [Bastion Host](#bastion-host)
6. [NAT Gateway vs. NAT Instance](#nat-gateway-vs-nat-instance)
7. [Security Groups vs. NACLs](#security-groups-vs-nacls)
8. [VPC Flow Logs](#vpc-flow-logs)
9. [VPC Peering](#vpc-peering)
10. [VPC Endpoints (AWS PrivateLink)](#vpc-endpoints-aws-privatelink)
11. [Site-to-Site VPN](#site-to-site-vpn)
12. [AWS Direct Connect (DX)](#aws-direct-connect-dx)
13. [Transit Gateway](#transit-gateway)
14. [VPC Traffic Mirroring](#vpc-traffic-mirroring)
15. [IPv6 & Egress-only Internet Gateway](#ipv6--egress-only-internet-gateway)
16. [AWS Network Firewall](#aws-network-firewall)
17. [네트워킹 비용 (Networking Costs)](#네트워킹-비용-networking-costs)
18. [AWS 네트워크 보안 계층]()
19. [VPC 전체 요약](#vpc-전체-요약)
20. [📌 시험 자주 출제 포인트](#-시험-자주-출제-포인트)
21. [📚 참고 자료](#-참고-자료)
---

## CIDR — IPv4 주소 체계

### CIDR (Classless Inter-Domain Routing)

IP 주소 범위를 정의하는 방법. Security Group 규칙 및 AWS 네트워킹 전반에 사용.

```
WW.XX.YY.ZZ/32   → 단 1개의 IP
0.0.0.0/0        → 모든 IP
192.168.0.0/26   → 192.168.0.0 ~ 192.168.0.63 (64개 IP)
```

**Subnet Mask 빠른 계산:**

| CIDR | 사용 가능 IP 수 | Subnet Mask |
| --- | --- | --- |
| /32 | 1 | 255.255.255.255 |
| /31 | 2 | - |
| /30 | 4 | - |
| /28 | 16 | - |
| /24 | 256 | 255.255.255.0 |
| /16 | 65,536 | 255.255.0.0 |
| /8 | 16,777,216 | 255.0.0.0 |

> 📌 **공식**: 사용 가능 IP 수 = 2^(32 - mask 숫자)
> 

---

### Private IP 범위 (IANA 지정)

| 범위 | CIDR | 용도 |
| --- | --- | --- |
| 10.0.0.0 ~ 10.255.255.255 | 10.0.0.0**/8** | 대규모 네트워크 |
| 172.16.0.0 ~ 172.31.255.255 | 172.16.0.0**/12** | **AWS Default VPC 범위** |
| 192.168.0.0 ~ 192.168.255.255 | 192.168.0.0**/16** | 홈 네트워크 |

나머지 IP는 모두 Public IP.

---

## VPC (Virtual Private Cloud)

| 항목 | 내용 |
| --- | --- |
| **리전당 최대 VPC 수** | **5개** (Soft limit, 증가 요청 가능) |
| **VPC당 최대 CIDR 수** | **5개** |
| **CIDR 최소 크기** | **/28** (16 IP) |
| **CIDR 최대 크기** | **/16** (65,536 IP) |
| **허용 IP 범위** | Private IP 범위만 (10.x, 172.16-31.x, 192.168.x) |

:::warning
⚠️ **VPC CIDR은 기업 네트워크와 겹치면 안 됨** (VPN/Direct Connect 연결 시 라우팅 충돌)
:::

### Default VPC

- 모든 신규 AWS 계정에 자동 생성
- EC2 인스턴스 생성 시 Subnet 미지정 시 Default VPC에 배치
- 기본적으로 Internet Connectivity + Public IPv4 주소 부여
- Public/Private IPv4 DNS 이름 모두 제공

---

## Subnet (서브넷)

- VPC 내 IPv4 주소의 하위 범위, **특정 AZ에 종속**
- AWS는 각 Subnet에서 **5개의 IP를 예약** (사용 불가)

**예시: 10.0.0.0/24 Subnet**

| 주소 | 용도 |
| --- | --- |
| **10.0.0.0** | Network Address |
| **10.0.0.1** | VPC Router용 (AWS 예약) |
| **10.0.0.2** | Amazon-provided DNS용 (AWS 예약) |
| **10.0.0.3** | 미래 용도 (AWS 예약) |
| **10.0.0.255** | Network Broadcast Address (AWS는 VPC에서 Broadcast 미지원, 예약) |

:::tip
29개의 IP가 필요하다면 `/27` (32 IP - 5 = 27 < 29) 불가 → `/26` (64 - 5 = 59) 필요
:::
---

## Internet Gateway (IGW)

- VPC 내 리소스가 **인터넷에 연결**되도록 하는 게이트웨이
- 수평 확장, 고가용성, 이중화 지원
- VPC와 별도로 생성 후 연결 (1 VPC : 1 IGW)
- **IGW 자체만으로는 인터넷 접근 불가** → **Route Table 수정 필수**

```
[EC2] → Route Table → Router → IGW → Internet
```

---

## Bastion Host

- **Public Subnet에 위치한 EC2 인스턴스**
- Private Subnet의 EC2 인스턴스에 SSH 접근하기 위한 Jump Server 역할

**보안 설정:**

```
Bastion Host Security Group:
  Inbound: Port 22 허용 — 회사 공인 IP(CIDR)에서만

Private EC2 Security Group:
  Inbound: Port 22 허용 — Bastion Host의 Security Group 또는 Private IP에서만
```

---

## NAT Gateway vs. NAT Instance

**목적**: Private Subnet의 EC2가 인터넷(외부)에 접근 가능하도록 (반대 방향은 차단)

| 항목 | NAT Gateway | NAT Instance |
| --- | --- | --- |
| **관리** | **AWS 완전 관리형** | 직접 관리 (OS 패치 등) |
| **가용성** | AZ 내 고가용성 | 스크립트로 Failover 구현 |
| **대역폭** | **최대 100 Gbps** (5 Gbps 기본, 자동 확장) | EC2 Instance Type에 따름 |
| **보안 그룹** | ❌ 불필요 | ✅ 직접 관리 |
| **Bastion Host** | ❌ 불가 | ✅ 가능 |
| **비용** | 시간당 + 전송량 | EC2 비용 + 네트워크 비용 |
| **상태** | 현재 권장 | **구식 (2020년 지원 종료)** |

**NAT Instance 필수 설정:**

- Public Subnet에 배치
- **Source/Destination Check 비활성화**
- Elastic IP 연결
- Route Table에서 Private Subnet → NAT Instance로 라우팅

### NAT Gateway 고가용성 구성

```
[AZ1: Private Subnet] → [NAT-GW-1 (AZ1)] → IGW → Internet
[AZ2: Private Subnet] → [NAT-GW-2 (AZ2)] → IGW → Internet
```

- NAT GW는 단일 AZ 내에서만 고가용성
- **Multi-AZ 구성**: AZ별로 별도 NAT GW 생성 (AZ 장애 시 크로스 AZ Failover 불필요)

---

## Security Groups vs. NACLs

### 트래픽 흐름

```
Inbound:  Internet → [NACL Inbound] → [Security Group Inbound] → EC2
Outbound: EC2 → [Security Group Outbound] → [NACL Outbound] → Internet
```

### 비교

| 항목 | Security Group | NACL |
| --- | --- | --- |
| **적용 레벨** | EC2 인스턴스 레벨 | **서브넷 레벨** |
| **규칙 유형** | Allow만 | Allow + **Deny 모두** |
| **상태** | **Stateful** (Inbound 허용 시 Outbound 자동 허용) | **Stateless** (Inbound/Outbound 각각 명시 필요) |
| **규칙 평가** | 모든 규칙 평가 후 결정 | 낮은 번호부터 **순서대로 평가, 첫 매칭 적용** |
| **자동 적용** | 명시적으로 지정해야 함 | **서브넷 내 모든 EC2에 자동 적용** |

---

### NACL 규칙 특성

- 규칙 번호: **1 ~ 32766** (낮은 번호가 높은 우선순위)
- 마지막 규칙:  (asterisk) — 일치하는 규칙 없으면 Deny
- 새로 생성된 NACL: **모든 트래픽 Deny**
- **Default NACL**: 모든 Inbound/Outbound 허용 → 수정 금지, Custom NACL 사용 권장
- 특정 IP 차단에 적합 (Security Group은 Deny 불가)
- 규칙 추가 시 **100 단위 증분** 권장

### Ephemeral Ports (임시 포트)

- 클라이언트가 서버에 연결할 때 **응답을 받을 임시 포트를 무작위로 할당**
- 운영체제별 임시 포트 범위:
    - IANA/Windows 10: **49152 ~ 65535**
    - Linux Kernel: **32768 ~ 60999**

**NACL에서 Ephemeral Port 고려 예시 (Web → DB 통신):**

```
Web-NACL Outbound:  TCP 3306 → DB Subnet CIDR 허용
DB-NACL Inbound:    TCP 3306 ← Web Subnet CIDR 허용
DB-NACL Outbound:   TCP 1024-65535 → Web Subnet CIDR 허용  ← 임시 포트!
Web-NACL Inbound:   TCP 1024-65535 ← DB Subnet CIDR 허용   ← 임시 포트!
```

### VPC Flow Logs로 SG/NACL 트러블슈팅

| Flow Log Action | 원인 |
| --- | --- |
| Inbound REJECT | **NACL** 또는 **Security Group** 차단 |
| Inbound ACCEPT + Outbound REJECT | **NACL** 차단 (Stateless) |

> 💡 Security Group은 Stateful이므로 Inbound ACCEPT 시 Outbound는 자동 허용됨.
> 

---

## VPC Flow Logs

- VPC / Subnet / ENI 레벨에서 **IP 트래픽 정보 캡처**
- AWS 관리형 인터페이스(ELB, RDS, ElastiCache, NAT GW 등)도 캡처 가능
- 대상: **S3, CloudWatch Logs, Kinesis Data Firehose**

**Flow Log 필드:**

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

**분석 아키텍처:**

```
VPC Flow Logs → CloudWatch Logs → Contributor Insights → Top-10 IP
VPC Flow Logs → CloudWatch Logs → Metric Filter → Alarm → SNS
VPC Flow Logs → S3 → Athena → QuickSight
```
:::important
⚠️ CloudWatch Logs로 전송 시 IAM Role에 `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` 권한 필요
:::

---

## VPC Peering

- **AWS 내부 네트워크로 두 VPC를 프라이빗 연결**
- 같은 네트워크처럼 동작

**제약 조건:**

- CIDR 범위 **겹치면 안 됨**
- **Non-transitive**: A-B, B-C 피어링이 있어도 A-C 직접 연결 안 됨 → A-C 피어링 별도 생성 필요
- 연결 후 **양쪽 VPC의 Route Table 모두 업데이트** 필요
- 서로 다른 계정/리전 간 VPC Peering 가능
- 피어링된 VPC의 Security Group 참조 가능 (같은 리전, 다른 계정 간)

```
VPC-A ←→ VPC-B ←→ VPC-C
A와 C가 통신하려면 별도 A-C 피어링 필요 (B를 통한 전이 없음)
```

---

## VPC Endpoints (AWS PrivateLink)

- Public Internet을 거치지 않고 **AWS 내부 네트워크로 AWS 서비스에 접근**
- IGW, NAT GW 없이 AWS 서비스 접근 가능
- 수평 확장, 이중화

### Endpoint 유형

| 항목 | Interface Endpoint | Gateway Endpoint |
| --- | --- | --- |
| **구현** | ENI (Private IP) 생성 | Route Table에 Gateway 추가 |
| **Security Group** | ✅ 필요 | ❌ 불필요 |
| **지원 서비스** | 대부분의 AWS 서비스 | **S3, DynamoDB만** |
| **비용** | 시간당 + GB당 | **무료** |
| **확장** | ENI 기반 | Route Table 기반 (자동 확장) |

**S3 Endpoint 선택 기준:**

```
일반적으로 → Gateway Endpoint 권장 (무료, Route Table만 수정)
On-premises (VPN/Direct Connect), 다른 VPC, 다른 리전에서 접근
                                  → Interface Endpoint 필요
```

---

## Site-to-Site VPN

- On-premises ↔ AWS VPC 간 **암호화된 VPN 터널 (공용 인터넷 경유)**

**구성 요소:**

| 구성요소 | 설명 |
| --- | --- |
| **VGW (Virtual Private Gateway)** | AWS 측 VPN Concentrator, VPC에 연결 |
| **CGW (Customer Gateway)** | 고객 측 소프트웨어 앱 또는 물리 장비 |

**설정 필수 사항:**

- CGW 디바이스의 **공인 IP** 사용 (NAT 뒤에 있으면 NAT의 Public IP)
- 서브넷 Route Table에서 **VGW Route Propagation 활성화**
- On-premises → EC2 Ping 필요 시 Security Group에 **ICMP 프로토콜** Inbound 허용

### AWS VPN CloudHub

- 여러 Site와 VPN 연결을 하나의 VGW에 연결 → **Hub-and-Spoke** 구성
- 각 Site 간 보안 통신 (저비용)
- 공용 인터넷 경유 (암호화됨)
- 동적 라우팅 + Route Table 설정 필요

---

## AWS Direct Connect (DX)

- On-premises ↔ AWS 간 **전용 물리 Private 연결** (인터넷 미경유)
- DC → AWS Direct Connect Location → VGW → VPC
- IPv4, IPv6 모두 지원

**Use Cases:**

- 대용량 데이터셋 처리 (높은 대역폭, 낮은 비용)
- 실시간 데이터 피드 (일관된 네트워크 성능)
- Hybrid 환경 (온프레미스 + 클라우드)

### Direct Connect 연결 유형

| 유형 | 용량 | 특징 |
| --- | --- | --- |
| **Dedicated Connections** | 1/10/100 Gbps | 물리적 Ethernet 포트 전용 |
| **Hosted Connections** | 50 Mbps ~ 10 Gbps | AWS Direct Connect Partner 통해 요청, 온디맨드 용량 조정 가능 |

:::important
⚠️ 신규 연결 구축 리드 타임: 보통 **1개월 이상**
:::

### Direct Connect 암호화

- 데이터 전송 중 암호화 **기본 없음** (Private 연결이지만 비암호화)
- 암호화 필요 시: **Direct Connect + Site-to-Site VPN** 조합 → IPsec 암호화

### Direct Connect Gateway

- 여러 리전의 여러 VPC에 Direct Connect를 연결할 때 사용
- 단일 Direct Connect로 **여러 리전의 VPC에 접근**

### Direct Connect 이중화 (Resiliency)

| 수준 | 구성 |
| --- | --- |
| **High Resiliency** | 여러 위치(Location)에 각 1개 DX 연결 |
| **Maximum Resiliency** | 여러 위치에서 각각 별도 디바이스에 2개 이상 DX 연결 |

### Direct Connect 장애 시 Backup

- DX 장애 대비 **Backup으로 Site-to-Site VPN** 연결 설정 가능

---

## Transit Gateway

- 수천 개의 VPC + On-premises를 **Hub-and-Spoke(Star) 방식으로 연결**
- VPC Peering의 Non-transitive 문제 해결 (Transit GW는 Transitive)

| 항목 | 내용 |
| --- | --- |
| **연결 가능 대상** | VPC, Direct Connect Gateway, VPN Connection (CGW) |
| **범위** | Regional Resource (크로스 리전 지원, 리전 간 TGW Peering) |
| **계정 공유** | **AWS Resource Access Manager (RAM)** 사용 |
| **Route Tables** | VPC 간 통신 제어 가능 |
| **특이 기능** | **IP Multicast** 지원 (다른 AWS 서비스 미지원) |

### Transit Gateway — ECMP (Equal-Cost Multi-Path)

- **ECMP**: 여러 최적 경로로 패킷 전달하는 라우팅 전략
- VPN → Transit Gateway 연결 시 여러 Site-to-Site VPN으로 **대역폭 배가** 가능

| 연결 방식 | 최대 처리량 |
| --- | --- |
| VPN → Virtual Private Gateway | **1.25 Gbps** (2개 터널) |
| VPN → Transit Gateway (ECMP) | **2.5 Gbps** (2 터널 사용) → VPN 추가로 배가 가능 |

> 💡 Direct Connect를 여러 계정과 공유할 때도 Transit Gateway + RAM 조합 사용
> 

---

## VPC Traffic Mirroring

- VPC 내 네트워크 트래픽을 **캡처하여 분석**
- 소스: ENI / 대상: ENI 또는 NLB
- 같은 VPC 또는 VPC Peering된 다른 VPC 간에도 가능
- **Use Cases**: 콘텐츠 검사, 위협 모니터링, 네트워크 트러블슈팅

---

## IPv6 & Egress-only Internet Gateway

- AWS의 모든 IPv6 주소는 **Public + 인터넷 라우팅 가능** (사설 IPv6 범위 없음)
- VPC에서 IPv4는 **비활성화 불가**, IPv6는 선택적 활성화 (Dual-Stack)
- EC2는 Private IPv4 + Public IPv6 모두 가짐

**Egress-only Internet Gateway:**

- **IPv6 전용 NAT Gateway 역할**
- VPC 인스턴스의 IPv6 아웃바운드 허용, 인터넷에서의 IPv6 인바운드 차단

| 방향 | IPv4 | IPv6 |
| --- | --- | --- |
| Private → Internet | **NAT Gateway** | **Egress-only IGW** |
| Internet → Private | 차단 | 차단 |

:::note
📌 EC2 인스턴스 시작 실패 시 — IPv6 주소 공간은 매우 넓으므로 고갈 아님. **IPv4 주소 고갈**이 원인 → Subnet에 새 IPv4 CIDR 추가
:::
---

## AWS Network Firewall

- **VPC 전체**를 Layer 3 ~ Layer 7까지 보호
- 내부적으로 **AWS Gateway Load Balancer** 사용
- AWS Firewall Manager로 **Cross-Account 중앙 관리**

**검사 가능한 트래픽:**

- VPC ↔ VPC
- 인터넷 Inbound/Outbound
- Direct Connect & Site-to-Site VPN

**세밀한 제어 (Fine-grained Controls):**

- IP/Port 기반 규칙
- Protocol 기반
- **Stateful Domain List**: `.mycorp.com` 등 도메인 허용/차단
- Regex 패턴 매칭
- Active Flow Inspection (침입 방지, IPS 기능)
- 로그 전송: S3, CloudWatch Logs, Kinesis Data Firehose

---

## 네트워킹 비용 (Networking Costs)

### EC2 인스턴스 간 트래픽 비용

| 경로 | 비용 |
| --- | --- |
| EC2 인바운드 트래픽 | **무료** |
| **같은 AZ** 내 EC2 간 (Private IP) | **무료** |
| **다른 AZ** 간 EC2 (Public/Elastic IP) | **$0.02/GB** |
| **다른 AZ** 간 EC2 (Private IP) | **$0.01/GB** |
| 리전 간 (Inter-Region) | **$0.02/GB** |

> 💡 **Private IP 사용 권장** (Public IP 대비 절반 비용 + 더 나은 성능)
> 

### S3 데이터 전송 비용

| 경로 | 비용 |
| --- | --- |
| S3 Ingress (업로드) | **무료** |
| S3 → 인터넷 | **$0.09/GB** |
| S3 Transfer Acceleration | +$0.04 ~ $0.08/GB |
| S3 → CloudFront | **무료** |
| CloudFront → 인터넷 | **$0.085/GB** (S3 직접보다 저렴 + 캐싱) |
| S3 Cross-Region Replication | **$0.02/GB** |

### NAT Gateway vs. Gateway VPC Endpoint (S3 접근 비용)

| 방법 | 비용 |
| --- | --- |
| **NAT Gateway → IGW → S3** | NAT GW 시간당 $0.045 + 처리 GB당 $0.045 + S3 데이터 전송 |
| **Gateway VPC Endpoint → S3** | **Gateway Endpoint 사용 무료** ($0.01 In/Out 동일 리전) |

> ✅ **Private Subnet에서 S3 접근 시 Gateway VPC Endpoint가 훨씬 저렴**
> 

---

## AWS 네트워크 보안 계층

```
L3-4 네트워크 보호:  NACLs, Security Groups
L3-7 완전 보호:      AWS Network Firewall (Gateway LB 내부 사용)
DDoS 보호:           AWS Shield (Standard/Advanced)
악성 HTTP 요청 차단:  AWS WAF
멀티 계정 관리:       AWS Firewall Manager
```

---

## VPC 전체 요약

```
CIDR              : IP 범위 정의
VPC               : 리전 내 격리된 가상 네트워크 (최대 5개/리전)
Subnet            : AZ에 종속된 IP 범위 (5개 IP AWS 예약)
IGW               : VPC → Internet (Route Table 수정 필요)
Bastion Host      : Public EC2 → Private EC2 SSH Jump 서버
NAT Instance      : 구식, Source/Dest Check 비활성화 필요
NAT Gateway       : AWS 관리형, Private → Internet (IPv4)
NACL              : Stateless, Subnet 레벨, Allow + Deny
Security Group    : Stateful, EC2 레벨, Allow만
VPC Peering       : Non-transitive, CIDR 비중복 필수, Route Table 업데이트
VPC Endpoints     : Gateway(S3/DynamoDB, 무료) / Interface(PrivateLink, 유료)
VPC Flow Logs     : IP 트래픽 캡처 → S3/CloudWatch/Firehose
Site-to-Site VPN  : VGW + CGW, 공용 인터넷 경유 암호화
VPN CloudHub      : Hub-and-Spoke 다중 Site VPN
Direct Connect    : 전용 물리 연결 (Private, 암호화 없음)
DX Gateway        : 단일 DX로 여러 리전 VPC 접근
Transit Gateway   : 수천 VPC/VPN/DX Hub-and-Spoke, IP Multicast
Traffic Mirroring : ENI 트래픽 캡처 분석
Egress-only IGW   : IPv6 전용 NAT GW
Network Firewall  : L3-7 완전 VPC 보호
```

---

## 📌 시험 자주 출제 포인트

| 포인트 | 내용 |
| --- | --- |
| Subnet 예약 IP | **5개** (첫 4개 + 마지막 1개) |
| VPC당 최대 CIDR | **5개** |
| CIDR 최소/최대 | /28 (16개) ~ /16 (65,536개) |
| IGW만으로 인터넷 접근 | **Route Table 수정 필요** |
| NAT Instance Source/Dest Check | **반드시 비활성화** |
| NAT GW 최대 대역폭 | **100 Gbps** (자동 확장) |
| NAT GW Security Group | **없음** (불필요) |
| NAT GW Multi-AZ HA | AZ마다 **별도 NAT GW** 생성 |
| NACL vs SG 상태 | NACL: **Stateless** / SG: **Stateful** |
| NACL 규칙 평가 | 낮은 번호부터 순서대로, **첫 매칭 적용** |
| Default NACL | **모든 트래픽 허용** (수정 금지) |
| 특정 IP 차단 | NACL의 **Deny 규칙** 사용 (SG는 Deny 불가) |
| VPC Peering Transitive | **없음** → 각 VPC 쌍마다 별도 Peering |
| Gateway Endpoint 대상 | **S3, DynamoDB만** |
| Gateway Endpoint 비용 | **무료** |
| Interface Endpoint + On-premises | Site-to-Site VPN/DX 경유 접근 시 **Interface Endpoint** |
| Direct Connect 암호화 | 기본 없음 → **DX + VPN으로 IPsec 추가** |
| Direct Connect 연결 리드 타임 | **1개월 이상** |
| DX Gateway | 단일 DX로 **여러 리전** VPC 접근 |
| Transit Gateway 특이점 | **IP Multicast** 지원 (AWS 유일) |
| Transit GW 계정 공유 | **AWS Resource Access Manager (RAM)** |
| VPN → TGW 대역폭 | **2.5 Gbps** (ECMP), VPN 추가 시 배가 가능 |
| VPN → VGW 대역폭 | **1.25 Gbps** |
| EC2 시작 실패 이유 | IPv6 주소 고갈 아님 → **IPv4 주소 고갈** |
| Egress-only IGW 대상 | **IPv6** 아웃바운드만 |
| Private → S3 최저 비용 | **Gateway VPC Endpoint** (무료) |
| Flow Log Action REJECT (Inbound) | NACL 또는 SG 차단 |
| Flow Log Action ACCEPT+REJECT | **NACL** (Stateless) 차단 |

---

## 📚 참고 자료

- [VPC 공식 문서](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)
- [Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)
- [AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html)