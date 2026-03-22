# Chapter 10. HTTP/2와 HTTP/3

## 10.1 HTTP 버전의 역사

### 타임라인

```
1991년: HTTP/0.9
- GET만 지원
- HTML만 전송

1996년: HTTP/1.0
- 헤더 추가
- POST, HEAD 메서드 추가
- 상태 코드 도입

1997년: HTTP/1.1 ⭐ 15년간 표준
- Keep-Alive (지속 연결)
- 파이프라이닝
- 청크 인코딩

2015년: HTTP/2 ⭐ 현재 널리 사용
- 멀티플렉싱
- 헤더 압축
- 서버 푸시

2022년: HTTP/3 (QUIC) ⭐ 차세대
- UDP 기반
- 더 빠른 핸드셰이크
- 패킷 손실에 강함
```

---

## 10.2 HTTP/1.1의 문제점

### 문제 1: HOL Blocking (Head-of-Line Blocking)

```
비유: 편도 1차선 도로

┌─────────┐         ┌─────────┐
│ 브라우저 │         │  서버   │
└────┬────┘         └────┬────┘
     │                   │
     │ 1. index.html 요청│
     │ ─────────────────→│
     │                   │
     │ 2. index.html 응답│
     │ ←─────────────────│
     │                   │ (여기까지 기다려야 함!)
     │ 3. style.css 요청 │
     │ ─────────────────→│
     │                   │
     │ 4. style.css 응답 │
     │ ←─────────────────│

한 번에 하나씩만 처리 → 느림!
```

### 문제 2: 헤더 중복

```
매 요청마다 동일한 헤더 전송

요청 1:
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 Chrome...
Accept: text/html
Cookie: session=abc123...
[300 bytes]

요청 2:
GET /style.css HTTP/1.1
Host: example.com                    ← 중복
User-Agent: Mozilla/5.0 Chrome...    ← 중복
Accept: text/css
Cookie: session=abc123...            ← 중복
[300 bytes]

→ 불필요한 대역폭 낭비
```

### 문제 3: 우선순위 없음

```
중요한 리소스와 덜 중요한 리소스를 구분 못함

브라우저가 원하는 순서:
1. index.html (높은 우선순위)
2. style.css (높은 우선순위)
3. script.js (높은 우선순위)
4. image1.jpg (낮은 우선순위)

HTTP/1.1: 순서대로만 처리
→ 이미지가 CSS보다 먼저 올 수 있음!
```

### 해결 시도: Domain Sharding

```
문제 우회:
브라우저가 도메인당 6개 동시 연결 제한
→ 여러 도메인 사용

example.com
static1.example.com
static2.example.com
static3.example.com

단점:
- DNS 조회 증가
- TCP 연결 증가
- 복잡도 증가
→ 근본적 해결 아님
```

---

## 10.3 HTTP/2 등장 ⭐

### 핵심 개선 사항

```
1. 멀티플렉싱 (Multiplexing)
2. 헤더 압축 (HPACK)
3. 서버 푸시 (Server Push)
4. 스트림 우선순위 (Stream Prioritization)
5. 바이너리 프로토콜
```

---

## 10.4 멀티플렉싱 ⭐ 가장 중요!

### 개념

```
하나의 TCP 연결에서 여러 요청/응답 동시 처리

비유: 8차선 고속도로

HTTP/1.1 (편도 1차선):
┌─────────┐         ┌─────────┐
│ 브라우저 │         │  서버   │
└────┬────┘         └────┬────┘
     │  1 ────────────→  │
     │  ←──────────── 1  │
     │  2 ────────────→  │
     │  ←──────────── 2  │

HTTP/2 (8차선):
┌─────────┐         ┌─────────┐
│ 브라우저 │         │  서버   │
└────┬────┘         └────┬────┘
     │  1 ────────────→  │
     │  2 ────────────→  │
     │  3 ────────────→  │ 동시 전송!
     │  ←──────────── 1  │
     │  ←──────────── 3  │ 순서 상관없이
     │  ←──────────── 2  │ 응답 가능
```

### 스트림 (Stream)

```
각 요청/응답 쌍 = 1개 스트림

┌──────────────────────────────────┐
│   하나의 TCP 연결                 │
│                                  │
│  ┌────────┐  Stream 1 (HTML)    │
│  ├────────┤  Stream 3 (CSS)     │
│  ├────────┤  Stream 5 (JS)      │
│  ├────────┤  Stream 7 (Image)   │
│  └────────┘                      │
│                                  │
│  여러 스트림이 프레임 단위로      │
│  인터리브(interleave)됨          │
└──────────────────────────────────┘

Stream ID:
- 클라이언트가 시작: 홀수 (1, 3, 5...)
- 서버가 시작: 짝수 (2, 4, 6...)
```

---

## 10.5 헤더 압축 (HPACK)

### 정적 테이블

```
자주 사용하는 헤더는 인덱스로 표현

정적 테이블:
┌────────┬─────────────────────────┐
│ Index  │  Header                 │
├────────┼─────────────────────────┤
│   1    │ :authority              │
│   2    │ :method GET             │
│   3    │ :method POST            │
│   4    │ :path /                 │
│  15    │ accept-encoding         │
└────────┴─────────────────────────┘

예:
"GET" → 인덱스 2
"POST" → 인덱스 3

HTTP/1.1: "GET" (3 bytes)
HTTP/2: 2 (1 byte) → 66% 절약!
```

### 동적 테이블

```
이전 요청의 헤더를 기억

첫 요청:
Cookie: session=abc123...def  (100 bytes)
→ 동적 테이블 인덱스 62에 저장

두 번째 요청:
Cookie: 인덱스 62 (1 byte)

→ 99% 절약!
```

### 압축 효과

```
HTTP/1.1 헤더:
요청 1: 300 bytes
요청 2: 300 bytes
요청 3: 300 bytes
합계: 900 bytes

HTTP/2 헤더 (HPACK):
요청 1: 300 bytes (최초)
요청 2: 50 bytes (압축)
요청 3: 50 bytes (압축)
합계: 400 bytes

→ 55% 절약!
```

---

## 10.6 서버 푸시 (Server Push)

### 개념

```
클라이언트가 요청하기 전에 서버가 먼저 전송

전통적 방식 (HTTP/1.1):
┌─────────┐                     ┌─────────┐
│ 브라우저 │                     │  서버   │
└────┬────┘                     └────┬────┘
     │ 1. index.html 요청          │
     │ ──────────────────────────→ │
     │                             │
     │ 2. index.html 응답          │
     │ ←────────────────────────── │
     │ (파싱 후 style.css 발견)    │
     │                             │
     │ 3. style.css 요청           │
     │ ──────────────────────────→ │
     │                             │
     │ 4. style.css 응답           │
     │ ←────────────────────────── │

HTTP/2 서버 푸시:
┌─────────┐                     ┌─────────┐
│ 브라우저 │                     │  서버   │
└────┬────┘                     └────┬────┘
     │ 1. index.html 요청          │
     │ ──────────────────────────→ │
     │                             │
     │ 2. index.html 응답          │
     │ ←────────────────────────── │
     │                             │
     │ 3. style.css 푸시 (자동!)   │
     │ ←────────────────────────── │
     │ (요청 안 했는데 보내줌)     │

→ 1 RTT 절약!
```

### NGINX 설정

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    location / {
        root /var/www/html;

        # 서버 푸시 설정
        http2_push /style.css;
        http2_push /script.js;
        http2_push /logo.png;
    }
}
```

### Node.js 설정

```javascript
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt')
});

server.on('stream', (stream, headers) => {
  const path = headers[':path'];

  if (path === '/') {
    // index.html 전송
    stream.respond({
      'content-type': 'text/html',
      ':status': 200
    });
    stream.end('<html><link rel="stylesheet" href="/style.css"></html>');

    // 서버 푸시: style.css
    stream.pushStream({ ':path': '/style.css' }, (err, pushStream) => {
      if (err) throw err;

      pushStream.respond({
        'content-type': 'text/css',
        ':status': 200
      });
      pushStream.end('body { color: red; }');
    });
  }
});

server.listen(443);
```

---

## 10.7 스트림 우선순위

### 우선순위 트리

```
중요한 리소스를 먼저 전송

        Root
          │
    ┌─────┴─────┐
    │           │
  HTML       CSS (weight: 200)
(weight: 256)  │
               ├── JS (weight: 32)
               └── Image (weight: 8)

Weight:
- 높을수록 우선순위 높음
- HTML > CSS > JS > Image

브라우저가 자동으로 설정하지만,
서버에서 조정 가능
```

---

## 10.8 HTTP/2 vs HTTP/1.1 비교

```
┌──────────────────┬─────────────┬─────────────┐
│      항목         │  HTTP/1.1   │  HTTP/2     │
├──────────────────┼─────────────┼─────────────┤
│  프로토콜         │  텍스트      │  바이너리    │
├──────────────────┼─────────────┼─────────────┤
│  연결             │  요청마다    │  하나로 재사용│
├──────────────────┼─────────────┼─────────────┤
│  멀티플렉싱       │  ❌         │  ✅         │
├──────────────────┼─────────────┼─────────────┤
│  헤더 압축        │  ❌         │  ✅ HPACK   │
├──────────────────┼─────────────┼─────────────┤
│  서버 푸시        │  ❌         │  ✅         │
├──────────────────┼─────────────┼─────────────┤
│  우선순위         │  ❌         │  ✅         │
├──────────────────┼─────────────┼─────────────┤
│  HOL Blocking    │  ✅ 있음    │  ❌ 해결    │
├──────────────────┼─────────────┼─────────────┤
│  HTTPS 필수       │  ❌         │  ✅ (사실상) │
└──────────────────┴─────────────┴─────────────┘
```

---

## 10.9 HTTP/2 적용하기

### NGINX 설정

```nginx
server {
    # HTTP/2 활성화
    listen 443 ssl http2;

    server_name example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # HTTP/2 설정 (선택)
    http2_max_field_size 16k;
    http2_max_header_size 32k;

    location / {
        root /var/www/html;
    }
}
```

### 확인 방법

```bash
# curl로 확인
curl -I --http2 https://example.com

# 결과에서 확인:
# HTTP/2 200

# Chrome DevTools
# Network 탭 → Protocol 열
# h2 = HTTP/2
# http/1.1 = HTTP/1.1
```

```javascript
// JavaScript로 확인
fetch('https://example.com')
  .then(response => {
    console.log(response.headers.get('x-firefox-spdy'));
    // h2 = HTTP/2
  });
```

---

## 10.10 HTTP/3 등장 ⭐

### HTTP/2의 남은 문제

```
HTTP/2는 여전히 TCP 기반
→ TCP의 HOL Blocking 문제

┌──────────────────────────────────┐
│  TCP 연결                         │
│                                  │
│  패킷 1 ✅                       │
│  패킷 2 ❌ 손실!                 │
│  패킷 3 ⏸️  대기...              │
│  패킷 4 ⏸️  대기...              │
│                                  │
│  패킷 2 재전송될 때까지           │
│  패킷 3, 4 모두 대기             │
│  (멀티플렉싱이 무의미)            │
└──────────────────────────────────┘
```

### QUIC 프로토콜

```
HTTP/3 = HTTP/2 over QUIC

QUIC:
- UDP 기반 ⭐
- 독립적 스트림
- 내장 암호화 (TLS 1.3)
- 빠른 핸드셰이크

프로토콜 스택 비교:

HTTP/1.1, HTTP/2:          HTTP/3:
┌────────────┐            ┌────────────┐
│   HTTP     │            │   HTTP     │
├────────────┤            ├────────────┤
│   TLS      │            │   QUIC     │ ← TLS 1.3 내장
├────────────┤            ├────────────┤
│   TCP      │            │   UDP      │ ← UDP!
├────────────┤            ├────────────┤
│   IP       │            │   IP       │
└────────────┘            └────────────┘
```

---

## 10.11 HTTP/3의 장점

### 1. 스트림별 독립성

```
TCP (HTTP/2):
패킷 2 손실 → 모든 스트림 대기

QUIC (HTTP/3):
패킷 2 손실 → Stream 1만 대기
             Stream 2, 3, 4 계속 진행 ✅

┌──────────────────────────────────┐
│  QUIC 연결                        │
│                                  │
│  Stream 1: ❌ 패킷 손실 (대기)   │
│  Stream 2: ✅ 정상 진행          │
│  Stream 3: ✅ 정상 진행          │
│  Stream 4: ✅ 정상 진행          │
│                                  │
│  → 독립적으로 동작!               │
└──────────────────────────────────┘
```

### 2. 0-RTT 연결 재개

```
HTTP/2 (TLS 1.2):
1-RTT 핸드셰이크 필요

HTTP/3 (QUIC):
첫 연결: 1-RTT
재연결: 0-RTT ⚡

┌──────────┐              ┌──────────┐
│ 클라이언트│              │   서버   │
└─────┬────┘              └─────┬────┘
      │                        │
      │ 1. 저장된 키 + 데이터   │
      │ ──────────────────────→│
      │                        │
      │ 2. 응답 (즉시!)        │
      │ ←──────────────────────│

→ 핸드셰이크 생략!
```

### 3. Connection Migration

```
네트워크 변경 시에도 연결 유지

시나리오:
1. WiFi로 영상 시청 중
2. WiFi → 4G로 전환
3. IP 주소 변경됨

TCP (HTTP/2):
→ 연결 끊김, 재연결 필요

QUIC (HTTP/3):
→ Connection ID로 식별
→ 끊김 없이 계속 진행 ✅

모바일 환경에 최적!
```

---

## 10.12 HTTP/3 적용하기

### NGINX (with QUIC support)

```nginx
# nginx-quic 빌드 필요
server {
    listen 443 ssl http3 reuseport;
    listen 443 ssl http2;

    server_name example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # HTTP/3 광고
    add_header Alt-Svc 'h3=":443"; ma=86400';

    location / {
        root /var/www/html;
    }
}
```

### Cloudflare 사용

```
Cloudflare를 통하면 자동으로 HTTP/3 지원

설정:
1. Cloudflare에 도메인 연결
2. SSL/TLS → Full
3. Network → HTTP/3 활성화

→ 끝! 자동으로 HTTP/3 제공
```

### 확인

```bash
# curl로 확인 (curl 7.66+)
curl -I --http3 https://example.com

# Chrome에서 확인
chrome://flags/#enable-quic

# DevTools
# Protocol: h3 = HTTP/3
```

---

## 10.13 버전별 성능 비교

### 테스트 환경

```
페이지:
- HTML: 1개
- CSS: 3개
- JS: 5개
- 이미지: 20개
합계: 29개 리소스

네트워크:
- Latency: 100ms
- Bandwidth: 10Mbps
```

### 로딩 시간 비교

```
HTTP/1.1 (6 parallel connections):
┌────────────────────────────────┐
│ ████████████████████████████   │ 5.2초
└────────────────────────────────┘

HTTP/2:
┌───────────────────┐
│ █████████████     │ 2.8초 (46% 개선)
└───────────────────┘

HTTP/3:
┌──────────────┐
│ ██████████   │ 2.1초 (60% 개선)
└──────────────┘

* 패킷 손실 5% 환경에서는:
HTTP/2: 4.5초
HTTP/3: 2.3초 (49% 빠름!)
```

---

## 10.14 실전 최적화 팁

### HTTP/2 최적화

```
1. Domain Sharding 제거
❌ Before (HTTP/1.1):
static1.example.com
static2.example.com

✅ After (HTTP/2):
example.com (하나로 통합)

2. 파일 합치기 불필요
❌ Before:
bundle.js (모든 JS 합침)

✅ After:
module1.js, module2.js (분리)
→ 캐싱 효율 증가

3. 이미지 스프라이트 불필요
❌ Before:
sprite.png (모든 아이콘 합침)

✅ After:
icon1.svg, icon2.svg (분리)
→ 필요한 것만 로드
```

### 서버 푸시 주의사항

```
⚠️ 주의:
서버 푸시는 신중하게 사용

문제:
- 클라이언트가 이미 캐시 보유
- 불필요한 푸시 → 대역폭 낭비

해결:
1. Cache-Digest 사용
2. 주요 리소스만 푸시
3. 쿠키로 푸시 여부 판단

예:
if (firstVisit) {
  http2_push /critical.css;
}
```

---

## 10.15 브라우저 지원

### HTTP/2

```
✅ 모든 최신 브라우저 지원
- Chrome 49+
- Firefox 52+
- Safari 10+
- Edge 12+

→ 안심하고 사용 가능 (2024년 기준 97% 지원)
```

### HTTP/3

```
⚠️ 지원 증가 중
- Chrome 87+ ✅
- Firefox 88+ ✅
- Safari 14+ ✅
- Edge 87+ ✅

→ 사용 가능하지만 Fallback 필요 (2024년 기준 75% 지원)

설정:
listen 443 ssl http3;
listen 443 ssl http2;  ← Fallback

Alt-Svc 헤더로 HTTP/3 광고:
Alt-Svc: h3=":443"; ma=86400
```

---

## 핵심 요약

### HTTP/2
```
멀티플렉싱: 하나의 연결로 여러 요청
헤더 압축: HPACK (50-90% 절약)
서버 푸시: 미리 전송
바이너리: 효율적 파싱
HTTPS 필수
```

### HTTP/3
```
QUIC: UDP 기반
0-RTT: 초고속 재연결
독립 스트림: 패킷 손실에 강함
Connection Migration: 네트워크 전환 OK
모바일 최적
```

### 선택 가이드
```
현재 (2024):
- HTTP/2: 필수 적용 ✅
- HTTP/3: 선택 적용 (Cloudflare 권장)

미래:
- HTTP/3가 표준이 될 것
```

---

## 다음 단계

HTTP 프로토콜을 마스터했으니, 이제 **RESTful API 설계**를 배워봅시다!

👉 [Chapter 11. RESTful API 설계](11-RESTful-API.md)

---

## 실습 과제

### 과제 1: HTTP/2 활성화 및 확인

```bash
# 1. NGINX HTTP/2 설정
sudo nano /etc/nginx/sites-available/default

# listen 443 ssl http2; 추가

# 2. 재시작
sudo systemctl restart nginx

# 3. 확인
curl -I --http2 https://yourdomain.com
```

### 과제 2: 성능 비교

```
1. Chrome DevTools 열기
2. Network 탭
3. Protocol 열 추가
4. 페이지 로드
5. h2 vs http/1.1 확인
6. 로딩 시간 비교
```

### 과제 3: 서버 푸시 실습

```nginx
server {
    listen 443 ssl http2;

    location / {
        root /var/www/html;

        # 주요 CSS/JS 푸시
        http2_push /static/critical.css;
        http2_push /static/app.js;
    }
}
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 2-3시간
**대상**: 프론트엔드/백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!