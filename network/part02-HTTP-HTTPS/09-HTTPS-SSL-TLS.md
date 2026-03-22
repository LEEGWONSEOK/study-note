# Chapter 9. HTTPS와 SSL/TLS

## 9.1 HTTPS란?

### 정의

**HTTPS = HTTP + SSL/TLS**

```
HTTP: HyperText Transfer Protocol
HTTPS: HTTP Secure (보안 버전)

포트:
- HTTP: 80
- HTTPS: 443
```

### 비유: 우편 vs 등기우편

```
HTTP (일반 우편):
┌─────────────────────┐
│ 보낸사람: 철수       │
│ 받는사람: 영희       │
│ 내용: "비밀번호 1234"│  ← 누구나 볼 수 있음!
└─────────────────────┘

HTTPS (암호화된 등기우편):
┌─────────────────────┐
│ 보낸사람: 철수       │
│ 받는사람: 영희       │
│ 내용: "#$@!%^&*"    │  ← 암호화됨
│ (영희만 열 수 있는 자물쇠)│
└─────────────────────┘
```

---

## 9.2 왜 HTTPS가 필요한가?

### HTTP의 문제점

```
1. 도청 (Eavesdropping)
   → 중간에서 데이터를 읽을 수 있음

┌─────────┐         ┌─────────┐         ┌─────────┐
│ 클라이언트│ -----→ │ 해커 👀 │ -----→ │  서버   │
└─────────┘         └─────────┘         └─────────┘
"비밀번호: 1234"를 해커가 모두 볼 수 있음!


2. 변조 (Tampering)
   → 중간에서 데이터를 바꿀 수 있음

┌─────────┐         ┌─────────┐         ┌─────────┐
│ 클라이언트│ -----→ │ 해커 ✏️  │ -----→ │  서버   │
└─────────┘         └─────────┘         └─────────┘
"계좌번호: 1234" → "계좌번호: 9999"


3. 위장 (Impersonation)
   → 가짜 서버로 속일 수 있음

┌─────────┐                     ┌─────────┐
│ 클라이언트│ ----------------→ │가짜 은행 │
└─────────┘                     └─────────┘
"진짜 은행인 줄 알았는데 피싱 사이트!"
```

### HTTPS의 해결책

```
1. 암호화 (Encryption)
   → 데이터를 읽을 수 없게 만듦

2. 무결성 (Integrity)
   → 데이터 변조를 감지

3. 인증 (Authentication)
   → 서버가 진짜인지 확인
```

---

## 9.3 SSL과 TLS

### 역사

```
SSL (Secure Sockets Layer):
- SSL 1.0: 출시 안 됨 (보안 문제)
- SSL 2.0: 1995년 (사용 중단)
- SSL 3.0: 1996년 (사용 중단)

TLS (Transport Layer Security):
- TLS 1.0: 1999년 (사용 중단)
- TLS 1.1: 2006년 (사용 중단)
- TLS 1.2: 2008년 ✅ 현재 사용
- TLS 1.3: 2018년 ✅ 최신, 가장 빠름

현재 표준: TLS
하지만 관습적으로 "SSL"이라고 많이 부름
```

### 프로토콜 스택 위치

```
┌─────────────────────────────────┐
│  Application Layer (HTTP)       │
├─────────────────────────────────┤
│  TLS/SSL Layer                  │  ← 여기!
├─────────────────────────────────┤
│  Transport Layer (TCP)          │
├─────────────────────────────────┤
│  Network Layer (IP)             │
└─────────────────────────────────┘
```

---

## 9.4 HTTPS 동작 원리

### 전체 흐름

```
1. TCP 연결 (3-way handshake)
2. TLS 핸드셰이크 (암호화 준비)
3. HTTP 통신 (암호화된 상태)
4. 연결 종료
```

### TLS 핸드셰이크 ⭐ 핵심!

```
┌─────────────────────────────────────────────┐
│  클라이언트                      서버        │
├─────────────────────────────────────────────┤
│                                             │
│  1. Client Hello                            │
│  ─────────────────────────→                 │
│  "안녕! 나는 TLS 1.3 지원해"                │
│  - 지원하는 암호화 방식 목록                 │
│  - 랜덤 데이터                              │
│                                             │
│  2. Server Hello                            │
│  ←─────────────────────────                 │
│  "나도 안녕! TLS 1.3 쓰자"                  │
│  - 선택한 암호화 방식                       │
│  - 서버 인증서 (SSL 인증서)                 │
│  - 랜덤 데이터                              │
│                                             │
│  3. 인증서 검증                              │
│  (클라이언트가 서버 인증서 확인)             │
│  ✅ 진짜 서버 맞음!                         │
│                                             │
│  4. 키 교환                                 │
│  ←──────────────────────→                   │
│  "이 키로 암호화하자" (대칭키 생성)          │
│                                             │
│  5. Finished                                │
│  ─────────────────────────→                 │
│  "준비 완료!"                               │
│                                             │
│  ←─────────────────────────                 │
│  "나도 준비 완료!"                          │
│                                             │
│  6. 암호화된 HTTP 통신 시작                  │
│  ←──────────────────────→                   │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 9.5 암호화 방식

### 대칭키 암호화 (Symmetric Encryption)

```
같은 키로 암호화/복호화

┌─────────────┐                   ┌─────────────┐
│  클라이언트  │                   │    서버     │
│             │                   │             │
│  암호화키: K │                   │ 암호화키: K │
└──────┬──────┘                   └──────┬──────┘
       │                                 │
       │  "Hello" + K = "#$@!"          │
       │  ──────────────────────→       │
       │                                 │
       │       "#$@!" + K = "Hello"     │
       │                                 │
       └─────────────────────────────────┘

장점: 빠름 ⚡
단점: 키를 안전하게 공유하기 어려움

예: AES, ChaCha20
```

### 비대칭키 암호화 (Asymmetric Encryption)

```
공개키(Public Key)와 개인키(Private Key) 쌍

┌─────────────┐                   ┌─────────────┐
│  클라이언트  │                   │    서버     │
│             │                   │             │
│             │                   │ 개인키: K_priv│
│             │                   │ 공개키: K_pub│
└──────┬──────┘                   └──────┬──────┘
       │                                 │
       │  "서버의 공개키 주세요"          │
       │  ──────────────────────→       │
       │                                 │
       │  ←──────────────────────       │
       │        K_pub                   │
       │                                 │
       │  "Hello" + K_pub = "#$@!"      │
       │  ──────────────────────→       │
       │                                 │
       │    "#$@!" + K_priv = "Hello"   │
       │    (개인키로만 복호화 가능!)    │
       │                                 │
       └─────────────────────────────────┘

장점: 키 공유 안전
단점: 느림 🐌

예: RSA, ECC (Elliptic Curve Cryptography)
```

### HTTPS에서의 조합 사용

```
TLS 핸드셰이크: 비대칭키 암호화
→ 대칭키를 안전하게 공유

실제 HTTP 통신: 대칭키 암호화
→ 빠른 속도

┌──────────────────────────────────┐
│  1. 비대칭키로 대칭키 교환        │
│     RSA/ECC로 AES 키 전달         │
├──────────────────────────────────┤
│  2. 대칭키로 HTTP 통신            │
│     AES로 빠르게 암호화/복호화    │
└──────────────────────────────────┘

→ 보안 + 성능 모두 확보!
```

---

## 9.6 SSL/TLS 인증서

### 인증서란?

```
서버의 신원을 증명하는 전자 문서

비유: 주민등록증

┌─────────────────────────────────┐
│  SSL 인증서                      │
├─────────────────────────────────┤
│  발급 대상: www.example.com     │
│  발급 기관: Let's Encrypt        │
│  유효 기간: 2024.01.01 ~ 2024.12.31│
│  공개키: RSA 2048 bit            │
│  서명: [발급 기관의 디지털 서명]  │
└─────────────────────────────────┘
```

### 인증서 체인 (Certificate Chain)

```
┌─────────────────────────────────┐
│  Root CA (최상위 인증 기관)      │
│  - 브라우저에 내장됨             │
│  - 절대적으로 신뢰               │
└────────┬────────────────────────┘
         │ 서명
┌────────┴────────────────────────┐
│  Intermediate CA (중간 인증 기관)│
│  - Root CA가 서명함              │
└────────┬────────────────────────┘
         │ 서명
┌────────┴────────────────────────┐
│  Server Certificate (서버 인증서)│
│  - Intermediate CA가 서명함      │
│  - www.example.com               │
└─────────────────────────────────┘

신뢰 체인:
Root CA를 신뢰
→ Intermediate CA를 신뢰
→ Server Certificate를 신뢰
```

### 인증서 발급 기관 (CA)

```
상용 CA:
- DigiCert
- GlobalSign
- Comodo
- GoDaddy

무료 CA:
- Let's Encrypt ✅ 가장 많이 사용
- ZeroSSL

가격:
- Let's Encrypt: 무료
- 상용 CA: 연간 $50 ~ $1000+
```

---

## 9.7 HTTPS 적용하기

### 방법 1: Let's Encrypt + Certbot

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install certbot python3-certbot-nginx

# 인증서 발급 (NGINX)
sudo certbot --nginx -d example.com -d www.example.com

# 인증서 발급 (standalone)
sudo certbot certonly --standalone -d example.com

# 자동 갱신 (Let's Encrypt는 90일마다 만료)
sudo certbot renew --dry-run

# Cron으로 자동 갱신 설정
sudo crontab -e
# 매일 새벽 2시에 체크
0 2 * * * certbot renew --quiet
```

### 방법 2: 클라우드 서비스

```
AWS:
- ACM (AWS Certificate Manager)
  → 무료, 자동 갱신
  → ALB/CloudFront에서 사용

GCP:
- Google-managed SSL certificates
  → 무료, 자동 갱신

Cloudflare:
- Universal SSL
  → 무료, 자동 갱신
```

### NGINX 설정

```nginx
# HTTP → HTTPS 리다이렉트
server {
    listen 80;
    server_name example.com www.example.com;

    # 모든 HTTP 요청을 HTTPS로 리다이렉트
    return 301 https://$server_name$request_uri;
}

# HTTPS 서버
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL 인증서 경로
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # SSL 프로토콜 (TLS 1.2, 1.3만 허용)
    ssl_protocols TLSv1.2 TLSv1.3;

    # 암호화 알고리즘 (강력한 것만)
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers on;

    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # OCSP Stapling (인증서 검증 성능 향상)
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # 애플리케이션 설정
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Node.js (Express) 설정

```javascript
const express = require('express');
const https = require('https');
const fs = require('fs');

const app = express();

// HTTPS 옵션
const httpsOptions = {
  key: fs.readFileSync('/etc/letsencrypt/live/example.com/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/example.com/fullchain.pem')
};

// 라우트
app.get('/', (req, res) => {
  res.send('Hello HTTPS!');
});

// HTTPS 서버 시작
https.createServer(httpsOptions, app).listen(443, () => {
  console.log('HTTPS server running on port 443');
});

// HTTP → HTTPS 리다이렉트
const http = require('http');
http.createServer((req, res) => {
  res.writeHead(301, { Location: `https://${req.headers.host}${req.url}` });
  res.end();
}).listen(80);
```

---

## 9.8 HTTPS 성능 최적화

### TLS 1.3 사용

```
TLS 1.2 vs TLS 1.3:

TLS 1.2: 2-RTT (왕복 2번)
┌──────────┐                    ┌──────────┐
│ 클라이언트│                    │   서버   │
└─────┬────┘                    └─────┬────┘
      │  1. Client Hello             │
      │  ──────────────────────────→ │
      │                               │
      │  2. Server Hello              │
      │  ←────────────────────────── │
      │                               │
      │  3. Key Exchange              │
      │  ──────────────────────────→ │
      │                               │
      │  4. Finished                  │
      │  ←────────────────────────── │
      │                               │
      │  5. HTTP 요청 시작            │

TLS 1.3: 1-RTT (왕복 1번) ⚡
┌──────────┐                    ┌──────────┐
│ 클라이언트│                    │   서버   │
└─────┬────┘                    └─────┬────┘
      │  1. Client Hello + Key       │
      │  ──────────────────────────→ │
      │                               │
      │  2. Server Hello + Finished   │
      │  ←────────────────────────── │
      │                               │
      │  3. HTTP 요청 시작 (즉시!)    │

→ 50% 빠른 핸드셰이크!
```

### Session Resumption (세션 재사용)

```
첫 방문:
- 전체 TLS 핸드셰이크 수행
- Session ID 저장

재방문:
- Session ID 재사용
- 핸드셰이크 생략 → 빠름!

┌──────────┐                    ┌──────────┐
│ 클라이언트│                    │   서버   │
└─────┬────┘                    └─────┬────┘
      │  "Session ID: abc123"        │
      │  ──────────────────────────→ │
      │                               │
      │  "OK, 재사용!"                │
      │  ←────────────────────────── │
      │                               │
      │  즉시 HTTP 통신 시작          │
```

### HTTP/2 사용

```
NGINX 설정:
listen 443 ssl http2;  ← http2 추가

HTTP/2 장점:
- 멀티플렉싱 (동시 요청)
- 헤더 압축
- 서버 푸시
```

---

## 9.9 보안 헤더

### HSTS (HTTP Strict Transport Security)

```
역할: 브라우저가 항상 HTTPS만 사용하도록 강제

Strict-Transport-Security: max-age=31536000; includeSubDomains

max-age: 1년 동안 유효
includeSubDomains: 서브도메인도 포함

효과:
- HTTP 요청을 자동으로 HTTPS로 변환
- 중간자 공격 방지
```

```javascript
// Express
app.use((req, res, next) => {
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains'
  );
  next();
});
```

### CSP (Content Security Policy)

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com

역할: XSS 공격 방어
```

---

## 9.10 HTTPS 디버깅

### 인증서 확인

```bash
# 인증서 정보 확인
openssl s_client -connect example.com:443 -showcerts

# 인증서 만료일 확인
openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# 결과:
# notBefore=Jan  1 00:00:00 2024 GMT
# notAfter=Dec 31 23:59:59 2024 GMT

# 인증서 체인 확인
curl --verbose https://example.com
```

### 브라우저에서 확인

```
Chrome:
1. 주소창 자물쇠 아이콘 클릭
2. "인증서" 클릭
3. 세부 정보 확인
   - 발급 기관
   - 유효 기간
   - 공개키 정보
```

### SSL Labs 테스트

```
https://www.ssllabs.com/ssltest/

점수 확인:
A+: 완벽
A: 좋음
B 이하: 문제 있음 (보안 설정 재검토 필요)
```

---

## 9.11 문제 해결

### 문제 1: Mixed Content

```
문제:
HTTPS 페이지에서 HTTP 리소스 로드

예:
https://example.com 에서
<img src="http://image.com/photo.jpg">
→ 브라우저가 차단!

해결:
1. 모든 리소스를 HTTPS로 변경
<img src="https://image.com/photo.jpg">

2. 프로토콜 생략 (상대 프로토콜)
<img src="//image.com/photo.jpg">
→ 페이지 프로토콜을 따름
```

### 문제 2: 인증서 만료

```
증상:
"Your connection is not private"
NET::ERR_CERT_DATE_INVALID

해결:
# Let's Encrypt 갱신
sudo certbot renew

# 자동 갱신 확인
sudo certbot renew --dry-run
```

### 문제 3: 중간 인증서 누락

```
문제:
일부 브라우저/디바이스에서만 인증서 오류

해결:
fullchain.pem 사용 (중간 인증서 포함)

# NGINX
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
# cert.pem이 아닌 fullchain.pem!
```

---

## 9.12 실전 시나리오

### 시나리오 1: 기존 HTTP 사이트를 HTTPS로 전환

```
단계:

1. 인증서 발급
sudo certbot --nginx -d example.com

2. NGINX 설정 업데이트
(자동으로 업데이트됨)

3. HTTP → HTTPS 리다이렉트 확인
curl -I http://example.com
# Location: https://example.com

4. HSTS 헤더 추가

5. 외부 리소스 HTTPS 변환
- Google Analytics
- CDN (폰트, 이미지)
- API 호출

6. 테스트
- SSL Labs 점수 확인
- 모든 페이지 동작 확인
- Mixed Content 오류 확인
```

### 시나리오 2: 멀티 도메인 인증서

```
하나의 인증서로 여러 도메인 커버

# SAN (Subject Alternative Names) 인증서
sudo certbot --nginx \
  -d example.com \
  -d www.example.com \
  -d api.example.com \
  -d admin.example.com

# 와일드카드 인증서
sudo certbot certonly --manual \
  --preferred-challenges dns \
  -d *.example.com
```

---

## 핵심 요약

### HTTPS란?
```
HTTP + SSL/TLS
암호화 + 무결성 + 인증
포트 443
```

### TLS 핸드셰이크
```
1. Client Hello
2. Server Hello + 인증서
3. 키 교환
4. 암호화 통신 시작
```

### 암호화
```
비대칭키: 키 교환 (RSA, ECC)
대칭키: 실제 통신 (AES)
```

### 인증서
```
서버 신원 증명
CA가 발급
Let's Encrypt: 무료, 90일
```

### 성능 최적화
```
TLS 1.3: 1-RTT
Session Resumption
HTTP/2
```

---

## 다음 단계

HTTPS를 이해했으니, 이제 **HTTP/2와 HTTP/3**를 배워봅시다!

👉 [Chapter 10. HTTP/2와 HTTP/3](10-HTTP2-HTTP3.md)

---

## 실습 과제

### 과제 1: Let's Encrypt 인증서 발급

```bash
# 1. Certbot 설치
sudo apt install certbot

# 2. Standalone 모드로 인증서 발급
# (테스트 도메인 또는 실제 도메인)
sudo certbot certonly --standalone -d yourdomain.com

# 3. 발급된 인증서 확인
sudo ls -la /etc/letsencrypt/live/yourdomain.com/
```

### 과제 2: NGINX HTTPS 설정

```nginx
# /etc/nginx/sites-available/yourdomain.com

# HTTP → HTTPS 리다이렉트
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

### 과제 3: SSL Labs 테스트

```
1. https://www.ssllabs.com/ssltest/ 접속
2. 본인 도메인 입력
3. 점수 확인 (목표: A 이상)
4. 문제점 개선
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 3시간
**대상**: 모든 웹개발자
**중요도**: ⭐⭐⭐ 매우 중요!