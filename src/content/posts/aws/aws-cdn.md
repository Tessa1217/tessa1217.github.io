---
title: 🌐 AWS CloudFront & Global Accelerator
published: 2026-03-22
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🌐 AWS CloudFront & Global Accelerator

> CDN을 통한 콘텐츠 가속과 네트워크 레벨 애플리케이션 가속의 핵심 서비스
> 
> 
> 두 서비스 모두 AWS Global Edge Network를 활용하지만 목적과 동작 방식이 다름
> 
---
## 목차
1. [CloudFront 개요](#cloudfront-개요)
2. [CloudFront Origins (콘텐츠 원본)](#cloudfront-origins-콘텐츠-원본)
3. [CloudFront vs. S3 Cross-Region Replication](#cloudfront-vs-s3-cross-region-replication)
4. [CloudFront 캐시 동작 (Caching)](#cloudfront-캐시-동작-caching)
5. [CloudFront Geo Restriction (지역 제한)](#cloudfront-geo-restriction-지역-제한)
6. [CloudFront + S3 보안 아키텍처 (OAC)](#cloudfront--s3-보안-아키텍처-oac)
7. [AWS Global Accelerator](#aws-global-accelerator)
8. [CloudFront vs. Global Accelerator 비교](#cloudfront-vs-global-accelerator-비교)
9. [시험 자주 출제 포인트 총정리](#시험-자주-출제-포인트-총정리)
10. [참고 자료](#참고-자료)
---
## CloudFront 개요

- **CDN (Content Delivery Network)**: 콘텐츠를 Edge Location에 캐시하여 사용자에게 낮은 지연시간으로 제공
- 전 세계 수백 개의 **Edge Locations (Points of Presence)**에서 콘텐츠 캐시
- **DDoS 보호**: AWS Shield + AWS WAF(Web Application Firewall)와 통합

---

## CloudFront Origins (콘텐츠 원본)

CloudFront가 콘텐츠를 가져오는 원본 서버는 크게 세 가지로 나뉜다.

### 1. S3 Bucket

- 파일 배포 및 Edge Location에 캐시
- CloudFront를 통한 S3 업로드 (Ingress) 지원
- *OAC (Origin Access Control)**로 S3 Bucket 보안 강화

```
[사용자]  →  [CloudFront Edge]  →  [S3 Bucket]
                                      ↑
                              OAC로 보호
                              (S3 Public 비활성화 유지)
```
:::tip
S3 Bucket을 CloudFront Origin으로 사용할 때는 **OAC** 사용. 구세대 방식인 OAI(Origin Access Identity)는 deprecated 예정.
:::

---

### 2. VPC Origin

- **VPC Private Subnet**에 호스팅된 애플리케이션에서 콘텐츠 제공
- 인터넷에 노출하지 않고 Private 리소스에서 직접 트래픽 수신:
    - Private ALB (Application Load Balancer)
    - Private NLB (Network Load Balancer)
    - Private EC2 Instances
- 기존 방식(Public Network)의 경우 Edge Location의 Public IP를 Security Group에 허용해야 했으나, **VPC Origin 방식은 이 불편함을 해소**

```
[사용자]
    │
    ▼
[CloudFront Edge]
    │  (VPC Origin 방식 — 인터넷 미경유)
    ▼
[Private ALB / NLB / EC2]  ← 인터넷에 노출 불필요
```

---

### 3. Custom Origin (HTTP)

- S3 Static Website (S3 정적 웹사이트 엔드포인트)
- 모든 Public HTTP Backend (온프레미스 서버, 다른 클라우드 등)

---

## CloudFront vs. S3 Cross-Region Replication

시험에서 두 서비스를 비교하는 문제가 자주 출제됨.

| 항목 | CloudFront | S3 Cross-Region Replication |
| --- | --- | --- |
| **범위** | 전 세계 모든 Edge Location 자동 적용 | 복제할 리전을 각각 직접 설정 |
| **업데이트 반영** | TTL 만료 시 반영 (캐시 기간 동안 지연) | **Near real-time** 업데이트 |
| **접근 방향** | 읽기/쓰기 가능 (Ingress 지원) | **읽기 전용 (Read only)** |
| **적합한 콘텐츠** | 전 세계에서 접근하는 **Static 콘텐츠** (이미지, 영상 등) | 소수 리전에서 **낮은 지연시간으로 Dynamic 콘텐츠** 제공 |
| **데이터 복사** | 복사 없음 (캐시만) | 실제 객체를 다른 리전으로 복제 |

---

## CloudFront 캐시 동작 (Caching)

### TTL (Time To Live)

- CloudFront는 TTL 동안 캐시된 콘텐츠를 사용자에게 제공
- TTL 동안은 Origin에 요청하지 않음 → Origin 부하 감소
- TTL은 Cache-Control, Expires Header로 제어 가능

### Cache Invalidation (캐시 무효화)

- Origin 콘텐츠를 업데이트해도 **TTL이 만료되기 전까지 CloudFront는 변경을 모름**
- **CloudFront Invalidation**을 직접 실행하면 TTL 무시하고 캐시 강제 갱신

```
Invalidation 경로 예시:
/**          → 전체 캐시 무효화
/images/**   → /images/ 하위 전체 무효화
/index.html  → 특정 파일만 무효화
```
:::tip[Deployment Tip]
콘텐츠 파일명에 버전/해시를 포함시키면 (예: `app.v2.js`) Invalidation 없이도 새 버전 즉시 제공 가능.
:::

---

## CloudFront Geo Restriction (지역 제한)

- 특정 국가 사용자의 콘텐츠 접근을 제어
- 국가 판별: **3rd Party Geo-IP Database** 사용

| 설정 | 설명 |
| --- | --- |
| **Allowlist** | 승인된 국가 목록의 사용자만 접근 허용 |
| **Blocklist** | 차단된 국가 목록의 사용자는 접근 거부 |

**Use Case**: 저작권법(Copyright Laws)에 따른 콘텐츠 배포 제한

---

## CloudFront + S3 보안 아키텍처 (OAC)

CloudFront를 통해서만 S3에 접근하고 직접 접근은 차단하는 패턴.

```
[사용자]
    │
    ▼
[CloudFront Distribution]
    │  OAC로 인증된 요청만
    ▼
[S3 Bucket] ← Block Public Access 활성화 유지
              Bucket Policy: CloudFront Service Principal만 허용
```

**Bucket Policy 예시 (OAC):**

```json
{
    "Effect": "Allow",
    "Principal": {
        "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
        "StringEquals": {
            "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT:distribution/DISTRIBUTION_ID"
        }
    }
}
```

---

### CloudFront Signed URL / Signed Cookies

- **Private 콘텐츠**에 대한 접근 제어 — S3 Pre-signed URL과 다름

| 항목 | CloudFront Signed URL | CloudFront Signed Cookie |
| --- | --- | --- |
| **접근 범위** | 파일 **1개**에 대한 접근 | **여러 파일** 또는 전체 경로 접근 |
| **사용 시나리오** | 개별 파일 다운로드 링크 | 프리미엄 멤버십 전체 콘텐츠 접근 |

**S3 Pre-signed URL과의 차이점:**

| 항목 | S3 Pre-signed URL | CloudFront Signed URL |
| --- | --- | --- |
| **경유 서버** | S3 직접 접근 | CloudFront Edge → S3 |
| **권한 범위** | 생성자의 IAM 권한 상속 | CloudFront Key Pair 기반 |
| **캐시 활용** | ❌ S3 직접 접근 | ✅ CloudFront 캐시 활용 |
| **IP 제한** | 불가 | 가능 (정책에 포함) |

:::note
**CDN을 통해 Private 콘텐츠를 제공할 때는 CloudFront Signed URL/Cookie 사용**. S3 Pre-signed URL은 S3 직접 접근이므로 CDN 캐시 혜택 없음.
:::

---

### CloudFront Price Classes

- CloudFront Edge Location은 지역별로 데이터 전송 비용이 다름
- **Price Class**로 사용할 Edge Location 범위를 제한하여 비용 절감

| Price Class | 포함 지역 | 비용 |
| --- | --- | --- |
| **Price Class All** | 전체 Edge Location | 가장 비쌈, 최고 성능 |
| **Price Class 200** | 대부분 지역 (비용 높은 일부 제외) | 중간 |
| **Price Class 100** | 가장 저렴한 지역만 (북미, 유럽 등) | 가장 저렴 |

---

### CloudFront Origin Groups (Failover)

- **Primary Origin**이 실패하면 **Secondary Origin**으로 자동 Failover
- High Availability 구성에 활용

```
[CloudFront Distribution]
    │
    ├── Primary Origin (us-east-1 S3)  ← 정상 시 사용
    └── Secondary Origin (us-west-2 S3) ← Primary 실패 시 자동 전환
```

---

### Lambda@Edge / CloudFront Functions

Edge Location에서 코드를 실행하여 요청/응답을 동적으로 처리.

| 항목 | CloudFront Functions | Lambda@Edge |
| --- | --- | --- |
| **런타임** | JavaScript | Node.js, Python |
| **실행 트리거** | Viewer Request/Response | Viewer/Origin Request/Response |
| **실행 위치** | CloudFront Edge (경량) | 리전 Edge Location |
| **최대 실행 시간** | 1ms | 5~10초 |
| **요청 수/초** | 수백만 req/s | 수천 req/s |
| **사용 사례** | Header 조작, URL 리다이렉트, A/B 테스트 | 복잡한 로직, 외부 API 호출 |

---

## AWS Global Accelerator

### 문제 정의

글로벌 사용자가 특정 리전에 배포된 애플리케이션에 접근할 때, Public Internet을 통한 **다수의 Hop(경유지)**으로 인해 지연 시간과 패킷 손실 발생.

```
[일본 사용자]  →  Public Internet  →  [us-east-1 ALB]
               (수많은 라우터 경유 → 높은 지연, 불안정)
```

### 해결: AWS 내부 네트워크 활용

```
[일본 사용자]  →  [도쿄 Edge Location]  →  AWS 전용 백본망  →  [us-east-1 ALB]
               (짧은 Public Internet 구간)    (고속, 안정)
```

---

### Anycast IP

- Global Accelerator는 애플리케이션에 **2개의 Anycast IP** 할당
- **Unicast IP**: 하나의 서버가 하나의 IP를 보유
- **Anycast IP**: 여러 서버가 **같은 IP**를 보유, 클라이언트는 자동으로 **가장 가까운 서버로 라우팅**

```
Anycast IP: 1.2.3.4  ← 전 세계 어디서 요청해도 같은 IP
    │
    ├── 도쿄 Edge → 도쿄 사용자 처리
    ├── 런던 Edge → 유럽 사용자 처리
    └── 버지니아 Edge → 미국 사용자 처리
```

---

### Global Accelerator 주요 특성

| 항목 | 내용 |
| --- | --- |
| **고정 IP** | **2개의 Anycast IP** (변경 없음) |
| **지원 리소스** | Elastic IP, EC2, ALB, NLB (Public/Private 모두) |
| **Health Check** | 애플리케이션 헬스 체크, 비정상 시 **1분 이내 Failover** |
| **보안** | 외부에 노출되는 IP가 **단 2개** → 화이트리스트 관리 용이 |
| **DDoS 보호** | AWS Shield 통합 |
| **클라이언트 캐시 이슈** | IP가 변경되지 않으므로 **DNS 캐시 문제 없음** |

---

## CloudFront vs. Global Accelerator 비교

> 📌 **시험에서 가장 자주 출제되는 비교 — 반드시 구분**
> 

| 항목 | CloudFront | Global Accelerator |
| --- | --- | --- |
| **주요 기능** | 콘텐츠 **캐싱 및 배포** | 네트워크 레벨 **트래픽 가속** |
| **캐싱** | ✅ Edge Location에 콘텐츠 캐시 | ❌ 캐시 없음 (프록시만) |
| **콘텐츠 처리** | Edge에서 **콘텐츠 직접 제공** | Edge에서 **패킷을 원본 서버로 전달** |
| **프로토콜** | **HTTP/HTTPS** 전용 | **TCP, UDP** 모두 지원 |
| **IP 주소** | 동적 IP (DNS로 접근) | **고정 Anycast IP 2개** |
| **Failover** | 느림 (DNS TTL 영향) | **1분 이내** 빠른 Failover |
| **적합한 Use Case** | Static/Dynamic **HTTP 콘텐츠** 가속 | 게임(UDP), IoT(MQTT), VoIP, 고정 IP 필요, 빠른 Regional Failover |

**간단 선택 기준:**

```
HTTP 콘텐츠를 전 세계에 빠르게 배포 → CloudFront
캐시와 무관한 TCP/UDP 애플리케이션 가속 → Global Accelerator
고정 IP 2개로 방화벽 화이트리스팅 필요 → Global Accelerator
HTTP 요청이지만 Static IP 또는 빠른 Failover 필요 → Global Accelerator
```

---

## 📌 시험 자주 출제 포인트 총정리

| 포인트 | 내용 |
| --- | --- |
| CloudFront 보안 통합 | **AWS Shield + WAF** |
| S3 Origin 보안 | **OAC (Origin Access Control)** (구: OAI deprecated 예정) |
| VPC Origin | Private ALB/NLB/EC2를 인터넷 노출 없이 Origin으로 사용 가능 |
| Cache Invalidation | `/**` 또는 `/path/**` 로 TTL 무시 강제 갱신 |
| Geo Restriction 판별 | **3rd Party Geo-IP Database** 사용 |
| CloudFront vs S3 CRR | 정적 전세계 배포 → CloudFront / 소수 리전 동적 콘텐츠 → S3 CRR |
| Signed URL | 파일 **1개** 접근 제한 |
| Signed Cookie | **여러 파일** 또는 경로 전체 접근 제한 |
| CloudFront Signed URL vs S3 Pre-signed URL | CloudFront: Edge 캐시 활용, IP 제한 가능 / S3: S3 직접 접근, IAM 권한 상속 |
| Price Class 100 | **가장 저렴한 지역만** (북미, 유럽 등) |
| Origin Failover | Primary 실패 시 Secondary로 자동 전환 |
| CloudFront Functions | **경량 JS**, 1ms, Viewer Request/Response |
| Lambda@Edge | **복잡한 로직**, 5~10초, Viewer + Origin 트리거 |
| Global Accelerator IP 수 | **2개의 Anycast IP** |
| Global Accelerator Failover | **1분 이내** |
| Anycast IP 개념 | 여러 서버가 **같은 IP** 공유, 클라이언트는 가장 가까운 서버로 라우팅 |
| Global Accelerator 지원 프로토콜 | **TCP + UDP** (CloudFront는 HTTP만) |
| 고정 IP 화이트리스팅 | **Global Accelerator** (IP 2개만 노출) |
| Non-HTTP 가속 (UDP, MQTT, VoIP) | **Global Accelerator** |

---

## 📚 참고 자료

- [CloudFront 공식 문서](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [CloudFront Origin Access Control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [CloudFront Signed URLs and Cookies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html)
- [Lambda@Edge 공식 문서](https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html)
- [Global Accelerator 공식 문서](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)