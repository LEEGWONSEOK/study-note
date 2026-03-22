# Chapter 19. NAT와 인터넷 게이트웨이

## 19.1 인터넷 게이트웨이 (IGW)

### 정의

```
Internet Gateway (IGW):
VPC와 인터넷 사이의 관문

비유: 아파트 정문
┌────────────────────────────────┐
│  아파트 (VPC)                   │
│                                │
│  ┌──────────┐                  │
│  │ 각 집     │                  │
│  │ (EC2)    │                  │
│  └────┬─────┘                  │
│       │                        │
│  ┌────▼──────┐                 │
│  │ 정문 (IGW)│ ←──→ 외부 세계  │
│  └───────────┘                 │
└────────────────────────────────┘
```

### 특징

```
1. VPC당 1개 IGW
   - 1:1 연결
   - Highly Available (HA)
   - 자동 확장

2. 무료
   - IGW 자체 비용 없음
   - 데이터 전송 비용만 발생

3. Stateful
   - 요청 → 응답 자동 허용
```

---

## 19.2 IGW 동작 원리

### NAT (Network Address Translation)

```
Private IP ↔ Public IP 변환

┌─────────────────────────────────┐
│  EC2                             │
│  Private IP: 10.0.1.10           │
│  Public IP: 13.125.1.100         │
└───────┬─────────────────────────┘
        │ 출발지: 10.0.1.10
        ↓
┌─────────────────────────────────┐
│  Internet Gateway                │
│  NAT: 10.0.1.10 → 13.125.1.100  │
└───────┬─────────────────────────┘
        │ 출발지: 13.125.1.100
        ↓
    Internet

응답:
Internet
    │ 목적지: 13.125.1.100
    ↓
IGW (NAT: 13.125.1.100 → 10.0.1.10)
    │ 목적지: 10.0.1.10
    ↓
EC2
```

### 인터넷 통신 요구사항

```
EC2가 인터넷과 통신하려면:

✅ 1. Public IP 또는 EIP 할당
✅ 2. IGW가 VPC에 연결
✅ 3. 라우팅 테이블: 0.0.0.0/0 → IGW
✅ 4. 보안 그룹/NACL 허용

하나라도 빠지면 통신 불가!
```

---

## 19.3 IGW 설정

### Terraform 예시

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "main-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true  # 자동 Public IP 할당

  tags = {
    Name = "public-subnet"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-route-table"
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# EC2 (Public)
resource "aws_instance" "web" {
  ami           = "ami-0c9c942bd7bf113a2"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id

  # Public IP 자동 할당 (서브넷 설정 덮어쓰기 가능)
  associate_public_ip_address = true

  tags = {
    Name = "web-server"
  }
}
```

---

## 19.4 NAT Gateway

### 정의

```
NAT Gateway:
Private Subnet의 EC2가 인터넷에 접근하도록 허용
하지만 인터넷에서 Private EC2로는 접근 불가

비유: 회사 프록시 서버
사원 (Private EC2) → 프록시 → 인터넷 ✅
인터넷 → 프록시 → 사원 ❌
```

### NAT Gateway vs IGW

```
┌──────────────┬────────────────┬─────────────┐
│   항목        │  IGW           │ NAT Gateway │
├──────────────┼────────────────┼─────────────┤
│ 위치          │ VPC 레벨       │ Subnet 레벨 │
│ 목적          │ 양방향 통신    │ 단방향 통신 │
│ Public IP 필요│ ✅            │ ❌          │
│ 인바운드      │ 허용           │ 차단        │
│ 아웃바운드    │ 허용           │ 허용        │
│ 비용          │ 무료           │ 유료        │
└──────────────┴────────────────┴─────────────┘
```

---

## 19.5 NAT Gateway 동작 원리

### 통신 흐름

```
Private EC2 → 인터넷

┌─────────────────────────────────┐
│  Private EC2: 10.0.2.10          │
│  (Public IP 없음)                │
└───────┬─────────────────────────┘
        │ 목적지: google.com
        │ 출발지: 10.0.2.10
        ↓
┌─────────────────────────────────┐
│  Route Table (Private)           │
│  0.0.0.0/0 → NAT Gateway        │
└───────┬─────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  NAT Gateway                     │
│  위치: Public Subnet             │
│  Private IP: 10.0.1.50           │
│  EIP: 13.125.1.200               │
│                                  │
│  NAT: 10.0.2.10 → 13.125.1.200  │
└───────┬─────────────────────────┘
        │ 출발지: 13.125.1.200
        ↓
┌─────────────────────────────────┐
│  Internet Gateway                │
└───────┬─────────────────────────┘
        ↓
    Internet

→ 인터넷에서는 NAT Gateway의 EIP만 보임
→ Private EC2의 IP는 노출 안 됨
```

### 인바운드 차단

```
인터넷 → Private EC2 시도

Internet
    │ 목적지: 13.125.1.200 (NAT EIP)
    ↓
NAT Gateway
    │ 이 연결을 시작한 적 없음
    │ → 차단! ❌

→ NAT Gateway는 내부에서 시작한 연결만 허용
→ 외부에서 시작한 연결은 차단
```

---

## 19.6 NAT Gateway 설정

### Terraform 전체 예시

```hcl
# Public Subnet (NAT Gateway 배치)
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

# Private Subnet
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "ap-northeast-2a"

  tags = {
    Name = "private-subnet"
  }
}

# Elastic IP (NAT Gateway용)
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = {
    Name = "nat-eip"
  }

  # IGW가 먼저 생성되어야 함
  depends_on = [aws_internet_gateway.main]
}

# NAT Gateway
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public.id  # Public Subnet!

  tags = {
    Name = "main-nat"
  }

  depends_on = [aws_internet_gateway.main]
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-rt"
  }
}

# Private Route Table
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id  # NAT Gateway!
  }

  tags = {
    Name = "private-rt"
  }
}

# Associations
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}
```

---

## 19.7 NAT Gateway vs NAT Instance

### 비교

```
┌──────────────┬────────────────┬─────────────────┐
│   항목        │ NAT Gateway    │ NAT Instance    │
├──────────────┼────────────────┼─────────────────┤
│ 가용성        │ HA (자동)      │ 직접 구성       │
│ 대역폭        │ 45Gbps         │ 인스턴스 타입   │
│ 유지보수      │ AWS 관리       │ 직접 관리       │
│ 비용          │ $0.045/시간    │ EC2 비용        │
│              │ + 데이터 전송  │                 │
│ 보안 그룹     │ 불필요         │ 필요            │
│ Bastion 역할  │ 불가           │ 가능            │
└──────────────┴────────────────┴─────────────────┘

권장: NAT Gateway ✅
(관리 부담 적고, 고가용성)
```

### NAT Instance 설정 (참고)

```hcl
# NAT Instance (저비용 필요 시)
resource "aws_instance" "nat" {
  ami           = "ami-nat-xxx"  # NAT AMI
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id

  # Source/Destination Check 비활성화 (필수!)
  source_dest_check = false

  tags = {
    Name = "nat-instance"
  }
}

# Private Route Table
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block  = "0.0.0.0/0"
    instance_id = aws_instance.nat.id  # NAT Instance
  }
}
```

---

## 19.8 고가용성 NAT 구성

### 멀티 AZ NAT Gateway

```
각 AZ마다 NAT Gateway 배치

┌────────────────────────────────────┐
│  VPC: 10.0.0.0/16                  │
│                                    │
│  ┌──────────────┐ ┌─────────────┐ │
│  │   AZ-A       │ │   AZ-C      │ │
│  ├──────────────┤ ├─────────────┤ │
│  │Public 10.0.1 │ │Public 10.0.3│ │
│  │ NAT-A (EIP1) │ │NAT-C (EIP2) │ │
│  ├──────────────┤ ├─────────────┤ │
│  │Private 10.0.2│ │Private 10.0.4│ │
│  │ → NAT-A      │ │ → NAT-C     │ │
│  └──────────────┘ └─────────────┘ │
│                                    │
│  AZ-A 장애 시:                     │
│  → AZ-C는 영향 없음 ✅             │
└────────────────────────────────────┘
```

```hcl
# AZ-A
resource "aws_subnet" "public_a" {
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-northeast-2a"
}

resource "aws_subnet" "private_a" {
  cidr_block        = "10.0.2.0/24"
  availability_zone = "ap-northeast-2a"
}

resource "aws_eip" "nat_a" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat_a" {
  allocation_id = aws_eip.nat_a.id
  subnet_id     = aws_subnet.public_a.id
}

resource "aws_route_table" "private_a" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_a.id
  }
}

# AZ-C (동일한 구조)
resource "aws_subnet" "public_c" {
  cidr_block        = "10.0.3.0/24"
  availability_zone = "ap-northeast-2c"
}

resource "aws_subnet" "private_c" {
  cidr_block        = "10.0.4.0/24"
  availability_zone = "ap-northeast-2c"
}

resource "aws_eip" "nat_c" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat_c" {
  allocation_id = aws_eip.nat_c.id
  subnet_id     = aws_subnet.public_c.id
}

resource "aws_route_table" "private_c" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_c.id
  }
}
```

---

## 19.9 비용 최적화

### NAT Gateway 비용

```
비용 구조:
1. 시간당 요금: $0.045/시간
2. 데이터 처리: $0.045/GB

월 비용 계산:
시간: $0.045 × 24 × 30 = $32.40
데이터: 100GB × $0.045 = $4.50
──────────────────────────────
합계: ~$37/월 (1개 NAT Gateway)

멀티 AZ (2개):
~$74/월
```

### 비용 절감 방법

```
1. 단일 NAT Gateway (개발 환경)
   모든 Private 서브넷이 1개 NAT 공유
   ⚠️ 고가용성 포기

2. VPC Endpoint 사용
   S3, DynamoDB 등 AWS 서비스는
   VPC Endpoint로 접근 → NAT 불필요

3. NAT Instance (초저비용)
   t3.nano: ~$3.5/월
   ⚠️ 관리 부담, 낮은 성능

4. 불필요한 트래픽 차단
   - CloudWatch Logs → 직접 전송 대신 VPC Endpoint
   - yum update → 스케줄링 (필요 시에만)
```

### VPC Endpoint 활용

```hcl
# S3 VPC Endpoint (Gateway 타입, 무료!)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-northeast-2.s3"

  route_table_ids = [
    aws_route_table.private.id
  ]

  tags = {
    Name = "s3-endpoint"
  }
}

# Private EC2에서 S3 접근
# → NAT Gateway 거치지 않음
# → 데이터 전송 비용 절감
```

---

## 19.10 실전 시나리오

### 시나리오 1: 웹 애플리케이션

```
구조:
- ALB (Public)
- Web/App 서버 (Private)
- RDS (Private)

┌──────────────────────────────────┐
│  Internet                        │
└────────┬─────────────────────────┘
         ↓
┌──────────────────────────────────┐
│  ALB (Public Subnet)             │
│  EIP: 13.125.1.100               │
└────────┬─────────────────────────┘
         ↓
┌──────────────────────────────────┐
│  EC2 (Private Subnet)            │
│  Private IP: 10.0.2.10           │
│  외부 접근: NAT Gateway          │
└────────┬─────────────────────────┘
         ↓
┌──────────────────────────────────┐
│  RDS (Private Subnet)            │
│  Private IP: 10.0.3.10           │
│  외부 접근: ❌ 불필요             │
└──────────────────────────────────┘

NAT Gateway:
- EC2가 패키지 업데이트용
- 외부 API 호출용
```

### 시나리오 2: Lambda + RDS

```
Lambda (VPC 내) → RDS

문제:
Lambda가 VPC에 있으면 인터넷 접근 불가

해결 1: NAT Gateway
┌───────────┐
│  Lambda   │ → NAT → Internet
│ (Private) │ → RDS
└───────────┘

해결 2: VPC Endpoint
┌───────────┐
│  Lambda   │ → S3 Endpoint → S3
│ (Private) │ → RDS
└───────────┘
```

---

## 19.11 문제 해결

### 문제 1: Private EC2 인터넷 안 됨

```
체크리스트:
✅ NAT Gateway가 Public Subnet에 있나?
✅ NAT Gateway에 EIP 할당?
✅ Private RT: 0.0.0.0/0 → NAT?
✅ Public RT: 0.0.0.0/0 → IGW?
✅ NAT Gateway 상태 available?
✅ 보안 그룹 아웃바운드 허용?

확인:
# Private EC2에서 (SSM Session Manager)
curl ifconfig.me  # NAT EIP 나와야 함
ping 8.8.8.8      # 동작해야 함
```

### 문제 2: NAT 비용 급증

```
원인:
- 대용량 데이터 전송
- 불필요한 외부 API 호출
- 로그 전송

해결:
# CloudWatch 확인
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToSource \
  --dimensions Name=NatGatewayId,Value=nat-xxx \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Sum

# VPC Flow Logs로 어떤 트래픽인지 분석
```

---

## 핵심 요약

### Internet Gateway
```
VPC ↔ 인터넷 연결
Public IP 필요
양방향 통신 가능
무료
```

### NAT Gateway
```
Private EC2 → 인터넷 (단방향)
Public Subnet에 배치
EIP 할당
$0.045/시간 + $0.045/GB
```

### 통신 요구사항
```
Public EC2:
- Public IP/EIP
- IGW
- RT: 0.0.0.0/0 → IGW

Private EC2:
- NAT Gateway
- RT: 0.0.0.0/0 → NAT
```

### 고가용성
```
각 AZ마다 NAT Gateway
→ AZ 장애 시에도 다른 AZ 정상 동작
```

---

## 다음 단계

NAT와 IGW를 이해했으니, 이제 **VPN과 Private Link**를 배워봅시다!

👉 [Chapter 20. VPN과 Private Link](20-VPN-PrivateLink.md)

---

## 실습 과제

### 과제 1: IGW 설정

```
1. Public Subnet 생성
2. IGW 연결
3. 라우팅 테이블 설정
4. EC2 배포 (Public IP)
5. 인터넷 접근 확인
```

### 과제 2: NAT Gateway 설정

```
1. Public + Private Subnet
2. NAT Gateway 배포
3. 라우팅 테이블 설정
4. Private EC2 배포
5. 인터넷 접근 확인 (curl ifconfig.me)
```

### 과제 3: 비용 분석

```
1. NAT Gateway CloudWatch 메트릭 확인
2. BytesOut 분석
3. VPC Flow Logs 활성화
4. Top Talkers 찾기
5. VPC Endpoint로 전환 가능한 트래픽 식별
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 2-3시간
**대상**: 백엔드 개발자, DevOps
**중요도**: ⭐⭐⭐ 매우 중요!
