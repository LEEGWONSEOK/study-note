# Chapter 13. DNS 레코드 타입

## 13.1 DNS 레코드란?

### 정의

```
DNS 레코드 = 도메인 이름과 연결된 정보

비유: 전화번호부의 항목

┌───────────────────────────────────┐
│  전화번호부                        │
├───────────────────────────────────┤
│  이름      → 전화번호 (A 레코드)   │
│  회사      → 주소 (MX 레코드)      │
│  별명      → 실명 (CNAME 레코드)   │
└───────────────────────────────────┘
```

### 레코드 구조

```
형식:
이름   TTL   클래스   타입   값

예시:
example.com.  300  IN  A  93.184.216.34

이름: example.com.
TTL: 300초 (5분)
클래스: IN (Internet, 거의 항상 IN)
타입: A (IPv4 주소)
값: 93.184.216.34
```

---

## 13.2 A 레코드 ⭐ 가장 중요!

### 개념

```
A = Address
도메인 → IPv4 주소 매핑

┌──────────────┐         ┌──────────────┐
│ example.com  │  ────→  │ 93.184.216.34│
└──────────────┘         └──────────────┘
```

### 예시

```
example.com.     300  IN  A  93.184.216.34
www.example.com. 300  IN  A  93.184.216.34
api.example.com. 300  IN  A  10.0.1.100
```

### 설정 방법 (Route 53)

```hcl
# Terraform
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"
  ttl     = 300
  records = ["93.184.216.34"]
}
```

### 다중 A 레코드 (Round Robin)

```
example.com.  300  IN  A  93.184.216.34
example.com.  300  IN  A  93.184.216.35
example.com.  300  IN  A  93.184.216.36

→ DNS가 번갈아 응답
→ 간단한 로드 밸런싱
```

---

## 13.3 AAAA 레코드

### 개념

```
AAAA = Address (IPv6)
도메인 → IPv6 주소 매핑

A 레코드의 IPv6 버전
```

### 예시

```
example.com.  300  IN  AAAA  2606:2800:220:1:248:1893:25c8:1946

IPv6 주소:
- 128bit (IPv4는 32bit)
- 예: 2001:db8::1
```

### 설정

```hcl
resource "aws_route53_record" "ipv6" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "AAAA"
  ttl     = 300
  records = ["2606:2800:220:1:248:1893:25c8:1946"]
}
```

---

## 13.4 CNAME 레코드

### 개념

```
CNAME = Canonical Name (정식 이름)
도메인 → 다른 도메인 매핑

비유: 별명 → 본명

별명: blog.example.com
본명: example.github.io
```

### 예시

```
blog.example.com.  300  IN  CNAME  example.github.io.
www.example.com.   300  IN  CNAME  example.com.
cdn.example.com.   300  IN  CNAME  d111111abcdef8.cloudfront.net.
```

### CNAME 체인

```
요청: blog.example.com

1. blog.example.com → CNAME → example.github.io
2. example.github.io → A → 185.199.108.153

최종: 185.199.108.153
```

### 중요: CNAME 제약

```
❌ Root 도메인에는 CNAME 불가!
example.com.  IN  CNAME  other.com.  (불가능)

✅ 서브도메인만 가능
www.example.com.  IN  CNAME  example.com.  (가능)

이유: DNS 표준 (RFC)
→ Root에는 A 또는 ALIAS 레코드 사용
```

---

## 13.5 ALIAS 레코드 (AWS Route 53 전용)

### 개념

```
CNAME과 유사하지만 Root 도메인 가능
AWS Route 53의 확장 기능
```

### A vs CNAME vs ALIAS

```
┌──────────┬────────────┬────────────┬────────────┐
│   항목    │     A      │   CNAME    │   ALIAS    │
├──────────┼────────────┼────────────┼────────────┤
│ Root 가능 │     ✅    │     ❌    │     ✅    │
│ 대상      │ IP 주소    │ 도메인     │ AWS 리소스 │
│ 요금      │ 유료       │ 유료       │ 무료       │
│ 표준      │ DNS 표준   │ DNS 표준   │ AWS 전용   │
└──────────┴────────────┴────────────┴────────────┘
```

### 사용 예시

```hcl
# ALB를 Root 도메인에 연결
resource "aws_route53_record" "root" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# CloudFront
resource "aws_route53_record" "cdn" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}
```

---

## 13.6 MX 레코드 (Mail Exchange)

### 개념

```
이메일 서버 지정

user@example.com 으로 메일 전송 시
→ example.com의 MX 레코드 조회
→ 해당 메일 서버로 전달
```

### 우선순위

```
example.com.  300  IN  MX  10  mail1.example.com.
example.com.  300  IN  MX  20  mail2.example.com.
example.com.  300  IN  MX  30  mail3.example.com.

숫자가 낮을수록 우선순위 높음:
1순위: mail1 (10)
2순위: mail2 (20)
3순위: mail3 (30, 백업)

mail1 실패 → mail2 시도
mail2 실패 → mail3 시도
```

### Gmail 설정 예시

```
example.com.  3600  IN  MX  1  aspmx.l.google.com.
example.com.  3600  IN  MX  5  alt1.aspmx.l.google.com.
example.com.  3600  IN  MX  5  alt2.aspmx.l.google.com.
example.com.  3600  IN  MX  10 alt3.aspmx.l.google.com.
example.com.  3600  IN  MX  10 alt4.aspmx.l.google.com.
```

---

## 13.7 TXT 레코드

### 개념

```
텍스트 정보 저장
용도:
- 도메인 소유권 인증
- SPF (이메일 스팸 방지)
- DKIM (이메일 서명)
- 기타 메타 정보
```

### 도메인 소유권 인증

```
Google Search Console:
example.com.  300  IN  TXT  "google-site-verification=abc123..."

AWS SES:
example.com.  300  IN  TXT  "amazonses:abc123..."

Cloudflare:
example.com.  300  IN  TXT  "cloudflare-verify=xyz789..."
```

### SPF (Sender Policy Framework)

```
이메일 발신 서버 지정
→ 스팸 방지

example.com.  300  IN  TXT  "v=spf1 include:_spf.google.com ~all"

의미:
- v=spf1: SPF 버전 1
- include:_spf.google.com: Gmail 서버 허용
- ~all: 나머지는 소프트 실패 (스팸 가능성)
```

### DKIM

```
이메일 디지털 서명

default._domainkey.example.com.  300  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0..."

발신 서버가 서명 → 수신 서버가 검증
→ 이메일 위변조 방지
```

---

## 13.8 NS 레코드 (Name Server)

### 개념

```
권한 있는 네임서버 지정

example.com.  172800  IN  NS  ns1.example.com.
example.com.  172800  IN  NS  ns2.example.com.

→ example.com의 DNS 레코드는
  ns1, ns2에서 관리함을 알림
```

### 서브도메인 위임

```
부모 도메인: example.com (Route 53에서 관리)
자식 도메인: blog.example.com (다른 DNS에서 관리)

example.com의 레코드:
blog.example.com.  300  IN  NS  ns1.blogger.com.
blog.example.com.  300  IN  NS  ns2.blogger.com.

→ blog.example.com의 DNS 쿼리는
  Blogger의 네임서버로 전달
```

---

## 13.9 SOA 레코드 (Start of Authority)

### 개념

```
Zone의 기본 정보

example.com.  3600  IN  SOA  ns1.example.com. admin.example.com. (
  2024010101  ; Serial (버전)
  3600        ; Refresh (갱신 주기)
  600         ; Retry (재시도)
  604800      ; Expire (만료)
  300         ; Minimum TTL
)
```

### 필드 설명

```
- Primary NS: ns1.example.com. (주 네임서버)
- Admin Email: admin.example.com. (@를 .으로 표기)
- Serial: 2024010101 (변경 시마다 증가)
- Refresh: 3600초 (1시간, 슬레이브가 체크)
- Retry: 600초 (10분, 실패 시 재시도)
- Expire: 604800초 (7일, 만료 기간)
- Minimum TTL: 300초 (5분, 네거티브 캐싱)
```

---

## 13.10 SRV 레코드 (Service)

### 개념

```
특정 서비스의 위치와 포트 지정

형식:
_service._protocol.domain.  TTL  IN  SRV  priority weight port target

예:
_minecraft._tcp.example.com.  300  IN  SRV  10 60 25565 mc1.example.com.
```

### 사용 예시

```
# Minecraft 서버
_minecraft._tcp.example.com.  300  IN  SRV  10 60 25565 mc.example.com.

# SIP (VoIP)
_sip._tcp.example.com.  300  IN  SRV  10 60 5060 sip.example.com.

# XMPP (메신저)
_xmpp-client._tcp.example.com.  300  IN  SRV  10 60 5222 xmpp.example.com.

필드:
- Priority: 10 (우선순위, 낮을수록 우선)
- Weight: 60 (가중치, 같은 우선순위 내에서)
- Port: 25565 (서비스 포트)
- Target: mc.example.com. (실제 호스트)
```

---

## 13.11 CAA 레코드 (Certification Authority Authorization)

### 개념

```
SSL 인증서 발급 기관 제한
→ 보안 강화

example.com.  300  IN  CAA  0 issue "letsencrypt.org"
```

### 예시

```
# Let's Encrypt만 허용
example.com.  300  IN  CAA  0 issue "letsencrypt.org"

# 여러 CA 허용
example.com.  300  IN  CAA  0 issue "letsencrypt.org"
example.com.  300  IN  CAA  0 issue "digicert.com"

# 와일드카드 인증서 허용
example.com.  300  IN  CAA  0 issuewild "letsencrypt.org"

# 위반 시 알림
example.com.  300  IN  CAA  0 iodef "mailto:security@example.com"
```

---

## 13.12 실전 시나리오

### 시나리오 1: 웹사이트 + 이메일

```
목표:
- www.example.com → 웹서버 (AWS ALB)
- example.com → www.example.com 리다이렉트
- mail.example.com → Gmail

DNS 레코드:

# 웹사이트 (ALIAS로 ALB 연결)
example.com.      300  IN  A     ALIAS alb.amazonaws.com
www.example.com.  300  IN  CNAME example.com.

# 이메일 (Gmail)
example.com.  3600  IN  MX  1  aspmx.l.google.com.
example.com.  3600  IN  MX  5  alt1.aspmx.l.google.com.

# SPF (Gmail 발신 허용)
example.com.  300  IN  TXT  "v=spf1 include:_spf.google.com ~all"
```

### 시나리오 2: 멀티 리전 서비스

```
목표:
- 한국: kr.example.com
- 미국: us.example.com
- 글로벌: example.com (자동 라우팅)

# 리전별 서버
kr.example.com.  300  IN  A  13.125.1.1
us.example.com.  300  IN  A  3.35.1.1

# Geolocation (Route 53)
# 한국 사용자 → 한국 서버
# 미국 사용자 → 미국 서버
```

### 시나리오 3: CDN + Origin 분리

```
목표:
- 정적 파일: CDN (CloudFront)
- API: Origin 서버

# 메인 사이트 (CloudFront)
example.com.      300  IN  A     ALIAS d111.cloudfront.net
www.example.com.  300  IN  CNAME d111.cloudfront.net

# API 서버 (직접 연결)
api.example.com.  300  IN  A  13.125.1.100

# CDN 정적 파일
cdn.example.com.  300  IN  CNAME d222.cloudfront.net
```

---

## 13.13 DNS 레코드 확인

### dig 명령어

```bash
# A 레코드
dig example.com A

# 결과:
# example.com.  300  IN  A  93.184.216.34

# MX 레코드
dig example.com MX

# 모든 레코드
dig example.com ANY

# 특정 네임서버로 쿼리
dig @8.8.8.8 example.com

# 짧은 형식
dig example.com +short
# 93.184.216.34
```

### nslookup

```bash
# A 레코드
nslookup example.com

# MX 레코드
nslookup -type=MX example.com

# NS 레코드
nslookup -type=NS example.com

# 특정 DNS 서버 사용
nslookup example.com 8.8.8.8
```

### 온라인 도구

```
DNS 조회 도구:
- https://dnschecker.org/
- https://www.whatsmydns.net/
- https://mxtoolbox.com/

기능:
- 전 세계 DNS 전파 확인
- MX, SPF, DKIM 검증
- DNS 레코드 분석
```

---

## 13.14 Terraform 설정 예시

### Route 53 Zone + 레코드

```hcl
# Zone 생성
resource "aws_route53_zone" "main" {
  name = "example.com"

  tags = {
    Environment = "production"
  }
}

# A 레코드 (ALIAS to ALB)
resource "aws_route53_record" "root" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# CNAME 레코드
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "CNAME"
  ttl     = 300
  records = ["example.com"]
}

# MX 레코드 (Gmail)
resource "aws_route53_record" "mx" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "MX"
  ttl     = 3600
  records = [
    "1 aspmx.l.google.com",
    "5 alt1.aspmx.l.google.com",
    "5 alt2.aspmx.l.google.com",
  ]
}

# TXT 레코드 (SPF)
resource "aws_route53_record" "spf" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "TXT"
  ttl     = 300
  records = [
    "v=spf1 include:_spf.google.com ~all"
  ]
}

# CAA 레코드
resource "aws_route53_record" "caa" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "CAA"
  ttl     = 300
  records = [
    "0 issue \"letsencrypt.org\"",
    "0 issuewild \"letsencrypt.org\""
  ]
}
```

---

## 핵심 요약

### 주요 레코드 타입

```
A: 도메인 → IPv4
AAAA: 도메인 → IPv6
CNAME: 도메인 → 도메인 (별명)
ALIAS: AWS 전용, Root 도메인 가능
MX: 이메일 서버
TXT: 텍스트 정보 (SPF, DKIM)
NS: 네임서버 지정
SOA: Zone 기본 정보
CAA: SSL 인증서 발급 기관 제한
```

### 레코드 선택 가이드

```
웹사이트:
- Root: A 또는 ALIAS
- www: CNAME

이메일:
- MX (메일 서버)
- TXT (SPF, DKIM)

CDN:
- CNAME

서브도메인 위임:
- NS
```

---

## 다음 단계

DNS 레코드 타입을 마스터했으니, 이제 **도메인 구매와 설정**을 배워봅시다!

👉 [Chapter 14. 도메인 구매와 설정](14-도메인-설정.md)

---

## 실습 과제

### 과제 1: DNS 레코드 조회

```bash
# 1. A 레코드 확인
dig google.com A

# 2. MX 레코드 확인
dig google.com MX

# 3. TXT 레코드 확인 (SPF)
dig google.com TXT

# 4. NS 레코드 확인
dig google.com NS

# 5. 모든 레코드
dig google.com ANY
```

### 과제 2: Route 53 설정

```hcl
# Terraform으로 본인 도메인 설정
# 1. Zone 생성
# 2. A 레코드 (웹사이트)
# 3. MX 레코드 (이메일)
# 4. TXT 레코드 (SPF)
# 5. CAA 레코드 (보안)
```

### 과제 3: DNS 전파 확인

```
1. https://dnschecker.org/ 접속
2. 본인 도메인 입력
3. 레코드 타입 선택
4. 전 세계 DNS 서버에서 확인
5. 전파 상태 체크
```

---

**작성일**: 2024년
**난이도**: ⭐⭐ 초중급
**예상 학습 시간**: 2-3시간
**대상**: 모든 웹개발자
**중요도**: ⭐⭐⭐ 매우 중요!