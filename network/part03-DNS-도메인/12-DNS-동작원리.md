# Chapter 12. DNS 동작 원리

## 12.1 DNS란?

**DNS (Domain Name System)**는 도메인 이름을 IP 주소로 변환하는 시스템입니다.

### 비유: 전화번호부

```
전화번호부:
김철수 → 010-1234-5678
이영희 → 010-9876-5432

DNS:
google.com → 142.250.207.46
naver.com → 223.130.195.95
```

---

## 12.2 왜 DNS가 필요한가?

### 사람 vs 컴퓨터

```
사람: 이름 기억에 유리
→ "google.com" 쉬움
→ "142.250.207.46" 어려움

컴퓨터: 숫자 처리에 유리
→ IP 주소로만 통신 가능

DNS = 둘 사이의 번역기!
```

### DNS 없다면?

```
❌ IP 주소를 외워야 함
http://142.250.207.46  (Google)
http://223.130.195.95  (Naver)

❌ IP 변경 시 모든 사용자가 새 주소를 알아야 함
```

---

## 12.3 DNS 동작 과정 ⭐ 핵심!

### 전체 흐름

```
브라우저에 www.example.com 입력
              ↓
┌──────────────────────────────────────┐
│ 1. 브라우저 캐시 확인                 │
│    "최근에 방문했나?"                 │
└───────┬──────────────────────────────┘
        ↓ 없음
┌──────────────────────────────────────┐
│ 2. OS 캐시 확인                      │
│    "이 PC가 아는 주소인가?"           │
└───────┬──────────────────────────────┘
        ↓ 없음
┌──────────────────────────────────────┐
│ 3. 라우터 캐시 확인                   │
│    "공유기가 아는 주소인가?"           │
└───────┬──────────────────────────────┘
        ↓ 없음
┌──────────────────────────────────────┐
│ 4. ISP DNS 서버 (Recursive)          │
│    "KT/SK 등 통신사 DNS"             │
└───────┬──────────────────────────────┘
        ↓
┌──────────────────────────────────────┐
│ 5. Root DNS 서버                     │
│    ".com은 어디로 가야 하나요?"       │
│    → "여기로 가세요: .com DNS 주소"   │
└───────┬──────────────────────────────┘
        ↓
┌──────────────────────────────────────┐
│ 6. TLD DNS 서버 (.com)               │
│    "example.com은 어디 있나요?"      │
│    → "example.com DNS 주소입니다"    │
└───────┬──────────────────────────────┘
        ↓
┌──────────────────────────────────────┐
│ 7. Authoritative DNS 서버            │
│    (example.com 관리 서버)           │
│    "www.example.com의 IP는?"        │
│    → "93.184.216.34입니다"          │
└───────┬──────────────────────────────┘
        ↓
    IP 주소 획득!
        ↓
    서버에 접속
```

### 상세 단계별 설명

#### 1단계: 브라우저 캐시

```javascript
// Chrome에서 DNS 캐시 확인
chrome://net-internals/#dns

// 캐시 삭제
chrome://net-internals/#dns
→ Clear host cache 클릭
```

#### 2단계: OS 캐시

```bash
# macOS/Linux - hosts 파일
cat /etc/hosts

# 예시:
127.0.0.1       localhost
192.168.0.10    myserver.local

# DNS 캐시 확인 (macOS)
dscacheutil -cachedump

# DNS 캐시 삭제 (macOS)
sudo dscacheutil -flushcache

# Windows
ipconfig /displaydns   # 캐시 확인
ipconfig /flushdns     # 캐시 삭제
```

#### 3-7단계: 재귀적 조회

```
┌─────────────────────────────────────────┐
│  Recursive DNS (ISP DNS)                │
│  - 대신 찾아주는 서버                    │
│  - 결과를 캐싱                          │
│  - 예: 8.8.8.8 (Google Public DNS)      │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  Root DNS (.  최상위)                   │
│  - 전 세계 13개 클러스터                 │
│  - ".com 도메인은 여기로" 알려줌         │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  TLD DNS (.com, .kr, .org 등)           │
│  - Top Level Domain 관리                │
│  - "example.com은 여기" 알려줌          │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  Authoritative DNS (example.com)        │
│  - 실제 IP 주소 보유                     │
│  - 최종 답변                            │
└─────────────────────────────────────────┘
```

---

## 12.4 DNS 서버 종류

### Recursive DNS (재귀 DNS)

```
역할: 사용자 대신 여러 DNS 서버를 순회하며 조회

예:
- 8.8.8.8 (Google Public DNS)
- 1.1.1.1 (Cloudflare DNS)
- 168.126.63.1 (KT)
- 210.220.163.82 (SK)
```

```bash
# DNS 서버 변경 (macOS)
System Preferences → Network → Advanced → DNS
→ 8.8.8.8 추가

# 확인
nslookup google.com
# Server: 8.8.8.8
```

### Authoritative DNS

```
역할: 도메인의 실제 DNS 레코드 관리

예:
- Route 53 (AWS)
- Cloud DNS (Google)
- Cloudflare DNS
```

---

## 12.5 DNS 조회 명령어

### nslookup

```bash
# 기본 조회
nslookup google.com

# 결과:
# Server:  8.8.8.8
# Address: 8.8.8.8#53
#
# Non-authoritative answer:
# Name:    google.com
# Address: 142.250.207.46

# 특정 DNS 서버로 조회
nslookup google.com 1.1.1.1

# 특정 레코드 타입 조회
nslookup -type=MX google.com  # 메일 서버
nslookup -type=NS google.com  # 네임서버
```

### dig (더 상세)

```bash
# 기본 조회
dig google.com

# 결과 예시:
# ;; QUESTION SECTION:
# ;google.com.                   IN      A
#
# ;; ANSWER SECTION:
# google.com.            300     IN      A       142.250.207.46
#
# ;; Query time: 23 msec
# ;; SERVER: 8.8.8.8#53(8.8.8.8)

# 짧은 형식
dig google.com +short
# 142.250.207.46

# 전체 경로 추적
dig google.com +trace
```

### host

```bash
# 간단한 조회
host google.com
# google.com has address 142.250.207.46

# 모든 레코드
host -a google.com
```

---

## 12.6 DNS 캐싱

### TTL (Time To Live)

```
DNS 레코드가 캐시에 유지되는 시간 (초)

예:
google.com. 300 IN A 142.250.207.46
            │
            └─ TTL: 300초 (5분)

5분 후 다시 조회해야 함
```

```bash
# TTL 확인
dig google.com

# 결과:
# google.com.            300     IN      A       142.250.207.46
#                        ↑ TTL
```

### 캐싱 레벨

```
┌──────────────────────────────────┐
│  브라우저 캐시 (가장 빠름)        │
│  TTL: 수 분                      │
└──────────────────────────────────┘
          ↓
┌──────────────────────────────────┐
│  OS 캐시                         │
│  TTL: 수 분~수 시간              │
└──────────────────────────────────┘
          ↓
┌──────────────────────────────────┐
│  라우터 캐시                      │
│  TTL: 수 시간                    │
└──────────────────────────────────┘
          ↓
┌──────────────────────────────────┐
│  ISP DNS 캐시                    │
│  TTL: DNS 레코드 설정에 따름      │
└──────────────────────────────────┘
```

---

## 12.7 DNS 전파 시간

### DNS 레코드 변경 시

```
도메인 IP 변경:
12.34.56.78 → 98.76.54.32

┌──────────────────────────────────┐
│  Authoritative DNS 업데이트      │
│  즉시 반영                       │
└──────────────────────────────────┘
          ↓
┌──────────────────────────────────┐
│  전 세계 캐시 만료 대기           │
│  TTL만큼 시간 소요               │
└──────────────────────────────────┘

TTL이 300초(5분)라면:
→ 최대 5분 후 전 세계 반영

TTL이 86400초(24시간)라면:
→ 최대 24시간 후 반영!
```

### 전파 확인

```bash
# 여러 DNS 서버에서 확인
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
dig @168.126.63.1 example.com

# 전 세계 DNS 전파 확인 (웹)
https://www.whatsmydns.net/
```

---

## 12.8 실전 시나리오

### 시나리오 1: 서버 IP 변경

```
기존 서버: 12.34.56.78
새 서버: 98.76.54.32

1. 새 서버 준비
2. TTL을 짧게 변경 (예: 300초)
3. 24시간 대기 (기존 캐시 만료)
4. DNS 레코드 변경
5. 5분 후 전 세계 반영
6. 기존 서버 종료
7. TTL을 원래대로 (예: 3600초)
```

### 시나리오 2: 무중단 배포

```
Blue-Green 배포:

1. Green 서버 배포
2. DNS를 Green으로 변경
3. TTL 대기
4. 모든 트래픽이 Green으로 이동
5. Blue 서버 종료
```

### 시나리오 3: CDN 설정

```
원본: origin.example.com (12.34.56.78)
CDN: cdn.example.com → CloudFront (여러 IP)

1. CloudFront 배포 생성
2. DNS CNAME 레코드 추가
   cdn.example.com → d111111abcdef8.cloudfront.net
3. 사용자가 cdn.example.com 접속
   → CloudFront의 가장 가까운 엣지로 연결
```

---

## 12.9 웹개발에서 DNS 활용

### 서브도메인 분리

```
www.example.com     → 웹사이트
api.example.com     → API 서버
admin.example.com   → 관리자 페이지
blog.example.com    → 블로그
cdn.example.com     → CDN
```

### 환경별 DNS

```
개발: dev.example.com     → 10.0.1.10
스테이징: staging.example.com → 10.0.2.10
운영: www.example.com     → 13.209.84.49
```

### DNS 라운드 로빈 (부하 분산)

```
example.com의 A 레코드:
- 12.34.56.78
- 98.76.54.32
- 11.22.33.44

DNS가 번갈아 응답 → 트래픽 분산
```

---

## 12.10 DNS 문제 해결

### 문제 1: DNS가 안 풀림

```bash
# 1. 로컬 DNS 캐시 삭제
sudo dscacheutil -flushcache  # macOS
ipconfig /flushdns            # Windows

# 2. DNS 서버 확인
cat /etc/resolv.conf  # macOS/Linux

# 3. 다른 DNS 서버 시도
nslookup example.com 8.8.8.8

# 4. Authoritative DNS 직접 조회
dig example.com +trace
```

### 문제 2: DNS 전파 안 됨

```bash
# 전 세계 DNS 확인
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com

# TTL 확인
dig example.com | grep TTL

# Authoritative DNS 확인
dig example.com NS  # 네임서버 확인
dig @ns1.example.com example.com  # 직접 조회
```

### 문제 3: 느린 DNS 조회

```bash
# DNS 응답 시간 측정
time dig google.com

# 빠른 Public DNS 사용
8.8.8.8      # Google
1.1.1.1      # Cloudflare (가장 빠름)
208.67.222.222  # OpenDNS
```

---

## 핵심 요약

### DNS란?
```
도메인 이름 → IP 주소 변환
인터넷의 전화번호부
```

### DNS 조회 과정
```
1. 브라우저 캐시
2. OS 캐시
3. ISP DNS (Recursive)
4. Root DNS
5. TLD DNS (.com 등)
6. Authoritative DNS (최종)
```

### 주요 명령어
```
nslookup: 기본 조회
dig: 상세 조회
host: 간단한 조회
```

### TTL
```
캐시 유효 시간
짧으면: 빠른 변경, 많은 쿼리
길면: 느린 변경, 적은 쿼리
```

---

## 다음 단계

DNS 동작 원리를 이해했으니, 이제 **DNS 레코드 타입**을 배워봅시다!

👉 [Chapter 13. DNS 레코드 타입](13-DNS-레코드.md)

---

**작성일**: 2024년
**난이도**: ⭐⭐ 초급
**예상 학습 시간**: 2시간
**대상**: 모든 웹개발자