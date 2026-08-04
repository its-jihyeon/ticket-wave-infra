# Ticket Wave — Infra
### EKS 기반 고가용성 티켓팅 서비스

> 이 저장소는 멋쟁이사자처럼 AWS DevOps 과정 팀 프로젝트(5인)의 fork 입니다.
> 원본 저장소 : [팀 레포 URL]
> 본 README는 담당 영역을 중심으로 재작성했습니다.

* 📦 **App 레포** : [team5-ticket-app](https://github.com/CLD-05/team5-ticket-app)
* ⚙️ **Config 레포** : [team5-ticket-config](https://github.com/CLD-05/team5-ticket-config)
* 🛠️ **Infra 레포** : [team5-ticket-infra](https://github.com/CLD-05/team5-ticket-infra)

# Ticket Wave

## 1. 개요

이 저장소는 Team5 티켓팅 플랫폼의 AWS 인프라를 Terraform으로 관리하는 IaC 저장소입니다.

티켓팅 서비스는 특정 시간에 트래픽이 급격히 몰리는 특성이 있으므로, 단순한 서버 배포가 아니라 트래픽 완충, 비동기 처리, 데이터 정합성, 오토스케일링, 운영 관측성을 함께 고려해야 합니다.

이를 위해 본 인프라는 아래 목표를 기준으로 설계했습니다.

### 주요 목표

- EKS 기반 애플리케이션 실행 환경 구성
- RDS, Redis, SQS를 활용한 예매 처리 안정성 확보
- dev/prod 환경 분리
- Terraform 기반 인프라 형상 관리
- GitHub Actions OIDC 기반 Terraform CI/CD 구성
- ArgoCD 기반 GitOps 배포 구조 지원
- 운영 환경 기준 고가용성, 보안, 관측성 확장 가능 구조 확보

---

## 2. 아키텍처

전체 구조는 AWS 위에 EKS를 중심으로 구성되어 있습니다.

### Dev 아키텍처
<img width="1560" height="1611" alt="image" src="https://github.com/user-attachments/assets/9546c58b-5cd4-4d46-ac32-9a695be87b31" />

### Prod 아키텍처
<img width="1994" height="1728" alt="image" src="https://github.com/user-attachments/assets/3717d93b-85d0-40c9-b8bd-1e6f41071be2" />

### 핵심 설계 방향

- Web API Pod와 Booking Worker Pod를 분리하여 요청 처리와 예매 확정 처리를 분리
- SQS를 통해 예매 확정 요청을 비동기 처리하여 DB 부하 완화
- Redis를 사용해 대기열, 좌석 선점, 동시성 제어를 처리
- RDS Proxy를 통해 DB Connection Pool 압박 완화
- Secrets Manager와 External Secrets Operator를 사용해 애플리케이션 설정을 Kubernetes Secret으로 동기화
- ArgoCD를 통해 Kubernetes Manifest를 GitOps 방식으로 적용
- prod 환경은 3AZ, Multi NAT, Multi-AZ RDS, Redis Replication Group 기반으로 고가용성 방향 적용
- prod 환경에서는 Route53과 WAF를 통해 외부 진입 도메인과 웹 요청 필터링을 구성
- S3는 공연 포스터 이미지를 저장하며, prod 환경에서는 CloudFront를 통해 정적 자산을 캐싱하고 제공

---

## 3. 저장소 구조

```text
team5-ticket-infra/
├── bootstrap-backend/
│   └── Terraform state backend용 S3 bucket 생성
│
├── modules/
│   ├── network/        # VPC, Subnet, NAT, Routing
│   ├── bastion/        # SSM Bastion
│   ├── eks/            # EKS Cluster, Node Group, IRSA
│   ├── database/       # RDS MySQL, RDS Proxy, Read Replica
│   ├── elasticache/    # Redis
│   ├── sqs/            # Booking Queue, DLQ
│   ├── ecr/            # ECR Repository
│   ├── secrets/        # Secrets Manager
│   ├── s3/             # Poster Image Bucket, CloudFront
│   ├── monitoring/     # KEDA / YACE / Cluster Autoscaler IAM
│   ├── waf/            # AWS WAF
│   ├── route53/        # Route53 Hosted Zone / DNS Record
│   └── github_oidc/    # GitHub Actions OIDC Role
│
├── envs/
│   ├── dev/
│   │   ├── infra/
│   │   └── platform-addons/
│   │
│   └── prod/
│       ├── infra/
│       └── platform-addons/
│
└── test/
    └── k6/             # 부하 테스트 시나리오
```

Repository는 `modules`와 `envs`로 분리되어 관리됩니다.

| Directory | Description |
| --- | --- |
| `modules` | 재사용 가능한 Terraform Module |
| `envs/dev` | 개발 및 통합 검증 환경 |
| `envs/prod` | 운영 환경 |
| `platform-addons` | ArgoCD Bootstrap 및 Kubernetes Provider 기반 리소스 |
| `test/k6` | 부하 테스트 시나리오 |

---

## 4. 환경 구성

### dev

개발 및 통합 검증을 위한 비용 효율 중심 환경입니다.

- 빠른 검증과 비용 절감 우선
- prod와 동일한 서비스 흐름 유지
- RDS, Redis, SQS, ECR, Secrets Manager, EKS 구성
- ALB DNS 기반 접근
- Single NAT, Single-AZ RDS, Single Redis Node 중심 구성

### prod

운영 환경 기준으로 구성됩니다.

- 고가용성 및 안정성 우선
- EKS Private Endpoint 사용
- Bastion/SSM 기반 클러스터 접근
- Route53 기반 도메인 접근
- Multi NAT, Multi-AZ RDS, Read Replica, Redis Replication Group 구성
- WAF를 통한 ALB 진입 요청 필터링
- CloudFront를 통한 S3 정적 자산 캐싱

---

## 5. 주요 인프라 구성 요소

### Network

VPC는 Public, Private, Database Subnet으로 분리합니다.

**Public Subnet**

- ALB
- NAT Gateway
- Bastion

**Private Subnet**

- EKS Worker Node
- Application Pod

**Database Subnet**

- RDS
- RDS Proxy
- Redis

dev는 Single NAT를, prod는 Multi NAT를 사용합니다.

### Compute

애플리케이션은 EKS에서 실행됩니다.

- EKS Node Group 구성
- Kubernetes Add-on 및 Application Manifest는 ArgoCD 관리
- prod는 Private Endpoint 사용

### Database

RDS MySQL은 최종 예매 데이터의 정합성을 담당합니다.

- RDS Proxy를 통한 DB Connection 관리
- prod Multi-AZ 구성
- prod Read Replica 구성
- Terraform Random Password 생성
- Secrets Manager 연동

### Cache

Redis는 티켓팅 서비스의 동시성 제어에 사용됩니다.

주요 사용처는 다음과 같습니다.

- 대기열
- Queue Token
- 좌석 임시 선점
- Redisson 기반 분산 락

dev에서는 Single Node Redis를 사용하고, prod에서는 Redis Replication Group을 사용합니다.

### Messaging

SQS는 예매 확정 요청을 비동기로 처리하기 위한 메시징 계층입니다.

```
Web API
    ↓
SQS
    ↓
Worker Pod
    ↓
RDS
```

### Security & Secrets

애플리케이션 런타임 설정은 AWS Secrets Manager에 저장합니다.

- Secrets Manager → External Secrets Operator → Kubernetes Secret → Pod 환경변수 흐름으로 민감 정보 주입
- GitHub OIDC 기반 IAM Role Assume 방식 사용
- AWS Access Key 미사용

### Observability & Autoscaling

지원 구성은 다음과 같습니다.

- Prometheus
- Grafana
- KEDA
- Cluster Autoscaler
- YACE
- CloudWatch
- WAF Logging

Terraform은 KEDA, YACE, Cluster Autoscaler 등에 필요한 IAM/IRSA를 관리하며, 실제 Kubernetes Add-on 설치와 설정은 Config Repository와 ArgoCD가 담당합니다.

---

## 6. 배포 구조

### Core Infra

`envs/{env}/infra`

생성 대상은 다음과 같습니다.

- VPC
- EKS
- Bastion
- RDS
- Redis
- SQS
- ECR
- Secrets Manager
- S3 / CloudFront
- WAF / Route53
- GitHub OIDC Role
- Monitoring IRSA

### Platform Add-ons

`envs/{env}/platform-addons`

ArgoCD Bootstrap을 담당합니다.

Repository 역할은 다음과 같습니다.

```
infra repo
    ↓
AWS Infrastructure
ArgoCD Bootstrap

config repo
    ↓
Kubernetes Add-on
Application Manifest

app repo
    ↓
Application Source
Docker Build
ECR Push
```

Terraform과 ArgoCD의 관리 영역을 분리하여 동일 리소스를 중복 관리하지 않습니다.

Terraform은 AWS 인프라와 ArgoCD Bootstrap을 담당하고, 실제 애플리케이션 및 Add-on Manifest는 Config Repository에서 관리합니다.

---

## 7. Terraform CI/CD

Workflow는 다음 파일에서 관리합니다.

```
.github/workflows/terraform.yml
```

### Pull Request

```
PR
 ↓
Changed Path Detection
 ↓
terraform fmt
 ↓
terraform init
 ↓
terraform validate
 ↓
terraform plan
 ↓
PR Comment
```

Plan 대상은 다음과 같습니다.

- `envs/dev/infra`
- `envs/prod/infra`

### Main Merge

```
main merge
    ↓
Changed Path Detection
    ↓
dev infra
    ↓
Auto Apply

prod infra
    ↓
GitHub Environment Approval
    ↓
Apply
```

### tfvars

GitHub Secrets는 다음 값을 사용합니다.

```
DEV_TERRAFORM_TFVARS
PROD_TERRAFORM_TFVARS
```

실제 `terraform.tfvars`는 Git에 커밋하지 않습니다. CI/CD에서는 GitHub Secrets의 `DEV_TERRAFORM_TFVARS`, `PROD_TERRAFORM_TFVARS` 값을 사용해 실행 시점에 생성합니다.

---

## 8. 운영 확인 항목

운영 시 주요 확인 사항은 다음과 같습니다.

- Terraform Plan 변경 사항 확인
- EKS Node Group 상태 확인
- RDS / Redis / SQS / ECR / Secrets Manager Output 확인
- External Secrets 동기화 확인
- ArgoCD Application Sync 상태 확인
- KEDA / Cluster Autoscaler / Prometheus Add-on 상태 확인

prod는 Private Endpoint를 사용하므로 Bastion/SSM을 통한 접근을 전제로 합니다.
