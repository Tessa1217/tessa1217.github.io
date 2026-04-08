---
title: 📨 AWS Integration & Messaging
published: 2026-03-22
tags: [AWS, Cloud, Certificates]
category: AWS
draft: true
---
# 📨 AWS Integration & Messaging

> SQS · SNS · Kinesis · Amazon MQ
> 
> 
> 애플리케이션 간 결합(Coupling)을 제거하고 독립적인 확장을 가능하게 하는 핵심 서비스
---
## 목차
1. [메시징 서비스 개요](#메시징-서비스-개요)
2. [Amazon SQS (Simple Queue Service)](#amazon-sqs-simple-queue-service)
3. [Amazon SQS - FIFO Queue](#amazon-sqs---fifo-queue)
4. [Amazon SNS (Simple Notification Service)](#amazon-sns-simple-notification-service)
5. [Amazon Kinesis](#amazon-kinesis)
6. [SQS vs. SNS vs. Kinesis — 최종 비교](#sqs-vs-sns-vs-kinesis--최종-비교)
7. [Amazon MQ](#amazon-mq)
8. [📌 시험 자주 출제 포인트 총정리](#-시험-자주-출제-포인트-총정리)
9. [📚 참고 자료](#-참고-자료)
---

## 메시징 서비스 개요

| 서비스 | 패턴 | 핵심 특징 |
| --- | --- | --- |
| **SQS** | Queue (Pull) | 메시지 큐, Consumer가 직접 Pull |
| **SNS** | Pub/Sub (Push) | 하나의 메시지를 다수 Subscriber에게 Push |
| **Kinesis** | Real-time Streaming | 실시간 대용량 데이터 스트리밍 |
| **Amazon MQ** | 오픈 프로토콜 브로커 | MQTT, AMQP 등 기존 온프레미스 프로토콜 지원 |

**Decoupling의 이유:**

```
Synchronous 통신:  [앱 A] ─직접 호출─→ [앱 B]   ← 한쪽 장애 시 전체 영향
Asynchronous 통신: [앱 A] → [Queue/Topic] → [앱 B] ← 독립적 확장 가능
```

---

## Amazon SQS (Simple Queue Service)

### Standard Queue

| 항목 | 내용 |
| --- | --- |
| **Throughput** | 무제한 (Unlimited) |
| **메시지 보존 기간** | 기본 **4일**, 최대 **14일** |
| **메시지 최대 크기** | **256 KB** |
| **지연 시간** | < 10ms (Publish/Receive) |
| **전달 방식** | At-least-once delivery (중복 가능) |
| **순서 보장** | Best-effort ordering (순서 미보장) |

---

### SQS 메시지 흐름

```
[Producer] ─ SendMessage API ─→ [SQS Queue] ─ Poll ─→ [Consumer]
                                                          │
                                                   처리 완료 후
                                                   DeleteMessage API
```

- Consumer는 한 번에 최대 **10개** 메시지 수신 (Receive)
- 메시지는 Consumer가 DeleteMessage API를 호출하기 전까지 Queue에 유지
- Consumer는 EC2, Lambda, 온프레미스 서버 등 어디서든 실행 가능

---

### Message Visibility Timeout (메시지 가시성 타임아웃)

```
Consumer가 메시지를 Poll
    │
    ▼
메시지가 다른 Consumer에게 "비가시(Invisible)" 상태로 전환
    │
    ▼
기본 30초 내에 처리 + DeleteMessage 완료 → 메시지 영구 삭제
처리 실패 / 시간 초과 → 메시지가 다시 Queue에 "가시(Visible)" 상태로 복귀
                          → 다른 Consumer가 재처리 (중복 처리 가능)
```

| 항목 | 내용 |
| --- | --- |
| **기본값** | **30초** |
| **타임아웃 연장 방법** | **ChangeMessageVisibility API** 호출 |
| **너무 낮으면** | 처리 시간 초과로 중복 메시지 발생 |
| **너무 높으면** | Consumer 크래시 시 재처리까지 오랜 대기 |

---

### Long Polling (롱 폴링)

- Consumer가 Queue에 메시지가 없을 때 **일정 시간 대기(Wait)**하여 메시지 도착 시 즉시 수신
- **Short Polling** (기본): 메시지 없으면 즉시 빈 응답 → API 호출 낭비
- **Long Polling 장점**:
    - SQS에 대한 API 호출 수 감소 → **비용 절감**
    - 지연 시간(Latency) 감소
    - 애플리케이션 효율 향상
- 대기 시간: **1초 ~ 20초** (20초 권장)
- 설정: Queue 레벨 또는 API 레벨 (`WaitTimeSeconds` 파라미터)

> 💡 **Long Polling이 Short Polling보다 항상 권장됨**
> 

---

### SQS 보안

| 항목 | 내용 |
| --- | --- |
| **In-flight 암호화** | HTTPS API |
| **At-rest 암호화** | AWS KMS |
| **Client-side 암호화** | 클라이언트 직접 처리 |
| **접근 제어** | IAM Policies |
| **SQS Access Policy** | S3 Bucket Policy 형태 — Cross-Account 접근, SNS/S3 등 다른 서비스의 쓰기 허용 시 사용 |

---

### SQS + Auto Scaling Group (ASG) 패턴

```
[요청]
  │
  ▼
[Frontend App (ASG)] ─ SendMessage ─→ [SQS Queue]
                                           │
                                     CloudWatch Metric
                                     (ApproximateNumberOfMessages)
                                           │ Queue 길이 임계값 초과
                                           ▼
                                     [CloudWatch Alarm]
                                           │
                                           ▼
                                     [Backend ASG Scale Out]
                                           │
                                     ReceiveMessage ─→ DB Insert
```

**활용 시나리오:**

- Frontend와 Backend를 **완전히 분리(Decouple)**
- Backend 처리 속도보다 요청이 빠를 때 **SQS를 Buffer**로 활용 → 데이터 유실 없음
- `ApproximateNumberOfMessages` CloudWatch Metric → ASG Scaling 트리거

---

## Amazon SQS - FIFO Queue

| 항목 | Standard Queue | FIFO Queue |
| --- | --- | --- |
| **순서** | Best-effort (미보장) | **엄격한 FIFO 보장** |
| **중복** | 가능 (at-least-once) | **정확히 1회 전송 (Exactly-once)** |
| **Throughput** | 무제한 | 배치 없음: **300 msg/s** / 배치: **3,000 msg/s** |
| **중복 제거** | - | **Deduplication ID** 기반 |
| **순서 그룹** | - | **Message Group ID** (필수 파라미터) |

> 📌 **Message Group ID**: 같은 Group ID 내 메시지는 순서 보장. 서로 다른 Group은 병렬 처리 가능.
> 

---

## Amazon SNS (Simple Notification Service)

### Pub/Sub 패턴

```
[Event Producer]
      │
      ▼
[SNS Topic]  ← 메시지 1회 발행
   │  │  │
   ▼  ▼  ▼
[Sub1] [Sub2] [Sub3] ...  ← 모든 Subscriber가 메시지 수신
```

| 항목 | 내용 |
| --- | --- |
| **Subscriber 수** | 토픽당 최대 **12,500,000**개 |
| **Topic 수** | 계정당 최대 **100,000**개 |
| **데이터 지속성** | ❌ 전달 실패 시 메시지 소멸 |

**지원 Subscriber 유형:**

- SQS, Lambda, Kinesis Data Firehose, HTTP/HTTPS Endpoint
- Email, SMS, Mobile Push Notification

---

### SNS 보안

SQS와 동일한 구조:

- In-flight: HTTPS / At-rest: KMS / Client-side: 고객 직접 처리
- **IAM Policies** + **SNS Access Policy** (Cross-Account, S3 등 서비스가 SNS에 쓰기 허용)

---

### SNS Message Filtering (메시지 필터링)

- JSON Policy로 각 Subscription이 수신할 메시지를 필터링
- 필터가 없는 Subscription → **모든 메시지 수신**

```
[SNS Topic: 주문 이벤트]
      │
      ├── [SQS: 주문완료 큐]    ← Filter: {"state": ["placed"]}
      ├── [SQS: 취소 큐]       ← Filter: {"state": ["cancelled"]}
      └── [Lambda: 전체 처리]   ← Filter 없음 (모든 메시지 수신)
```

---

### SNS + SQS Fan-Out 패턴

**문제**: S3 Event는 하나의 Rule에 하나의 대상만 설정 가능
**해결**: SNS Topic을 중간에 두고 여러 SQS Queue로 Fan-Out

```
[S3 Event] ─→ [SNS Topic] ─→ [SQS Queue A] → 썸네일 생성
                          ─→ [SQS Queue B] → 메타데이터 저장
                          ─→ [SQS Queue C] → 감사 로그
```

**Fan-Out 장점:**

- 완전한 Decoupling, 데이터 유실 없음
- SQS: 데이터 영속성, 지연 처리, 재시도 가능
- 나중에 Subscriber 추가 가능 (기존 아키텍처 변경 없이)
- Cross-Region Delivery 지원 (다른 리전의 SQS Queue에 전달 가능)

---

### SNS → Kinesis Data Firehose → S3 패턴

```
[서비스] → [SNS Topic] → [Kinesis Data Firehose] → [S3 / Redshift / OpenSearch]
```

SNS가 직접 지원하지 않는 대상(S3 등)에 데이터를 전달할 때 Firehose를 중간 단계로 활용.

---

### SNS FIFO Topic

- SQS FIFO와 유사한 기능:
    - **Message Group ID** 기반 순서 보장
    - Deduplication ID 또는 Content-Based Deduplication으로 중복 제거
- Subscriber: SQS Standard 또는 SQS FIFO Queue
- 제한된 Throughput

**SNS FIFO + SQS FIFO = Fan-Out + Ordering + Deduplication** 동시 달성

---

## Amazon Kinesis

### Kinesis 서비스 구성

| 서비스 | 역할 |
| --- | --- |
| **Kinesis Data Streams** | 실시간 스트리밍 데이터 수집 및 저장 |
| **Amazon Data Firehose** | 스트리밍 데이터를 S3/Redshift/OpenSearch 등으로 전달 |
| **Kinesis Data Analytics** | SQL로 스트리밍 데이터 실시간 분석 |

---

### Kinesis Data Streams

**데이터 흐름:**

```
[Producers: App, IoT, Click Stream] → [Kinesis Data Streams] → [Consumers: Lambda, ECS, App]
```

| 항목 | 내용 |
| --- | --- |
| **데이터 보존** | 기본 24시간, 최대 **365일** |
| **Replay** | ✅ 데이터 재처리(Replay) 가능 |
| **삭제** | ❌ 만료 전 삭제 불가 (Immutable) |
| **메시지 크기** | 최대 **10 MiB** (일반적으로 소규모 실시간 데이터) |
| **순서 보장** | 동일 **Partition ID** 내에서 순서 보장 |
| **암호화** | At-rest: KMS / In-flight: HTTPS |

---

### Kinesis Data Streams - Capacity Modes

| 모드 | Provisioned | On-Demand |
| --- | --- | --- |
| **용량 설정** | Shard 수 직접 지정 | 자동 |
| **입력 처리량** | Shard당 **1 MB/s** (or 1,000 records/s) | 기본 **4 MB/s** (or 4,000 records/s) |
| **출력 처리량** | Shard당 **2 MB/s** | 자동 |
| **스케일링** | 수동 | 최근 30일 피크 기반 자동 확장 |
| **과금** | Shard 시간당 | 시간당 + 데이터 GB당 |

---

### Amazon Data Firehose (구: Kinesis Data Firehose)

- **완전 관리형, Serverless, 자동 스케일링**
- **Near Real-Time** 전달 (버퍼링으로 인한 약간의 지연)
- 사용한 만큼 과금 (Pay for what you use)

**지원 대상 (Destinations):**

```
AWS: Amazon S3, Amazon Redshift, Amazon OpenSearch Service
3rd Party: Splunk, MongoDB, Datadog, NewRelic
Custom: HTTP Endpoint
```

**데이터 변환:**

- Lambda로 Custom 변환 (예: CSV → JSON)
- Parquet/ORC 변환, gzip/snappy 압축

---

### Kinesis Data Streams vs. Amazon Data Firehose

| 항목 | Kinesis Data Streams | Amazon Data Firehose |
| --- | --- | --- |
| **목적** | 스트리밍 데이터 **수집/저장** | 스트리밍 데이터 **전달/로드** |
| **실시간성** | **Real-time** | **Near Real-time** (버퍼링) |
| **관리** | Producer/Consumer 코드 직접 작성 | **완전 관리형** |
| **스케일링** | Provisioned 또는 On-Demand | **자동 스케일링** |
| **데이터 저장** | ✅ 최대 365일 | ❌ 저장 없음 |
| **Replay** | ✅ 가능 | ❌ 불가 |

---

## SQS vs. SNS vs. Kinesis — 최종 비교

| 항목 | SQS | SNS | Kinesis |
| --- | --- | --- | --- |
| **방식** | **Pull** (Consumer가 가져감) | **Push** (Subscriber에게 전달) | Pull / Enhanced Fan-Out (Push) |
| **데이터 지속성** | ✅ (소비 후 삭제) | ❌ (미전달 시 소멸) | ✅ (최대 365일) |
| **Replay** | ❌ | ❌ | ✅ |
| **Consumer/Subscriber 수** | Worker 수 제한 없음 | **12,500,000 Subscribers** | Shard별 분배 |
| **순서 보장** | FIFO Queue만 | SNS FIFO Topic만 | Partition ID별 |
| **처리량 설정** | 불필요 (자동) | 불필요 (자동) | Shard 직접 관리 (Provisioned) 또는 On-Demand |
| **Use Case** | 작업 큐, 비동기 처리 | 이벤트 알림, Fan-Out | 실시간 대용량 스트리밍, 분석 |

---

## Amazon MQ

### 왜 Amazon MQ인가?

- SQS/SNS는 **AWS 독점(Cloud-Native) 프로토콜** 사용
- 기존 온프레미스 애플리케이션은 **오픈 표준 프로토콜** 사용:
    - MQTT, AMQP, STOMP, OpenWire, WSS 등
- 클라우드 이전 시 애플리케이션 재설계 없이 기존 프로토콜 그대로 사용 가능

### Amazon MQ 특성

| 항목 | 내용 |
| --- | --- |
| **지원 브로커** | **RabbitMQ, ActiveMQ** |
| **스케일** | SQS/SNS만큼 확장되지 않음 (서버 기반) |
| **고가용성** | **Multi-AZ with Failover** |
| **기능** | Queue 기능(~SQS) + Topic 기능(~SNS) 동시 지원 |

> 📌 **선택 기준**: 새로운 애플리케이션 → **SQS/SNS** 사용. 기존 온프레미스 Message Broker 마이그레이션 → **Amazon MQ** 사용.
> 

---

## 📌 시험 자주 출제 포인트 총정리

| 포인트 | 내용 |
| --- | --- |
| SQS 메시지 최대 크기 | **256 KB** |
| SQS 기본 보존 기간 | **4일**, 최대 **14일** |
| SQS 한 번에 수신 최대 메시지 수 | **10개** |
| Visibility Timeout 기본값 | **30초** |
| Visibility Timeout 연장 방법 | **ChangeMessageVisibility API** |
| Long Polling 대기 시간 | **1~20초** (20초 권장) |
| FIFO Throughput | 배치 없음: **300 msg/s**, 배치: **3,000 msg/s** |
| FIFO 중복 제거 | **Deduplication ID** |
| FIFO 순서 그룹 | **Message Group ID** (필수) |
| SNS 최대 Subscriber 수 | **12,500,000개/토픽** |
| SNS 데이터 지속성 | ❌ 미전달 시 **소멸** |
| SNS Filter Policy 없으면 | **모든 메시지 수신** |
| Fan-Out 패턴 | **SNS → 여러 SQS** |
| S3 Event → 여러 대상 | **SNS Fan-Out** 사용 |
| Kinesis 데이터 보존 | 기본 24h, 최대 **365일** |
| Kinesis 메시지 삭제 | **불가** (만료 시까지 유지) |
| Kinesis 순서 보장 단위 | **Partition ID** |
| Provisioned Mode 입력 | Shard당 **1 MB/s** |
| Provisioned Mode 출력 | Shard당 **2 MB/s** |
| Kinesis Replay | ✅ 가능 (SQS/SNS는 불가) |
| Data Firehose 실시간성 | **Near Real-time** (버퍼링) |
| Data Firehose Replay | ❌ 불가 |
| Amazon MQ 지원 브로커 | **RabbitMQ, ActiveMQ** |
| Amazon MQ 선택 기준 | 기존 오픈 프로토콜(MQTT, AMQP 등) **마이그레이션** 시 |
| SQS ASG 트리거 지표 | **ApproximateNumberOfMessages** |

---

## 📚 참고 자료

- [Amazon SQS 공식 문서](https://docs.aws.amazon.com/sqs/)
- [Amazon SNS 공식 문서](https://docs.aws.amazon.com/sns/)
- [Amazon Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)
- [Amazon Data Firehose](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
- [Amazon MQ 공식 문서](https://docs.aws.amazon.com/amazon-mq/)