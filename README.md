# KYEOL 프로젝트 문서

> **KYEOL** - Saleor 기반 B2C 전자상거래 플랫폼
> AWS EKS에서 운영되는 클라우드 네이티브 아키텍처

---

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [아키텍처 핵심 원칙](#아키텍처-핵심-원칙)
3. [환경 구성](#환경-구성)
4. [책임 분리 철학](#책임-분리-철학)
5. [문서 네비게이션](#문서-네비게이션)
6. [관점별 가이드](#관점별-가이드)
7. [레포지토리](#레포지토리)
8. [FAQ](#faq)
9. [디렉토리 구조](#디렉토리-구조)

---

## 프로젝트 개요

**KYEOL**은 오픈소스 전자상거래 플랫폼인 [Saleor](https://saleor.io/)를 기반으로 구축된 B2C 서비스입니다.

### 기술 스택
- **Backend**: Saleor Core (Django, GraphQL API)
- **Frontend**: Saleor Dashboard (React), Saleor Storefront (Next.js)
- **Infrastructure**: AWS EKS (Kubernetes), Terraform
- **Database**: Amazon RDS for PostgreSQL
- **Cache**: Amazon ElastiCache for Redis (Valkey)
- **Storage**: Amazon S3 (미디어 파일)
- **CDN**: CloudFront
- **GitOps**: ArgoCD + Kustomize
- **Observability**: CloudWatch Logs + CloudTrail + Grafana

### 주요 특징
- **클라우드 네이티브**: Kubernetes 기반 컨테이너 오케스트레이션
- **멀티 환경**: MGMT, DEV, STAGE, PROD 환경 분리
- **IaC**: Terraform으로 인프라 코드화
- **GitOps**: Git을 단일 진실 공급원(Single Source of Truth)으로 사용
- **보안**: VPC 격리, IAM 역할 기반 접근 제어, WAF, CloudTrail
- **확장성**: Auto Scaling, Multi-AZ 배포

---

## 아키텍처 핵심 원칙

### ✅ 해야 할 것

1. **ALB는 AWS Load Balancer Controller로 생성**
   - Kubernetes Ingress 리소스 생성 시 자동으로 ALB 프로비저닝
   - 동적 타겟 그룹 관리 및 Health Check

2. **GitOps 우선**
   - 모든 Kubernetes 리소스는 Git 레포지토리에서 관리
   - ArgoCD를 통한 자동 동기화
   - `kubectl apply` 직접 사용 최소화

3. **로그와 메트릭 분리**
   - 로그: Fluent Bit → CloudWatch Logs
   - 메트릭: CloudWatch Container Insights
   - 통합 UI: Grafana

4. **IRSA 사용**
   - Pod별 IAM 역할 할당
   - S3, Secrets Manager 등 AWS 서비스 접근 시 IRSA 사용

5. **환경별 격리**
   - VPC, EKS 클러스터, RDS, Redis 모두 환경별 독립 구성
   - STAGE 환경에서 충분한 검증 후 PROD 배포

### ❌ 하지 말아야 할 것

1. **Terraform으로 ALB 직접 생성 금지**
   - ALB는 반드시 AWS Load Balancer Controller + Ingress로 생성
   - Terraform으로 생성 시 타겟 그룹 동적 관리 불가

2. **Prometheus 사용 금지**
   - CloudWatch Container Insights + Grafana 사용
   - 추가 메트릭 수집 시스템 불필요

3. **프로덕션 직접 수정 금지**
   - 모든 변경은 DEV → STAGE → PROD 순서로 진행
   - 긴급 상황에서도 GitOps 프로세스 준수

4. **민감 정보 평문 저장 금지**
   - Kubernetes Secret은 반드시 암호화
   - AWS Secrets Manager 사용 권장

5. **kubectl apply 직접 사용 최소화**
   - GitOps 워크플로우 우선
   - 디버깅 목적으로만 제한적 사용

---

## 환경 구성

| 환경 | VPC CIDR | 용도 | EKS 클러스터 | RDS | Redis |
|------|----------|------|--------------|-----|-------|
| **MGMT** | 10.0.0.0/16 | 관리/도구 (ArgoCD, Bastion) | kyeol-mgmt-eks | - | - |
| **DEV** | 10.10.0.0/16 | 개발/테스트 | kyeol-dev-eks | db.t3.medium | cache.t3.micro |
| **STAGE** | 10.20.0.0/16 | 스테이징/QA | kyeol-stage-eks | db.t3.large | cache.t3.small |
| **PROD** | 10.30.0.0/16 | 프로덕션 | kyeol-prod-eks | db.r5.xlarge | cache.r5.large |

### 리전
- **ap-northeast-2** (서울)

### 네임스페이스 구조
```
EKS 클러스터
├── kube-system (시스템 컴포넌트)
├── argocd (GitOps - MGMT만)
├── saleor (Saleor Core + Worker + Redis)
├── dashboard (Saleor Dashboard)
└── storefront (Saleor Storefront)
```

---

## 책임 분리 철학

### Terraform (인프라 레이어)
**관리 대상**:
- VPC, 서브넷, 라우팅 테이블, NAT Gateway
- EKS 클러스터, 노드 그룹
- RDS, ElastiCache
- S3, ECR
- IAM 역할, 정책
- WAF, CloudFront, Route53
- CloudTrail, CloudWatch

**관리하지 않는 대상**:
- ALB (AWS Load Balancer Controller가 관리)
- Kubernetes 리소스 (GitOps가 관리)

### GitOps (플랫폼 + 애플리케이션 레이어)
**관리 대상**:
- Kubernetes 매니페스트 (Deployment, Service, Ingress, ConfigMap, Secret)
- Helm 차트
- Kustomize 오버레이

**워크플로우**:
```
코드 변경 → Git Push → ArgoCD 감지 → 자동 동기화 → 클러스터 배포
```

---

## 문서 네비게이션

### 📚 Phase별 가이드

#### Phase 1: 인프라 구축
**문서**: [runbook/runbook-infra.md](runbook/runbook-infra.md)

**다루는 내용**:
- Terraform으로 VPC, EKS, RDS, Redis 구축
- Bootstrap (S3 백엔드, 상태 잠금)
- CloudFront, WAF, CloudTrail 설정
- 환경별 인프라 확장 (MGMT → DEV → STAGE → PROD)
- RDS 백업/복원, NAT Gateway 관리
- 비용 모니터링 및 최적화

**언제 읽어야 하는가**:
- 새 환경 구축 시
- 인프라 리소스 추가/수정 시
- RDS/Redis 백업/복원 시
- 비용 이슈 발생 시

---

#### Phase 2: 플랫폼 구성
**문서**: [runbook/runbook-platform.md](runbook/runbook-platform.md)

**다루는 내용**:
- AWS Load Balancer Controller 설치 및 운영
- ExternalDNS로 Route53 자동 관리
- Fluent Bit으로 로그 수집
- ArgoCD 설치 및 애플리케이션 배포
- IRSA 구성 (S3, Secrets Manager 접근)
- HPA (Horizontal Pod Autoscaler) 설정

**언제 읽어야 하는가**:
- EKS 클러스터 초기 설정 시
- Ingress 생성 시 ALB가 자동 생성되지 않을 때
- ArgoCD 앱 배포/동기화 시
- 로그 확인이 필요할 때
- Pod Auto Scaling 설정 시

---

#### Phase 3: 애플리케이션 운영
**문서**: [runbook/runbook-ops.md](runbook/runbook-ops.md)

**다루는 내용**:
- Saleor Core, Dashboard, Storefront 배포
- 데이터베이스 마이그레이션
- 환경 변수 관리 (ConfigMap, Secret)
- 카탈로그 시딩 (`scripts/seed_kyeol_catalog.py`)
- S3 미디어 업로드 (`scripts/upload_images_to_s3.py`)
- 권한 체크 (`scripts/check_saleor_permissions.py`)
- API 토큰 관리

**언제 읽어야 하는가**:
- Saleor 애플리케이션 배포 시
- DB 마이그레이션 실행 시
- 제품 카탈로그 초기화 시
- 이미지 업로드 문제 발생 시
- API 인증 문제 발생 시

---

#### 장애 대응
**문서**: [troubleshooting.md](troubleshooting.md)

**다루는 내용**:
- **인프라**: Terraform apply 실패, RDS 연결 실패, NAT Gateway 장애
- **플랫폼**: ALB 미생성, Pod ImagePullBackOff, ArgoCD 동기화 실패
- **애플리케이션**: Saleor 500 에러, DB 마이그레이션 실패, 이미지 업로드 실패
- **네트워크**: CloudFront 503 에러, DNS 미해결, 보안 그룹 차단

**구조**: 증상 → 원인 → 확인 → 해결 → 재발 방지

**언제 읽어야 하는가**:
- 배포 후 서비스가 정상 동작하지 않을 때
- CloudWatch 알람 발생 시
- 사용자로부터 오류 신고 접수 시
- 특정 증상과 일치하는 케이스 검색 시

---

#### 유틸리티 스크립트
**문서**: [scripts/README.md](../scripts/README.md)

**다루는 내용**:
- `check_saleor_permissions.py`: Saleor API 권한 체크
- `seed_kyeol_catalog.py`: 제품 카탈로그 초기 데이터 삽입
- `upload_images_to_s3.py`: 제품 이미지 S3 업로드

**언제 읽어야 하는가**:
- 제품 카탈로그 초기화 시
- 이미지 대량 업로드 시
- API 권한 문제 디버깅 시

---

## 관점별 가이드

### 🔒 보안 관점
1. **네트워크 격리**
   - VPC별 독립 구성
   - Private 서브넷에 애플리케이션 배포
   - Public 서브넷에는 NAT Gateway, Bastion만 배치

2. **접근 제어**
   - IAM 역할 기반 권한 관리
   - IRSA로 Pod별 최소 권한 부여
   - Security Group으로 네트워크 제어

3. **감사 및 로깅**
   - CloudTrail로 모든 API 호출 기록
   - CloudWatch Logs로 애플리케이션 로그 수집
   - 로그 보관 기간: DEV(7일), STAGE(30일), PROD(90일)

4. **데이터 보호**
   - RDS 암호화 (at-rest)
   - S3 버킷 암호화
   - Kubernetes Secret 암호화 권장

5. **WAF**
   - CloudFront에 AWS WAF 연동
   - 기본 보호 규칙 활성화 (SQL Injection, XSS)

**참고**: [runbook-infra.md](runbook/runbook-infra.md) - WAF 설정, [runbook-ops.md](runbook/runbook-ops.md) - Secret 관리

---

### 💰 비용 관점
1. **리소스 최적화**
   - DEV: 최소 사양 (db.t3.medium, cache.t3.micro)
   - STAGE: 중간 사양 (db.t3.large, cache.t3.small)
   - PROD: 고사양 (db.r5.xlarge, cache.r5.large)

2. **자동 스케일링**
   - EKS 노드 그룹 Auto Scaling
   - HPA로 Pod 자동 증감
   - 야간 시간대 리소스 축소 고려

3. **비용 모니터링**
   - AWS Cost Explorer로 월별 추적
   - 태그 기반 환경별 비용 분리
   - Unused 리소스 정기 점검

4. **주요 비용 요소**
   - EKS 클러스터 ($0.10/시간)
   - EC2 인스턴스 (노드 그룹)
   - RDS 인스턴스 + 스토리지
   - NAT Gateway ($0.045/시간 + 데이터 전송)
   - CloudFront 데이터 전송

**참고**: [runbook-infra.md](runbook/runbook-infra.md) - 비용 모니터링 섹션

---

### 🛠️ 운영 관점
1. **배포 워크플로우**
   ```
   코드 수정 → Git Push → ArgoCD 자동 동기화 → 배포 → Health Check
   ```

2. **환경별 배포 순서**
   ```
   DEV (기능 개발) → STAGE (통합 테스트 + QA) → PROD (프로덕션)
   ```

3. **롤백 전략**
   - ArgoCD: 이전 Revision으로 Rollback
   - Kubernetes: `kubectl rollout undo`
   - RDS: 자동 백업 스냅샷에서 복원

4. **정기 작업**
   - 일일: CloudWatch 알람 확인, 로그 모니터링
   - 주간: RDS 백업 확인, 불필요한 리소스 정리
   - 월간: 비용 리포트 검토, 보안 패치 적용

5. **긴급 대응**
   - [troubleshooting.md](troubleshooting.md)에서 증상 검색
   - CloudWatch Logs에서 에러 로그 확인
   - ArgoCD UI에서 배포 상태 확인
   - 필요 시 롤백 실행

**참고**: [runbook-platform.md](runbook/runbook-platform.md), [runbook-ops.md](runbook/runbook-ops.md)

---

## 레포지토리

| 레포지토리 | 용도 | 기술 스택 |
|-----------|------|----------|
| **kyeol-infra-terraform** | 인프라 코드 (IaC) | Terraform, AWS |
| **kyeol-app-gitops** | Kubernetes 매니페스트 | Kustomize, Helm |
| **saleor** | Saleor Core (백엔드) | Django, GraphQL |
| **kyeol-dashboard** | 관리자 대시보드 | React, TypeScript |
| **kyeol-storefront** | 고객용 프론트엔드 | Next.js, TypeScript |

---

## FAQ

### Q1. ALB가 자동 생성되지 않아요
**A**: AWS Load Balancer Controller가 정상 동작하는지 확인하세요.
```bash
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```
Ingress 리소스의 어노테이션도 확인하세요:
```yaml
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/scheme: internet-facing
```
**참고**: [runbook-platform.md](runbook/runbook-platform.md) - AWS Load Balancer Controller 섹션, [troubleshooting.md](troubleshooting.md) - Case 1

---

### Q2. Terraform apply 시 ALB 관련 에러가 발생해요
**A**: Terraform으로 ALB를 직접 생성하려고 하지 않았나요? ALB는 AWS Load Balancer Controller + Ingress로만 생성해야 합니다.

**금지**:
```hcl
resource "aws_lb" "app_alb" {  # ❌ 절대 금지
  name = "kyeol-alb"
  ...
}
```

**올바른 방법**: Kubernetes Ingress 리소스 생성
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: saleor-ingress
  annotations:
    kubernetes.io/ingress.class: alb
```

**참고**: [runbook-infra.md](runbook/runbook-infra.md) - ALB 생성 금지 경고

---

### Q3. Saleor에서 이미지 업로드가 안 돼요
**A**: S3 버킷 정책과 IRSA 설정을 확인하세요.

1. **S3 버킷 CORS 설정 확인**:
```json
[
  {
    "AllowedOrigins": ["*"],
    "AllowedMethods": ["GET", "POST", "PUT"],
    "AllowedHeaders": ["*"]
  }
]
```

2. **IRSA 역할 확인**:
```bash
kubectl describe sa saleor -n saleor | grep eks.amazonaws.com/role-arn
```

**참고**: [runbook-ops.md](runbook/runbook-ops.md) - S3 미디어 관리 섹션, [troubleshooting.md](troubleshooting.md) - Case 31

---

### Q4. CloudFront에서 503 에러가 발생해요
**A**: Origin (ALB) Health Check 상태를 확인하세요.

1. **ALB 타겟 그룹 확인**:
   - AWS 콘솔 → EC2 → Target Groups
   - Health status가 "healthy"인지 확인

2. **보안 그룹 확인**:
   - ALB 보안 그룹이 CloudFront IP 대역 허용하는지 확인

**참고**: [troubleshooting.md](troubleshooting.md) - Case 3

---

### Q5. RDS 연결이 안 돼요
**A**: 3가지를 확인하세요.

1. **DB 엔드포인트 확인**:
```bash
terraform output -state=kyeol-infra-terraform/dev/terraform.tfstate rds_endpoint
```

2. **Secret 값 확인**:
```bash
kubectl get secret saleor-db-secret -n saleor -o yaml
```

3. **보안 그룹 확인**:
   - RDS 보안 그룹이 EKS 노드 그룹 CIDR 허용하는지 확인 (Port 5432)

**참고**: [troubleshooting.md](troubleshooting.md) - Case 22

---

### Q6. ArgoCD에서 동기화가 실패해요
**A**: Sync 상태와 에러 메시지를 확인하세요.

```bash
argocd app get <app-name>
```

**일반적인 원인**:
- 매니페스트 YAML 문법 오류
- 이미지 태그 없음 또는 ECR 접근 권한 부족
- ConfigMap/Secret 참조 오류

**참고**: [runbook-platform.md](runbook/runbook-platform.md) - ArgoCD 운영 섹션, [troubleshooting.md](troubleshooting.md) - Case 13

---

### Q7. 제품 카탈로그를 초기화하고 싶어요
**A**: 시딩 스크립트를 사용하세요.

```bash
cd scripts
python seed_kyeol_catalog.py --env dev
```

**환경 변수**:
```bash
export SALEOR_API_URL=https://dev.kyeol.com/graphql/
export SALEOR_API_TOKEN=your-staff-token
```

**참고**: [scripts/README.md](../scripts/README.md), [runbook-ops.md](runbook/runbook-ops.md) - 카탈로그 시딩 섹션

---

### Q8. 프로덕션 배포 전에 꼭 확인해야 할 것은?
**A**: 다음 체크리스트를 확인하세요.

- [ ] STAGE 환경에서 최소 24시간 안정성 확인
- [ ] RDS 백업 스냅샷 최신 상태 확인
- [ ] CloudWatch 알람 설정 확인
- [ ] 롤백 계획 수립
- [ ] 배포 시간대 확인 (야간 권장)
- [ ] 주요 이해관계자 공지

**참고**: [runbook-ops.md](runbook/runbook-ops.md) - 배포 전 체크리스트

---

### Q9. Terraform 상태 파일을 실수로 삭제했어요
**A**: S3 백엔드를 사용하고 있다면 복구 가능합니다.

```bash
cd kyeol-infra-terraform/dev
terraform init  # S3에서 자동으로 다운로드
```

S3 버전 관리가 활성화되어 있으므로, AWS 콘솔에서 이전 버전 복원도 가능합니다.

**참고**: [runbook-infra.md](runbook/runbook-infra.md) - Bootstrap 섹션

---

### Q10. 비용이 너무 많이 나와요
**A**: 주요 비용 요소를 점검하세요.

1. **NAT Gateway**: 환경당 2개 (Multi-AZ) → 고정 비용
2. **EKS 노드 그룹**: 불필요하게 큰 인스턴스 타입 사용 중?
3. **RDS**: Reserved Instance로 전환 고려
4. **CloudFront**: 캐시 히트율 확인
5. **S3**: 오래된 로그/백업 삭제

**참고**: [runbook-infra.md](runbook/runbook-infra.md) - 비용 모니터링 및 최적화

---

## 디렉토리 구조

```
kyeol-project/
├── kyeol-docs/                         # 📚 프로젝트 문서 (이 디렉토리)
│   ├── README.md                       # 이 파일 - 프로젝트 진입점
│   ├── runbook/
│   │   ├── runbook-infra.md            # 인프라 운영 가이드
│   │   ├── runbook-platform.md         # 플랫폼 운영 가이드
│   │   └── runbook-ops.md              # 애플리케이션 운영 가이드
│   └── troubleshooting.md              # 장애 대응 가이드 (36개 실제 사례)
├── scripts/                            # 🛠️ 유틸리티 스크립트
│   ├── README.md                       # 스크립트 사용법
│   ├── check_saleor_permissions.py
│   ├── seed_kyeol_catalog.py
│   └── upload_images_to_s3.py
├── archive/                            # 📦 데이터 아카이브
│   ├── product-images/
│   │   ├── kyeol_product_images.zip
│   │   └── kyeol_product_images/
│   └── seed-data/
│       ├── kyeol_products_100.csv
│       ├── kyeol_variants_100.csv
│       ├── products_original_images.csv
│       ├── products_thumbnail_images.csv
│       └── product_image_urls.json
├── kyeol-infra-terraform/              # 🏗️ Terraform 인프라 코드
│   ├── modules/                        # 재사용 가능한 모듈
│   ├── bootstrap/                      # S3 백엔드, 잠금 설정
│   ├── mgmt/                           # MGMT 환경
│   ├── dev/                            # DEV 환경
│   ├── stage/                          # STAGE 환경
│   └── prod/                           # PROD 환경
├── kyeol-app-gitops/                   # ⚙️ GitOps 매니페스트
│   ├── base/                           # 공통 매니페스트
│   └── overlays/                       # 환경별 오버레이
│       ├── dev/
│       ├── stage/
│       └── prod/
├── saleor/                             # 🛒 Saleor Core (백엔드)
├── kyeol-dashboard/                    # 📊 Saleor Dashboard (관리자 UI)
└── kyeol-storefront/                   # 🛍️ Saleor Storefront (고객 UI)
```

---

## 시작하기

1. **인프라 구축**: [runbook-infra.md](runbook/runbook-infra.md)를 따라 VPC, EKS, RDS 등 인프라 구축
2. **플랫폼 구성**: [runbook-platform.md](runbook/runbook-platform.md)를 따라 EKS에 AWS Load Balancer Controller, ArgoCD 설치
3. **애플리케이션 배포**: [runbook-ops.md](runbook/runbook-ops.md)를 따라 Saleor 애플리케이션 배포
4. **문제 발생 시**: [troubleshooting.md](troubleshooting.md)에서 증상 검색

---

## 문의 및 지원

- **Terraform 관련**: [runbook-infra.md](runbook/runbook-infra.md) 참조
- **Kubernetes/GitOps**: [runbook-platform.md](runbook/runbook-platform.md) 참조
- **Saleor 애플리케이션**: [runbook-ops.md](runbook/runbook-ops.md) 참조
- **장애 대응**: [troubleshooting.md](troubleshooting.md) 참조

---

**마지막 업데이트**: 2026-01-21
**문서 버전**: v3.0
**프로젝트 상태**: 운영 중
