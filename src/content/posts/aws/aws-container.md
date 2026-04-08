---
title: 🐳 AWS Container Services
published: 2026-03-22
tags: [AWS, Cloud, Certificates]
category: AWS
series: AWS Certification
image: /images/aws.webp
draft: false
---
# 🐳 AWS Container Services

> Docker · ECS · ECR · EKS · App Runner · App2Container
> 
> 
> 컨테이너 기반 애플리케이션 배포 및 관리의 전체 스택
> 
---
## 목차
1. [Docker 기초](#docker-기초)
2. [AWS에서의 Container 관리 서비스](#aws에서의-container-관리-서비스)
3. [Amazon ECS (Elastic Container Service)](#amazon-ecs-elastic-container-service)
4. [Amazon ECR (Elastic Container Registry)](#amazon-ecr-elastic-container-registry)
5. [Amazon EKS (Elastic Kubernetes Service)](#amazon-eks-elastic-kubernetes-service)
6. [AWS App Runner](#aws-app-runner)
7. [AWS App2Container (A2C)](#aws-app2container-a2c)
8. [Container 서비스 선택 가이드](#container-서비스-선택-가이드)
9. [📌 시험 자주 출제 포인트 총정리](#-시험-자주-출제-포인트-총정리)
10. [📚 참고 자료](#-참고-자료)
---

## Docker 기초

### 개념

- 애플리케이션을 **Container**로 패키징하여 어떤 OS, 어떤 환경에서도 동일하게 실행
- **"Works on my machine" 문제 해결**: 개발/테스트/운영 환경 불일치 제거
- Use Cases: 마이크로서비스(Microservices), 온프레미스 앱의 AWS Lift-and-Shift

### Docker Image 저장소

| 저장소 | 설명 |
| --- | --- |
| **Docker Hub** | Public 저장소, OS/기술별 Base Image 제공 |
| **Amazon ECR (Private)** | AWS 관리형 Private 저장소 |
| **Amazon ECR Public Gallery** | Public Image 저장소 |

### Docker vs. Virtual Machine

```
Virtual Machine:
  [Hardware] → [Hypervisor] → [Guest OS 1] → [App A]
                           → [Guest OS 2] → [App B]

Docker:
  [Hardware] → [Host OS] → [Docker Engine] → [Container A: App A]
                                           → [Container B: App B]
```

- Docker: Host OS 커널 공유 → **가볍고 빠름**
- VM: 각각 완전한 OS 포함 → **무겁지만 강한 격리**

---

## AWS에서의 Container 관리 서비스

| 서비스 | 역할 | 인프라 관리 |
| --- | --- | --- |
| **Amazon ECS** | AWS 자체 컨테이너 플랫폼 | EC2(직접) 또는 Fargate(관리형) |
| **Amazon EKS** | 관리형 Kubernetes | EC2(직접) 또는 Fargate(관리형) |
| **AWS Fargate** | Serverless 컨테이너 플랫폼 | 인프라 불필요 |
| **Amazon ECR** | Container Image 저장소 | - |
| **AWS App Runner** | 완전 관리형 웹앱/API 배포 | 완전 자동 |

---

## Amazon ECS (Elastic Container Service)

### EC2 Launch Type

- ECS Cluster 위에 **직접 EC2 Instance를 프로비저닝/관리**
- 각 EC2 Instance에 **ECS Agent** 설치 필수 → ECS Cluster에 등록
- AWS는 Container의 시작/중지만 담당

```
[ECS Cluster]
  ├── EC2 Instance 1 (ECS Agent 실행)
  │     ├── Container A
  │     └── Container B
  └── EC2 Instance 2 (ECS Agent 실행)
        └── Container C
```

---

### Fargate Launch Type

- EC2 Instance 없이 **완전 Serverless**로 컨테이너 실행
- Task Definition만 생성하면 AWS가 알아서 컨테이너 실행
- 스케일링: Task 수만 늘리면 됨 — EC2 Instance 관리 불필요

```
[ECS Cluster]
  └── Fargate Task A  ← AWS가 인프라 자동 관리
  └── Fargate Task B
  └── Fargate Task C
```

---

### EC2 Launch Type vs. Fargate 비교

| 항목 | EC2 Launch Type | Fargate |
| --- | --- | --- |
| **인프라 관리** | 직접 EC2 프로비저닝/패치 | ❌ 불필요 |
| **비용** | EC2 비용 (장기 사용 시 유리) | Task별 CPU/RAM 사용량 과금 |
| **스케일링 복잡도** | EC2 + Task 두 레벨 관리 | Task만 관리 |
| **Spot Instance 활용** | ✅ 가능 | ✅ Fargate Spot 가능 |
| **맞춤 설정** | 높음 (EC2 직접 제어) | 낮음 |

---

### ECS IAM Roles

**두 가지 Role을 명확히 구분** — 시험 빈출

| Role | 사용 주체 | 목적 |
| --- | --- | --- |
| **EC2 Instance Profile** | ECS Agent (EC2 Launch Type만 해당) | ECS 서비스 API 호출, CloudWatch 로그 전송, ECR 이미지 Pull, Secrets Manager/SSM 접근 |
| **ECS Task Role** | 각 ECS Task | Task별로 다른 AWS 서비스 접근 권한 부여 |

```
EC2 Instance
  └── ECS Agent (EC2 Instance Profile 사용)
        └── ECS Task A (Task Role A: S3 Read 권한)
        └── ECS Task B (Task Role B: DynamoDB Write 권한)
```
:::tip
Task Role은 **Task Definition에서 정의**. EC2 Instance Profile과 독립적으로 동작.
:::
---

### ECS Load Balancer 통합

| Load Balancer | 지원 여부 | 권장 상황 |
| --- | --- | --- |
| **ALB** | ✅ | 대부분의 Use Case (권장) |
| **NLB** | ✅ | 고처리량/고성능, AWS Private Link 연계 |
| **CLB** | ✅ (비권장) | 고급 기능 없음, Fargate 미지원 |

---

### ECS - Data Volumes (EFS 연동)

- ECS Task에 **EFS 파일 시스템 마운트** 가능
- EC2 및 Fargate Launch Type 모두 지원
- 어떤 AZ에서 실행되든 동일한 EFS 데이터 공유

```
Fargate + EFS = 완전 Serverless 영구 스토리지
```
:::important
Amazon S3는 ECS Task에 **파일 시스템으로 마운트 불가** (S3는 Object Storage).
:::
---

### ECS Service Auto Scaling

ECS Service(Task 레벨) Auto Scaling과 EC2 Instance Auto Scaling은 **별개**임.

**ECS Service Auto Scaling 정책:**

| 정책 | 설명 |
| --- | --- |
| **Target Tracking** | 특정 CloudWatch 지표의 목표값 유지 |
| **Step Scaling** | CloudWatch Alarm 기반 단계별 조정 |
| **Scheduled Scaling** | 특정 일시에 미리 스케일링 |

**스케일링 지표:**

- ECS Service Average CPU Utilization
- ECS Service Average Memory Utilization
- ALB Request Count Per Target

---

### EC2 Launch Type — Auto Scaling EC2 Instances

ECS Service가 스케일 아웃되면 EC2 인스턴스도 함께 늘어야 함.

| 방법 | 설명 |
| --- | --- |
| **Auto Scaling Group** | CPU Utilization 기반 EC2 Scale Out |
| **ECS Cluster Capacity Provider** | Task 실행에 필요한 용량 부족 시 자동으로 EC2 추가 (ASG와 연동) |

:::tip
**Capacity Provider가 권장됨**: Task 수요에 따라 EC2를 자동 프로비저닝.
:::
---

### ECS 이벤트 기반 아키텍처

**Pattern 1: S3 → EventBridge → ECS Task (On-demand)**

```
[사용자] → S3 Upload
              │
              ▼ EventBridge Rule
         [ECS Task 실행] → S3에서 파일 가져오기 → DynamoDB 저장
         (ECS Task Role: S3 + DynamoDB 권한)
```

**Pattern 2: EventBridge Schedule → ECS Task (Batch)**

```
[EventBridge: 매 1시간] → ECS Task 실행 → S3 Batch Processing
```

---

## Amazon ECR (Elastic Container Registry)

| 항목 | 내용 |
| --- | --- |
| **저장소 유형** | Private Repository / Public Repository (ECR Public Gallery) |
| **통합** | ECS와 완전 통합, 백엔드는 Amazon S3 |
| **접근 제어** | IAM 기반 (권한 오류 → IAM Policy 확인) |
| **부가 기능** | 이미지 취약점 스캔(Vulnerability Scanning), 버저닝, Image Tags, Lifecycle 정책 |

```
[Docker Build] → [docker push] → [ECR] → [ECS Pull] → [Container 실행]
                                     ↑
                              IAM 권한 필요
                              (Pull 시 ECR에 대한 읽기 권한)
```

---

## Amazon EKS (Elastic Kubernetes Service)

### 개요

- **완전 관리형 Kubernetes Cluster** 서비스
- **Kubernetes**: 컨테이너 자동 배포, 스케일링, 관리를 위한 오픈소스 시스템
- ECS와 유사한 목적이지만 **다른 API** (Kubernetes API)
- **Cloud-agnostic** → 다른 클라우드/온프레미스의 Kubernetes와 호환

:::tip
- **선택 기준**: 
   - 이미 Kubernetes를 사용 중이거나 멀티 클라우드 전략 → **EKS**. 
   - AWS에서 새로 시작 → **ECS**가 더 단순.
:::

---

### EKS Node Types

| 유형 | 설명 | 관리 주체 |
| --- | --- | --- |
| **Managed Node Groups** | EKS가 EC2 Node 생성 및 관리, ASG 자동 관리 | AWS |
| **Self-Managed Nodes** | 사용자가 EC2 생성 후 EKS Cluster에 등록 | 사용자 |
| **AWS Fargate** | Serverless, Node 관리 불필요 | AWS |

Managed/Self-Managed: On-Demand 또는 **Spot Instance** 모두 지원

---

### EKS Data Volumes

- EKS Cluster에 **StorageClass Manifest** 지정 필요
- **CSI (Container Storage Interface)** 드라이버 사용

| 스토리지 | Fargate 지원 |
| --- | --- |
| **Amazon EBS** | ❌ |
| **Amazon EFS** | ✅ |
| **FSx for Lustre** | ❌ |
| **FSx for NetApp ONTAP** | ❌ |

---

## AWS App Runner

- 소스 코드 또는 Container Image로부터 **웹앱/API를 완전 자동으로 배포**
- 인프라 경험 불필요 — 빌드, 배포, 스케일링, 로드 밸런싱, 암호화 모두 자동
- VPC 접근 지원 → DB, Cache, 메시지 큐 연결 가능

```
[Source Code 또는 Container Image]
          │
          ▼
   [App Runner]
   ├── 자동 빌드
   ├── 자동 배포
   ├── 자동 스케일링
   ├── 로드 밸런서
   └── HTTPS 자동 설정
```

**Use Cases:** 웹앱, API, 마이크로서비스, 빠른 프로덕션 배포

---

## AWS App2Container (A2C)

- **Java / .NET 웹 애플리케이션**을 Docker Container로 변환하는 **CLI 도구**
- 온프레미스(베어 메탈, VM) 또는 다른 클라우드에서 실행 중인 앱을 AWS로 Lift-and-Shift
- **코드 변경 없이** 레거시 앱 현대화(Modernization) 가속

### A2C 프로세스

```
1. Discover & Analyze
   앱 인벤토리 생성 + 런타임 의존성 분석

2. Extract & Containerize
   앱 + 의존성 추출 → Docker Image 생성

3. Create Deployment Artifacts
   ECS Task Definition / EKS Pod Definition 생성
   CloudFormation Template (인프라: 컴퓨팅, 네트워크) 생성
   CI/CD Pipeline 생성

4. Deploy to AWS
   Docker Image → ECR 등록
   → ECS / EKS / App Runner 배포
```

---

## Container 서비스 선택 가이드

```
컨테이너 이미지 저장이 필요한가?
└── 예 → Amazon ECR

어떤 컨테이너 오케스트레이터를 사용할 것인가?
├── AWS Native, 간단한 설정 원함 → Amazon ECS
│     ├── 서버 관리 직접 하고 싶음 → EC2 Launch Type
│     └── Serverless 원함          → Fargate Launch Type
│
└── Kubernetes 사용 중 or 멀티 클라우드 → Amazon EKS
      ├── 서버 관리 직접 하고 싶음 → Managed/Self-Managed Nodes
      └── Serverless 원함          → Fargate

코드만 있고 인프라는 전혀 신경 쓰기 싫음 → App Runner

기존 Java/.NET 앱을 컨테이너로 이전 → App2Container
```

---

## 📌 시험 자주 출제 포인트 총정리

| 포인트 | 내용 |
| --- | --- |
| EC2 Instance Profile vs Task Role | Instance Profile: ECS Agent용 / Task Role: 각 Task용, **Task Definition에서 정의** |
| ECS + EFS | 멀티 AZ 공유 영구 스토리지, Fargate와 함께 **Serverless 영구 스토리지** |
| S3 ECS Task 마운트 | **불가** (Object Storage는 파일 시스템 마운트 불가) |
| ECS Capacity Provider | Task 수요에 따라 **EC2 자동 프로비저닝** |
| ECR 권한 오류 | **IAM Policy** 확인 |
| ECR 백엔드 | **Amazon S3** |
| EKS Kubernetes 특징 | **Cloud-agnostic**, 오픈소스 |
| EKS Fargate EFS 지원 | ✅ (EBS는 Fargate에서 미지원) |
| App Runner | 소스코드/Container Image → **인프라 없이 자동 배포** |
| App2Container 대상 | **Java, .NET** 앱 |
| App2Container 결과물 | Docker Image(ECR) + Task/Pod Definition + **CloudFormation Template** + CI/CD Pipeline |
| Fargate Auto Scaling | EC2 관리 불필요, **Task 수만 조절** |
| ALB vs NLB (ECS) | 대부분 **ALB** / 초고성능 또는 Private Link → **NLB** |
| CLB + Fargate | **미지원** |
| EKS Spot Instance | Managed/Self-Managed Node에서 **지원** |

---

## 📚 참고 자료

- [Amazon ECS 공식 문서](https://docs.aws.amazon.com/ecs/)
- [Amazon EKS 공식 문서](https://docs.aws.amazon.com/eks/)
- [Amazon ECR 공식 문서](https://docs.aws.amazon.com/ecr/)
- [AWS Fargate 공식 문서](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)
- [AWS App Runner 공식 문서](https://docs.aws.amazon.com/apprunner/)
- [AWS App2Container 공식 문서](https://docs.aws.amazon.com/app2container/)