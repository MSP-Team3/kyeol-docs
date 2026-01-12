# Regional NAT Gateway 마이그레이션 변경 내역 상세

이 문서는 Regional NAT Gateway 도입을 위해 실제로 수정된 파일과 실행된 명령어의 상세 내역을 기록합니다.

## 1. 파일 변경 내역 (Code Changes)

### 1-1. 모듈 수정 (`modules/vpc`)

#### 📄 `modules/vpc/variables.tf`
새로운 기능을 제어하기 위한 변수 `nat_gateway_mode`를 추가했습니다.
```hcl
variable "nat_gateway_mode" {
  description = "NAT Gateway 모드 (zonal: AZ별 생성, regional: 리전 통합)"
  type        = string
  default     = "zonal"
  validation {
    condition     = contains(["zonal", "regional"], var.nat_gateway_mode)
    error_message = "Valid values for nat_gateway_mode are (zonal, regional)."
  }
}
```

#### 📄 `modules/vpc/nat.tf`
`availability_mode = "regional"`을 사용하는 새로운 NAT 리소스를 정의하고, 기존 리소스와 조건부로 분기했습니다.
```hcl
# 기존 Zonal NAT (mode == "zonal" 일 때만 생성)
resource "aws_nat_gateway" "internet" {
  count = var.enable_nat_gateway && var.nat_gateway_mode == "zonal" ? ... 
  # ... (기존 설정 유지)
}

# 신규 Regional NAT (mode == "regional" 일 때만 생성)
resource "aws_nat_gateway" "regional" {
  count = var.enable_nat_gateway && var.nat_gateway_mode == "regional" ? 1 : 0

  availability_mode = "regional"  # 핵심 설정
  vpc_id            = aws_vpc.main.id
  
  # 고정 IP(EIP) 할당 (Manual Mode)
  dynamic "availability_zone_address" {
    for_each = var.single_nat_gateway ? [var.azs[0]] : var.azs
    content {
      availability_zone = availability_zone_address.value
      # HA 구성을 위해 각 AZ별로 다른 EIP 인덱스를 매핑합니다.
      allocation_ids    = [aws_eip.internet[var.single_nat_gateway ? 0 : index(var.azs, availability_zone_address.value)].id]
    }
  }

  tags = merge(var.tags, {
    Name    = "${var.name_prefix}-nat-regional"
    Purpose = "general-internet-traffic-regional"
  })
}
```
*   **주요 설정**: `availability_mode = "regional"`, `vpc_id` 사용 (subnet_id 미사용).
*   **EIP 설정 (HA)**: `single_nat_gateway = false` 설정 시 **각 AZ별로 고정 IP**를 할당하고, EIP 이름에도 AZ Suffix를 붙여(`-a`, `-c`) 식별을 용이하게 변경했습니다.
*   **Naming 규칙 적용**: `tags` 블록에 명시적으로 `Name = ...-nat-regional` 태그를 추가하여 리소스 식별성을 강화했습니다.
*   **주요 설정**: `availability_mode = "regional"`, `vpc_id` 사용 (subnet_id 미사용).
*   **EIP 설정**: `single_nat_gateway = false` (HA 구성) 설정 시 **각 AZ별로 고정 IP**를 할당하여(`availability_zone_address` 블록), AZ 장애 시에도 다른 AZ의 통신을 보장합니다.

#### 📄 `modules/vpc/route_tables.tf`
라우팅 테이블이 현재 모드에 맞는 NAT Gateway ID를 바라보도록 수정했습니다.
```hcl
nat_gateway_id = var.nat_gateway_mode == "regional" ? aws_nat_gateway.regional[0].id : aws_nat_gateway.internet[0].id
```

#### 📄 `modules/vpc/outputs.tf`
출력값(Output)도 모드에 따라 올바른 ID를 반환하도록 삼항 연산자를 적용했습니다.

### 6. 다른 환경(Dev/Prod) 적용 가이드 (For Team)

현재 Stage 환경에만 Regional NAT가 적용되어 있습니다. 팀원들이 Dev나 Prod 환경에도 이를 적용하려면 다음 변경이 필요합니다.

### 6-1. Provider 버전 업데이트 (필수)
모든 환경의 `versions.tf` 파일에서 `hashicorp/aws` 공급자 버전을 **6.28.0 이상**으로 올려야 합니다.
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.28.0" # 필수 업데이트
    }
  }
}
```

### 6-2. 환경별 `main.tf` 설정 변경
각 환경의 `main.tf` 내 `module "vpc"` 블록에 다음 설정을 추가합니다.

```hcl
module "vpc" {
  # ... 기존 설정 ...

  # Regional NAT 모드 활성화
  nat_gateway_mode   = "regional" 
  
  # HA 구성 (Multi-AZ EIP 사용 시 false / 단일 IP 사용 시 true)
  # Prod 환경은 반드시 false (HA) 권장
  single_nat_gateway = false 
}
```

### 6-3. 적용 절차
1.  `rm -rf .terraform .terraform.lock.hcl` (버전 캐시 초기화)
2.  `terraform init`
3.  `terraform apply` (기존 Zonal NAT 삭제 및 Regional NAT 생성됨)

### 1-2. 환경 설정 수정 (`envs/stage`)

#### 📄 `envs/stage/main.tf`
새로운 기능을 Stage 환경에 적용했습니다.
```hcl
module "vpc" {
  # ...
  enable_nat_gateway = true
  
  # Regional NAT 설정 활성화
  nat_gateway_mode   = "regional" 
  single_nat_gateway = false     # HA 구성 (EIP 2개 사용)
}
```

#### 📄 `envs/stage/versions.tf` (및 전체 모듈)
Regional NAT 기능을 지원하는 AWS Provider 버전을 사용하기 위해 `~> 5.0`을 `6.28.0`으로 고정했습니다.
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.28.0"  # <-- 업데이트됨
    }
  }
}
```

## 2. 실행 명령어 내역 (Executed Commands)

파일 수정 외에 시스템 상에서 실행된 주요 명령어들입니다.

### 2-1. Provider 버전 일괄 업데이트
모든 모듈의 `versions.tf`가 구버전(`~> 5.0`)을 가리키고 있어 충돌이 발생했기에, 이를 일괄 수정했습니다.
```bash
# 실제로 수행된 작업 (AI Agent 내부 로직)
find ../../modules -name "versions.tf" | xargs sed -i 's/~> 5.0/6.28.0/g'
# 추가로 envs/stage/versions.tf 파일도 직접 6.28.0으로 수정함
```

### 2-2. Terraform 초기화 및 적용
공급자(Provider) 버전이 변경되었으므로 재초기화 후 적용했습니다.
```bash
# 1. 기존 .terraform 캐시 삭제 (버전 꼬임 방지)
rm -rf .terraform .terraform.lock.hcl

# 2. 초기화 (v6.28.0 다운로드)
terraform init

# 3. 적용 (Regional NAT 생성)
terraform apply -auto-approve
```