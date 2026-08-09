# Ticket Wave — Infra
### EKS 기반 고가용성 티켓팅 서비스

> 이 저장소는 멋쟁이사자처럼 AWS 기반 DevOps 엔지니어 과정 팀 프로젝트(5인)의 fork 입니다. <br>
> 본 README는 담당 영역을 중심으로 재작성했습니다. <br>

<br>

원본 저장소
* **App 레포** : [team5-ticket-app](https://github.com/CLD-05/team5-ticket-app)
* **Config 레포** : [team5-ticket-config](https://github.com/CLD-05/team5-ticket-config)
* **Infra 레포** : [team5-ticket-infra](https://github.com/CLD-05/team5-ticket-infra) <br>

<br>

## 프로젝트 개요

예매 오픈 시점에 트래픽이 순간적으로 집중되는 티켓팅 서비스 특성상, 

단순 서버 배포가 아니라 트래픽 완충·비동기 처리·데이터 정합성·오토스케일링·운영 관측성을 함께 고려한 인프라가 필요했습니다. 

이를 EKS 기반으로 구축하고, dev/prod 환경을 분리해 Terraform으로 관리했습니다.

<br>

- 기간 : 2026.06.02 ~ 2026.07.08
- 팀 구성 : 5인
- 담당 : EKS 접근 통제 체계 구축 · 실시간 좌석 API 개발

<br>

## 아키텍처

### Dev 아키텍처
<img width="1560" height="1611" alt="image" src="https://github.com/user-attachments/assets/9546c58b-5cd4-4d46-ac32-9a695be87b31" />

### Prod 아키텍처
<img width="1994" height="1728" alt="image" src="https://github.com/user-attachments/assets/3717d93b-85d0-40c9-b8bd-1e6f41071be2" />

<br>
<br>

## 내가 담당한 부분

### EKS 접근 통제 체계
- EKS AccessEntry 기반 클러스터 접근 권한 관리
  
  → aws-auth ConfigMap 대신 선택한 이유 : 직접 편집 시 오타·권한 누락으로 클러스터 접근 불가 장애가 발생할 수 있어, AWS 관리형 API인 AccessEntry로 대체해 설정 오류 위험 제거
- SSM Session Manager 기반 Bastion 구성 (SSH 포트 미개방)
- Pod Identity로 Pod 단위 최소 권한 IAM Role 매핑
- 관련 코드 : `modules/eks/access_entry.tf`, `modules/eks/iam.tf`

<br>

**[실시간 좌석 API 개발](https://github.com/its-jihyeon/ticket-wave-app)**

<br>

## 기술 스택
AWS (EKS, RDS, IAM, SSM) · Terraform · Redis · Java
