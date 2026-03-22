# Chapter 20. VPN과 Private Link

## 20.1 프라이빗 네트워킹의 필요성

### 문제 상황

```
Public 인터넷을 통한 통신의 위험:

사내망 (온프레미스) ──→ Public Internet ──→ AWS VPC
                      ↑
                   보안 위험:
                   - 도청 가능
                   - 변조 가능
                   - DDoS 공격

해결책:
- VPN: 암호화된 터널
- Direct Connect: 전용선
- PrivateLink: AWS 내부 통신
```

---

## 20.2 VPN (Virtual Private Network)

### 정의

```
VPN: 가상 사설 네트워크
공용 인터넷을 통해 암호화된 프라이빗 연결 생성

비유: 해저 터널
┌──────────────────────────────────────┐
│  육지 A ═══════ 터널 ═══════ 육지 B  │
│  (온프레미스)  (암호화)  (AWS VPC)   │
└──────────────────────────────────────┘

외부에서 터널 내부를 볼 수 없음
```

### VPN 종류

```
1. Site-to-Site VPN
   네트워크 ↔ 네트워크
   온프레미스 ↔ AWS VPC

2. Client VPN
   개인 PC ↔ 네트워크
   재택근무자 ↔ 회사망
```

---

## 20.3 AWS Site-to-Site VPN

### 구성요소

```
┌────────────────────────────────────────┐
│  온프레미스                             │
│  ┌──────────────────┐                  │
│  │ Customer Gateway │ (물리 장비)      │
│  │ (CGW)            │                  │
│  └────────┬─────────┘                  │
└───────────┼────────────────────────────┘
            │ IPsec 터널
            │ (Public Internet)
            ↓
┌───────────┼────────────────────────────┐
│  AWS VPC  │                            │
│  ┌────────▼─────────┐                  │
│  │ Virtual Private  │                  │
│  │ Gateway (VGW)    │                  │
│  └──────────────────┘                  │
│           │                            │
│  ┌────────▼─────────┐                  │
│  │ Private Subnet   │                  │
│  │ EC2 Instances    │                  │
│  └──────────────────┘                  │
└────────────────────────────────────────┘
```

### IPsec 터널

```
2개의 터널 (고가용성):
Tunnel 1: Active
Tunnel 2: Standby (장애 시 자동 전환)

암호화:
- IKE (Internet Key Exchange)
- AES-256 암호화
- SHA-2 해싱
```

---

## 20.4 Site-to-Site VPN 설정

### Terraform 예시

```hcl
# Virtual Private Gateway
resource "aws_vpn_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-vgw"
  }
}

# Customer Gateway (온프레미스 라우터)
resource "aws_customer_gateway" "main" {
  bgp_asn    = 65000
  ip_address = "203.0.113.100"  # 온프레미스 공인 IP
  type       = "ipsec.1"

  tags = {
    Name = "office-cgw"
  }
}

# VPN Connection
resource "aws_vpn_connection" "main" {
  vpn_gateway_id      = aws_vpn_gateway.main.id
  customer_gateway_id = aws_customer_gateway.main.id
  type                = "ipsec.1"
  static_routes_only  = true  # 또는 BGP 사용

  tags = {
    Name = "office-vpn"
  }
}

# Static Route (온프레미스 네트워크)
resource "aws_vpn_connection_route" "office" {
  destination_cidr_block = "192.168.0.0/16"  # 온프레미스 CIDR
  vpn_connection_id      = aws_vpn_connection.main.id
}

# VPC Route Table (VPN으로 라우팅)
resource "aws_route" "vpn" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "192.168.0.0/16"
  gateway_id             = aws_vpn_gateway.main.id
}

# VPN Gateway를 라우팅 테이블에 전파
resource "aws_vpn_gateway_route_propagation" "private" {
  vpn_gateway_id = aws_vpn_gateway.main.id
  route_table_id = aws_route_table.private.id
}
```

### 온프레미스 라우터 설정 (예시)

```bash
# 온프레미스 라우터에서 설정
# (AWS Console에서 설정 파일 다운로드 가능)

# 1. IPsec Tunnel 생성
crypto isakmp policy 200
  encryption aes 256
  hash sha256
  authentication pre-share
  group 2

# 2. Pre-Shared Key 설정
crypto isakmp key [AWS에서 제공한 키] address [AWS VPN IP]

# 3. 라우팅 설정
ip route 10.0.0.0 255.255.0.0 Tunnel1
```

---

## 20.5 VPN 통신 흐름

### 예시

```
온프레미스 서버 (192.168.1.10) → AWS EC2 (10.0.2.20)

┌────────────────────────────────────┐
│  온프레미스 서버                    │
│  IP: 192.168.1.10                  │
└───────┬────────────────────────────┘
        │ 목적지: 10.0.2.20
        ↓
┌────────────────────────────────────┐
│  온프레미스 라우터                  │
│  "10.0.x.x → VPN Tunnel"          │
└───────┬────────────────────────────┘
        │ IPsec 암호화
        ↓
    Public Internet
    (암호화된 패킷)
        ↓
┌────────────────────────────────────┐
│  AWS Virtual Private Gateway       │
│  IPsec 복호화                      │
└───────┬────────────────────────────┘
        │ 목적지: 10.0.2.20
        ↓
┌────────────────────────────────────┐
│  AWS EC2                           │
│  IP: 10.0.2.20                     │
└────────────────────────────────────┘

→ 중간에 암호화되어 있어 안전
```

---

## 20.6 AWS Client VPN

### 개념

```
개인 PC/노트북 → AWS VPC

용도:
- 재택근무
- 외부에서 AWS 리소스 접근
- 보안 강화

┌──────────────┐
│ 재택근무자 PC │
│ (VPN Client) │
└──────┬───────┘
       │ OpenVPN
       │ (암호화)
       ↓
┌──────────────────────────────────┐
│  AWS Client VPN Endpoint         │
└──────┬───────────────────────────┘
       │
       ↓
┌──────────────────────────────────┐
│  Private Subnet (10.0.2.0/24)    │
│  EC2, RDS 등                     │
└──────────────────────────────────┘
```

### 설정 예시

```hcl
# Client VPN Endpoint
resource "aws_ec2_client_vpn_endpoint" "main" {
  description            = "Client VPN"
  server_certificate_arn = aws_acm_certificate.server.arn
  client_cidr_block      = "172.16.0.0/16"  # VPN 클라이언트 IP 대역

  authentication_options {
    type                       = "certificate-authentication"
    root_certificate_chain_arn = aws_acm_certificate.client.arn
  }

  connection_log_options {
    enabled               = true
    cloudwatch_log_group  = aws_cloudwatch_log_group.vpn.name
    cloudwatch_log_stream = aws_cloudwatch_log_stream.vpn.name
  }

  tags = {
    Name = "client-vpn"
  }
}

# Client VPN Network Association
resource "aws_ec2_client_vpn_network_association" "main" {
  client_vpn_endpoint_id = aws_ec2_client_vpn_endpoint.main.id
  subnet_id              = aws_subnet.private.id
}

# Authorization Rule (접근 허용)
resource "aws_ec2_client_vpn_authorization_rule" "main" {
  client_vpn_endpoint_id = aws_ec2_client_vpn_endpoint.main.id
  target_network_cidr    = "10.0.0.0/16"  # VPC CIDR
  authorize_all_groups   = true
}
```

---

## 20.7 AWS PrivateLink

### 정의

```
PrivateLink:
AWS 서비스에 프라이빗하게 접근
Public 인터넷 거치지 않음

비유: 건물 내부 전용 통로
┌────────────────────────────────────┐
│  VPC A          VPC B               │
│  ┌─────┐       ┌─────┐             │
│  │ EC2 │ ───→ │ ALB │             │
│  └─────┘       └─────┘             │
│    │             ↑                  │
│    └─── PrivateLink (내부 통로)    │
└────────────────────────────────────┘

인터넷 노출 없이 안전
```

### VPC Endpoint

```
PrivateLink를 사용하는 방법

2가지 타입:
1. Interface Endpoint (ENI)
   - 대부분의 AWS 서비스
   - EC2, RDS, S3 등

2. Gateway Endpoint
   - S3, DynamoDB만
   - 무료!
```

---

## 20.8 VPC Endpoint (Interface)

### 개념

```
AWS 서비스를 프라이빗하게 접근

Without VPC Endpoint:
EC2 (Private) → NAT → IGW → S3 (Public)
비용 발생, 인터넷 노출

With VPC Endpoint:
EC2 (Private) → VPC Endpoint → S3 (Private)
비용 절감, 보안 강화
```

### Terraform 예시

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

# DynamoDB VPC Endpoint (Gateway 타입, 무료!)
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-northeast-2.dynamodb"

  route_table_ids = [
    aws_route_table.private.id
  ]

  tags = {
    Name = "dynamodb-endpoint"
  }
}

# EC2 VPC Endpoint (Interface 타입, 유료)
resource "aws_vpc_endpoint" "ec2" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-northeast-2.ec2"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private.id]
  security_group_ids  = [aws_security_group.vpc_endpoint.id]

  private_dns_enabled = true  # ec2.amazonaws.com으로 접근 가능

  tags = {
    Name = "ec2-endpoint"
  }
}

# VPC Endpoint 보안 그룹
resource "aws_security_group" "vpc_endpoint" {
  name   = "vpc-endpoint-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # VPC CIDR
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 20.9 PrivateLink 서비스 공유

### 자체 서비스를 PrivateLink로 공유

```
시나리오:
VPC-A의 서비스를 VPC-B에서 프라이빗하게 사용

┌────────────────┐         ┌────────────────┐
│   VPC-A        │         │   VPC-B        │
│  ┌─────┐       │         │  ┌─────┐       │
│  │ NLB │←──────┼─────────┼→│ ENI │       │
│  └──┬──┘       │         │  └──┬──┘       │
│     │          │PrivateLink  │            │
│  ┌──▼──┐       │         │  ┌──▼──┐       │
│  │ EC2 │       │         │  │ EC2 │       │
│  │(API)│       │         │  │     │       │
│  └─────┘       │         │  └─────┘       │
└────────────────┘         └────────────────┘
```

### Terraform 설정

```hcl
# VPC-A: Service Provider

# Network Load Balancer
resource "aws_lb" "api" {
  name               = "api-nlb"
  internal           = true
  load_balancer_type = "network"
  subnets            = [aws_subnet.private_a.id]

  tags = {
    Name = "api-nlb"
  }
}

# VPC Endpoint Service
resource "aws_vpc_endpoint_service" "api" {
  acceptance_required        = false
  network_load_balancer_arns = [aws_lb.api.arn]

  tags = {
    Name = "api-service"
  }
}

# VPC-B: Service Consumer

# VPC Endpoint (Interface)
resource "aws_vpc_endpoint" "api" {
  vpc_id              = aws_vpc.vpc_b.id
  service_name        = aws_vpc_endpoint_service.api.service_name
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private_b.id]
  security_group_ids  = [aws_security_group.endpoint_b.id]

  tags = {
    Name = "api-endpoint"
  }
}
```

---

## 20.10 Direct Connect

### 개념

```
Direct Connect (DX):
AWS와 온프레미스 간 전용 네트워크 연결

VPN vs Direct Connect:
┌──────────────┬─────────────┬─────────────┐
│   항목        │ VPN         │ DX          │
├──────────────┼─────────────┼─────────────┤
│ 연결 방식     │ Public      │ Private     │
│ 대역폭        │ ~1.25Gbps   │ 1-100Gbps   │
│ 지연시간      │ 가변        │ 일정        │
│ 비용          │ 저렴        │ 비쌈        │
│ 설정 시간     │ 즉시        │ 수주-수개월 │
│ 용도          │ 소규모      │ 대규모      │
└──────────────┴─────────────┴─────────────┘
```

### 구조

```
┌─────────────────────────────────────┐
│  온프레미스 데이터센터               │
│  ┌──────────────┐                   │
│  │ 온프레미스    │                   │
│  │ 라우터       │                   │
│  └───────┬──────┘                   │
└──────────┼──────────────────────────┘
           │ 전용선 (1Gbps-100Gbps)
           ↓
┌──────────┼──────────────────────────┐
│  AWS Direct Connect Location        │
│  ┌───────▼──────┐                   │
│  │ DX Router    │                   │
│  └───────┬──────┘                   │
└──────────┼──────────────────────────┘
           │ AWS Backbone
           ↓
┌──────────┼──────────────────────────┐
│  AWS VPC │                          │
│  ┌───────▼──────┐                   │
│  │ Virtual      │                   │
│  │ Private      │                   │
│  │ Gateway      │                   │
│  └──────────────┘                   │
└─────────────────────────────────────┘
```

---

## 20.11 비용 비교

### VPN 비용

```
Site-to-Site VPN:
- VPN 연결: $0.05/시간 × 730 = $36.50/월
- 데이터 전송: $0.09/GB

Client VPN:
- Endpoint: $0.10/시간 × 730 = $73/월
- 연결: $0.05/시간/사용자
- 데이터 전송: $0.09/GB

→ 소규모/임시 연결에 적합
```

### Direct Connect 비용

```
포트 비용:
- 1Gbps: ~$216/월
- 10Gbps: ~$1,620/월

데이터 전송:
- Out: $0.02-0.09/GB (리전별)
- In: 무료

초기 설정: $수천-수만
→ 대규모/장기 사용에 적합
```

### VPC Endpoint 비용

```
Gateway Endpoint (S3, DynamoDB):
- 무료! ✅

Interface Endpoint:
- $0.01/시간 × 730 = $7.30/월
- 데이터 처리: $0.01/GB

→ NAT Gateway보다 저렴
```

---

## 20.12 실전 시나리오

### 시나리오 1: 하이브리드 클라우드

```
온프레미스 + AWS 연동

구조:
온프레미스: 192.168.0.0/16
AWS VPC: 10.0.0.0/16

연결:
1. 초기: Site-to-Site VPN
   빠른 구축, 테스트

2. 운영: Direct Connect
   안정적, 대용량

3. 백업: VPN (DX 장애 시)
   HA 구성
```

### 시나리오 2: 멀티 VPC 아키텍처

```
여러 VPC 간 통신

Before (VPC Peering):
VPC-A ↔ VPC-B
VPC-A ↔ VPC-C
VPC-B ↔ VPC-C
→ N×(N-1)/2 = 3개 연결

After (Transit Gateway):
VPC-A ──┐
VPC-B ──┼── Transit Gateway
VPC-C ──┘
→ N = 3개 연결

→ 확장성 증가
```

### 시나리오 3: Lambda VPC 접근

```
Lambda → RDS 접근

문제:
Lambda VPC 내 → 인터넷 접근 불가

해결 1: NAT Gateway
비용: ~$40/월

해결 2: VPC Endpoint
S3, DynamoDB, SQS 등
비용: ~$7/월

→ VPC Endpoint 활용으로 비용 절감
```

---

## 핵심 요약

### VPN
```
Site-to-Site: 온프레미스 ↔ AWS
Client VPN: 개인 PC ↔ AWS
암호화: IPsec
비용: ~$36/월
용도: 소규모, 임시
```

### Direct Connect
```
전용선 연결
대역폭: 1-100Gbps
지연시간: 일정
비용: ~$216/월~
용도: 대규모, 장기
```

### PrivateLink (VPC Endpoint)
```
AWS 서비스 프라이빗 접근
Gateway: S3, DynamoDB (무료)
Interface: 기타 서비스 ($7/월)
NAT 대체 가능
```

---

## 다음 단계

Part 4 (클라우드 네트워킹) 완료! 이제 **보안** 파트로 이동합니다!

👉 [Chapter 21. 방화벽 기초](../part05-보안/21-방화벽.md)

---

## 실습 과제

### 과제 1: Site-to-Site VPN 시뮬레이션

```
1. 2개 VPC 생성 (온프레미스 시뮬레이션)
2. VPN Gateway + Customer Gateway
3. VPN Connection 생성
4. 라우팅 설정
5. EC2 간 통신 확인
```

### 과제 2: VPC Endpoint 활용

```
1. Private Subnet EC2 생성
2. S3 Gateway Endpoint 생성
3. NAT Gateway 없이 S3 접근 확인
4. 비용 절감 계산
```

### 과제 3: PrivateLink 서비스 공유

```
1. VPC-A: NLB + 웹서버
2. VPC Endpoint Service 생성
3. VPC-B: VPC Endpoint 생성
4. 프라이빗 접근 확인
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: DevOps, 네트워크 엔지니어
**중요도**: ⭐⭐⭐ 매우 중요!