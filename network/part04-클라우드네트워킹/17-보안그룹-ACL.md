# Chapter 17. 보안 그룹과 ACL

## 17.1 네트워크 보안의 두 계층

### 비유: 아파트 보안

```
┌─────────────────────────────────────────┐
│  아파트 단지 (VPC)                       │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  출입구 (Network ACL)              │  │
│  │  "단지 전체 출입 통제"             │  │
│  │  - 차량 번호로 필터링              │  │
│  └────────────┬──────────────────────┘  │
│               ↓                         │
│  ┌───────────────────────────────────┐  │
│  │  101동 (Subnet)                   │  │
│  │                                   │  │
│  │  ┌───────────────────────────┐    │  │
│  │  │ 각 집 현관문 (Security Group)│  │  │
│  │  │ "누가 우리 집에 올 수 있나"  │  │  │
│  │  │ - 초대한 사람만              │  │  │
│  │  └───────────────────────────┘    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

---

## 17.2 보안 그룹 (Security Group) ⭐ 가장 중요!

### 정의

**인스턴스 수준의 가상 방화벽** (EC2, RDS, ALB 등에 적용)

### 핵심 특징

```
1. Stateful (상태 유지)
   → 나가는 요청의 응답은 자동 허용

2. 허용(Allow) 규칙만 존재
   → 거부(Deny) 규칙 없음

3. 인스턴스 레벨
   → 각 EC2 인스턴스마다 적용

4. 여러 보안 그룹 적용 가능
   → 인스턴스에 여러 SG 연결 가능
```

### Stateful 동작 방식

```
┌──────────────────────────────────────┐
│  클라이언트 (1.2.3.4)                 │
└────────┬─────────────────────────────┘
         │
         │ 1. 요청 (Src: 1.2.3.4:54321)
         ↓
┌──────────────────────────────────────┐
│  Security Group                      │
│                                      │
│  인바운드 규칙 확인:                  │
│  - 포트 80 허용? ✅                  │
│  → 통과                              │
└────────┬─────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│  EC2 인스턴스 (10.0.1.10)            │
└────────┬─────────────────────────────┘
         │
         │ 2. 응답 (Dst: 1.2.3.4:54321)
         ↓
┌──────────────────────────────────────┐
│  Security Group                      │
│                                      │
│  아웃바운드 규칙 확인 생략!           │
│  → 요청의 응답이므로 자동 허용 ✅     │
└────────┬─────────────────────────────┘
         ↓
    클라이언트로 응답
```

---

## 17.3 보안 그룹 규칙 구성

### 인바운드 규칙 (Inbound Rules)

```
외부 → 인스턴스로 들어오는 트래픽

┌──────────┬─────────┬──────────┬─────────────────┐
│   타입    │ 프로토콜 │  포트    │   소스           │
├──────────┼─────────┼──────────┼─────────────────┤
│  HTTP    │  TCP    │   80     │  0.0.0.0/0      │
│  HTTPS   │  TCP    │   443    │  0.0.0.0/0      │
│  SSH     │  TCP    │   22     │  1.2.3.4/32     │
│  MySQL   │  TCP    │  3306    │  sg-app123      │
│  Custom  │  TCP    │  3000    │  10.0.0.0/16    │
└──────────┴─────────┴──────────┴─────────────────┘

소스 종류:
- 0.0.0.0/0: 모든 IP (전체 공개)
- 1.2.3.4/32: 특정 IP만
- 10.0.0.0/16: 특정 CIDR 블록
- sg-xxxxxx: 다른 보안 그룹
```

### 아웃바운드 규칙 (Outbound Rules)

```
인스턴스 → 외부로 나가는 트래픽

기본값: 모든 트래픽 허용 (0.0.0.0/0)

보통 변경하지 않음
```

---

## 17.4 실전 보안 그룹 예시

### 예시 1: 웹 서버 보안 그룹

```hcl
resource "aws_security_group" "web" {
  name        = "web-server-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  # 인바운드: HTTP (전체 공개)
  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # 인바운드: HTTPS (전체 공개)
  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # 인바운드: SSH (내 IP만)
  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["1.2.3.4/32"]  # 내 공인 IP
  }

  # 아웃바운드: 모든 트래픽 허용
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-server-sg"
  }
}
```

### 예시 2: 애플리케이션 서버 보안 그룹

```hcl
resource "aws_security_group" "app" {
  name        = "app-server-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  # 인바운드: 웹 서버에서만 접근 허용
  ingress {
    description     = "Node.js from web servers"
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]  # 웹 SG만!
  }

  # 인바운드: SSH (Bastion Host에서만)
  ingress {
    description     = "SSH from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  # 아웃바운드: 모든 트래픽
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "app-server-sg"
  }
}
```

### 예시 3: 데이터베이스 보안 그룹

```hcl
resource "aws_security_group" "db" {
  name        = "database-sg"
  description = "Security group for RDS database"
  vpc_id      = aws_vpc.main.id

  # 인바운드: 앱 서버에서만 MySQL 접근
  ingress {
    description     = "MySQL from app servers only"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # 아웃바운드: 없음 (DB는 외부 통신 불필요)
  # 또는 최소한만 허용

  tags = {
    Name = "database-sg"
  }
}
```

### 계층별 보안 그룹

```
┌─────────────────────────────────────────┐
│  Internet                               │
└────────┬────────────────────────────────┘
         ↓ 80, 443 (모든 IP)
┌─────────────────────────────────────────┐
│  ALB Security Group                     │
│  - 인바운드: 80, 443 (0.0.0.0/0)        │
└────────┬────────────────────────────────┘
         ↓ 3000 (ALB SG에서만)
┌─────────────────────────────────────────┐
│  App Server Security Group              │
│  - 인바운드: 3000 (sg-alb만)            │
└────────┬────────────────────────────────┘
         ↓ 3306 (App SG에서만)
┌─────────────────────────────────────────┐
│  Database Security Group                │
│  - 인바운드: 3306 (sg-app만)            │
│  - 외부 접근 완전 차단!                  │
└─────────────────────────────────────────┘
```

---

## 17.5 Network ACL (NACL) ⭐ 질문하신 ACL!

### 정의

**서브넷 수준의 방화벽** (서브넷 전체에 적용)

### 핵심 특징

```
1. Stateless (무상태)
   → 요청과 응답을 각각 따로 확인

2. 허용(Allow)과 거부(Deny) 규칙 모두 가능
   → 명시적 거부 가능!

3. 서브넷 레벨
   → 서브넷 전체에 적용

4. 규칙 번호 순서대로 평가
   → 낮은 번호가 우선
```

### Stateless 동작 방식

```
┌──────────────────────────────────────┐
│  클라이언트 (1.2.3.4)                 │
└────────┬─────────────────────────────┘
         │
         │ 1. 요청 (Src: 1.2.3.4:54321)
         ↓
┌──────────────────────────────────────┐
│  Network ACL (인바운드)               │
│                                      │
│  규칙 100: 포트 80 허용? ✅          │
│  → 통과                              │
└────────┬─────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│  EC2 인스턴스                        │
└────────┬─────────────────────────────┘
         │
         │ 2. 응답 (Dst: 1.2.3.4:54321)
         ↓
┌──────────────────────────────────────┐
│  Network ACL (아웃바운드)             │
│                                      │
│  규칙 확인 필수!                      │
│  → Ephemeral 포트 허용 필요          │
│  (1024-65535)                        │
└────────┬─────────────────────────────┘
         ↓
    클라이언트로 응답
```

---

## 17.6 Network ACL 규칙 구성

### 규칙 번호와 우선순위

```
규칙은 번호 순서대로 평가
→ 낮은 번호가 먼저 평가
→ 매칭되면 즉시 적용 (이후 규칙 무시)

┌────────┬──────┬──────────┬──────────┬────────┐
│ 규칙 번호│ 타입 │  프로토콜 │  포트    │  CIDR   │
├────────┼──────┼──────────┼──────────┼────────┤
│  100   │ Allow│   TCP    │   80     │ 0.0.0.0/0│
│  110   │ Deny │   TCP    │   22     │ 0.0.0.0/0│
│  120   │ Allow│   TCP    │   22     │ 1.2.3.4/32│ ← 실행 안 됨!
│  *     │ Deny │   All    │   All    │ 0.0.0.0/0│
└────────┴──────┴──────────┴──────────┴────────┘

위 예시 문제:
→ 110번 규칙이 모든 SSH를 차단
→ 120번 규칙은 절대 실행 안 됨!

올바른 순서:
┌────────┬──────┬──────────┬──────────┬────────┐
│ 규칙 번호│ 타입 │  프로토콜 │  포트    │  CIDR   │
├────────┼──────┼──────────┼──────────┼────────┤
│  100   │ Allow│   TCP    │   22     │ 1.2.3.4/32│ ← 먼저!
│  110   │ Deny │   TCP    │   22     │ 0.0.0.0/0│
│  120   │ Allow│   TCP    │   80     │ 0.0.0.0/0│
│  *     │ Deny │   All    │   All    │ 0.0.0.0/0│
└────────┴──────┴──────────┴──────────┴────────┘
```

### Ephemeral Ports (임시 포트)

```
문제:
클라이언트의 응답 포트는 랜덤 (1024-65535)
→ NACL 아웃바운드에서 이 범위를 허용해야 함!

해결:
아웃바운드에 Ephemeral 포트 허용

인바운드 규칙:
100: TCP 80 허용 (0.0.0.0/0)

아웃바운드 규칙:
100: TCP 1024-65535 허용 (0.0.0.0/0)  ← 필수!
```

---

## 17.7 NACL 실전 예시

### Public 서브넷 NACL

```hcl
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = [aws_subnet.public.id]

  # 인바운드 규칙
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  ingress {
    rule_no    = 110
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  ingress {
    rule_no    = 120
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "1.2.3.4/32"  # 내 IP만
    from_port  = 22
    to_port    = 22
  }

  # Ephemeral ports (응답용)
  ingress {
    rule_no    = 140
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  # 아웃바운드 규칙
  egress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  egress {
    rule_no    = 110
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  # Ephemeral ports
  egress {
    rule_no    = 140
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  tags = {
    Name = "public-nacl"
  }
}
```

### Private 서브넷 NACL

```hcl
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = [aws_subnet.private.id]

  # 인바운드: VPC 내부에서만
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "10.0.0.0/16"  # VPC CIDR
    from_port  = 0
    to_port    = 65535
  }

  # 인바운드: Ephemeral ports (외부 응답)
  ingress {
    rule_no    = 110
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  # 아웃바운드: 모든 트래픽
  egress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = "private-nacl"
  }
}
```

---

## 17.8 보안 그룹 vs NACL 비교 ⭐ 필수 이해!

```
┌──────────────┬──────────────────┬──────────────────┐
│    항목       │  Security Group  │  Network ACL     │
├──────────────┼──────────────────┼──────────────────┤
│  적용 레벨    │  인스턴스        │  서브넷          │
│              │  (EC2, RDS 등)   │  (서브넷 전체)   │
├──────────────┼──────────────────┼──────────────────┤
│  상태         │  Stateful        │  Stateless       │
│              │  (응답 자동 허용) │  (양방향 설정)   │
├──────────────┼──────────────────┼──────────────────┤
│  규칙 타입    │  Allow만         │  Allow + Deny    │
├──────────────┼──────────────────┼──────────────────┤
│  규칙 평가    │  모든 규칙 평가   │  순서대로 평가    │
│              │  (OR 조건)       │  (첫 매칭 적용)  │
├──────────────┼──────────────────┼──────────────────┤
│  기본 정책    │  모두 거부       │  모두 허용       │
├──────────────┼──────────────────┼──────────────────┤
│  용도         │  인스턴스별 제어  │  서브넷 방어선   │
│              │  (주 방화벽)     │  (추가 보안 계층)│
├──────────────┼──────────────────┼──────────────────┤
│  변경 적용    │  즉시            │  즉시            │
├──────────────┼──────────────────┼──────────────────┤
│  사용 빈도    │  매우 높음 ⭐⭐⭐ │  낮음            │
└──────────────┴──────────────────┴──────────────────┘
```

### 언제 무엇을 사용할까?

```
보안 그룹 (Security Group):
✅ 대부분의 경우 (90%)
✅ 인스턴스별 세밀한 제어
✅ 다른 보안 그룹 참조 가능
✅ 관리 간편

Network ACL:
✅ 서브넷 전체 차단 필요 시
✅ 특정 IP 명시적 거부
✅ 추가 보안 계층
✅ 규정 준수 (Compliance)

권장:
→ 보안 그룹을 메인으로 사용
→ NACL은 추가 보안 계층으로
```

---

## 17.9 실전 시나리오

### 시나리오 1: 특정 IP 차단

```
문제:
특정 IP (5.6.7.8)에서 DDoS 공격

해결:
Network ACL로 명시적 거부 (보안 그룹은 Deny 불가!)

resource "aws_network_acl_rule" "deny_attack" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 50  # 낮은 번호 (우선순위 높음)
  protocol       = "-1"
  rule_action    = "deny"
  cidr_block     = "5.6.7.8/32"
}
```

### 시나리오 2: 3계층 웹 애플리케이션

```
구조:
ALB → 웹서버 → 앱서버 → DB

보안 그룹 설계:

1. ALB 보안 그룹:
   - 인바운드: 80, 443 (0.0.0.0/0)

2. 웹서버 보안 그룹:
   - 인바운드: 80 (sg-alb)
   - 인바운드: 22 (sg-bastion)

3. 앱서버 보안 그룹:
   - 인바운드: 3000 (sg-web)
   - 인바운드: 22 (sg-bastion)

4. DB 보안 그룹:
   - 인바운드: 3306 (sg-app)
```

```hcl
# ALB 보안 그룹
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 웹서버 보안 그룹
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "HTTP from ALB"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  ingress {
    description     = "SSH from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 앱서버 보안 그룹
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "App port from web"
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }

  ingress {
    description     = "SSH from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# DB 보안 그룹
resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "MySQL from app servers only"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # DB는 아웃바운드 불필요 (선택)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 시나리오 3: Bastion Host (Jump Server)

```
문제:
Private 서브넷의 인스턴스에 SSH 접근 필요

해결:
Bastion Host를 Public 서브넷에 배치

┌──────────────────────────────────────┐
│  개발자 PC (1.2.3.4)                  │
└────────┬─────────────────────────────┘
         │ SSH (포트 22)
         ↓
┌──────────────────────────────────────┐
│  Public Subnet                       │
│  ┌────────────────────────────────┐  │
│  │ Bastion Host                   │  │
│  │ 보안 그룹:                      │  │
│  │ - 인바운드: 22 (1.2.3.4/32)    │  │
│  └─────────────┬──────────────────┘  │
└────────────────┼─────────────────────┘
                 │ SSH (포트 22)
                 ↓
┌──────────────────────────────────────┐
│  Private Subnet                      │
│  ┌────────────────────────────────┐  │
│  │ 앱 서버                         │  │
│  │ 보안 그룹:                      │  │
│  │ - 인바운드: 22 (sg-bastion)    │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

```hcl
# Bastion 보안 그룹
resource "aws_security_group" "bastion" {
  name   = "bastion-sg"
  vpc_id = aws_vpc.main.id

  # 내 IP에서만 SSH 허용
  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["1.2.3.4/32"]  # 내 공인 IP
  }

  # Private 인스턴스로 SSH 가능
  egress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # VPC CIDR
  }
}
```

---

## 17.10 보안 Best Practices

### 1. 최소 권한 원칙 (Least Privilege)

```
❌ 나쁜 예:
모든 포트 열기 (0-65535)
모든 IP 허용 (0.0.0.0/0)

✅ 좋은 예:
필요한 포트만 열기 (80, 443)
특정 IP/보안 그룹만 허용
```

### 2. 계층별 분리

```
Public Subnet:
→ 로드 밸런서, Bastion

Private Subnet (App):
→ 애플리케이션 서버

Private Subnet (DB):
→ 데이터베이스
→ 절대 외부 노출 금지!
```

### 3. 보안 그룹 명명 규칙

```
일관된 명명:
sg-{environment}-{service}-{role}

예시:
sg-prod-web-server
sg-dev-api-server
sg-prod-db-mysql
```

### 4. 정기적인 감사

```
주기적으로 확인:
- 불필요한 규칙 제거
- 0.0.0.0/0 허용 검토
- 사용하지 않는 보안 그룹 삭제
```

---

## 핵심 요약

### 보안 그룹 (Security Group)
```
- 인스턴스 레벨 방화벽
- Stateful (응답 자동 허용)
- Allow 규칙만
- 주요 보안 메커니즘 (90%)
```

### Network ACL
```
- 서브넷 레벨 방화벽
- Stateless (양방향 설정 필요)
- Allow + Deny 규칙
- 추가 보안 계층 (10%)
- Ephemeral 포트 주의!
```

### 사용 패턴
```
1. 보안 그룹으로 세밀한 제어
2. NACL로 서브넷 보호
3. 계층별 분리 (Public/Private)
4. 최소 권한 원칙
```

---

## 다음 단계

보안 그룹과 ACL을 이해했으니, 이제 **라우팅 테이블**을 배워봅시다!

👉 [Chapter 18. 라우팅 테이블](18-라우팅테이블.md)

---

## 실습 과제

### 과제 1: 보안 그룹 생성 및 테스트

```bash
# 1. 웹서버 보안 그룹 생성 (AWS CLI)
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server security group" \
  --vpc-id vpc-xxxxx

# 2. 규칙 추가
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### 과제 2: Stateful vs Stateless 실험

```
실험:
1. EC2 인스턴스 생성
2. 보안 그룹: 인바운드 80 허용, 아웃바운드 모두 차단
3. curl로 접속 테스트
   → 정상 작동 (Stateful 때문!)

4. NACL: 인바운드 80 허용, 아웃바운드 차단
5. curl로 접속 테스트
   → 실패! (Stateless, 응답 차단됨)
```

### 과제 3: 3-Tier 아키텍처 보안 설계

```
요구사항:
- ALB
- 웹 서버 (NGINX)
- API 서버 (Node.js)
- DB (PostgreSQL)

보안 그룹 설계:
1. 각 계층별 보안 그룹 정의
2. 최소 권한 적용
3. Terraform 코드 작성
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자, DevOps
**중요도**: ⭐⭐⭐ 매우 중요!
