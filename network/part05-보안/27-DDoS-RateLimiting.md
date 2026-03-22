# Chapter 27. DDoS와 Rate Limiting

## 27.1 DDoS 공격이란?

### 정의

```
DDoS (Distributed Denial of Service):
분산 서비스 거부 공격

비유: 식당 마비
┌────────────────────────────────┐
│ 정상 고객 10명 → 식당 정상 운영│
│ 가짜 고객 1000명 → 식당 마비   │
│ 진짜 고객 서비스 불가           │
└────────────────────────────────┘
```

### 공격 유형

```
1. Volumetric Attack (용량 공격)
   대량 트래픽으로 대역폭 소진
   - UDP Flood
   - ICMP Flood

2. Protocol Attack (프로토콜 공격)
   프로토콜 약점 악용
   - SYN Flood
   - Ping of Death

3. Application Layer Attack (L7 공격)
   애플리케이션 리소스 고갈
   - HTTP Flood
   - Slowloris
```

---

## 27.2 SYN Flood

### 공격 원리

```
TCP 3-way handshake 악용

정상 연결:
Client: SYN →
Server: ← SYN-ACK
Client: ACK →
[연결 성립]

SYN Flood:
Attacker: SYN → (가짜 IP)
Server: ← SYN-ACK (응답 안 옴)
Server: 대기... (리소스 소모)

→ 수천 개 연결 대기 → 서버 마비
```

### 방어

```bash
# SYN Cookie 활성화
sysctl -w net.ipv4.tcp_syncookies=1

# 백로그 큐 크기 증가
sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# 타임아웃 단축
sysctl -w net.ipv4.tcp_synack_retries=2
```

---

## 27.3 HTTP Flood

### 공격 원리

```
대량의 정상적인 HTTP 요청

초당 10,000 요청:
GET / HTTP/1.1
GET / HTTP/1.1
GET / HTTP/1.1
...

→ 서버 CPU/메모리 고갈
→ 정상 사용자 접근 불가
```

### 방어: Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

// 기본 Rate Limiter
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15분
  max: 100,  // 최대 100 요청
  message: '너무 많은 요청입니다. 나중에 다시 시도하세요.',
  standardHeaders: true,
  legacyHeaders: false,
});

app.use(limiter);

// API별 다른 제한
const apiLimiter = rateLimit({
  windowMs: 1 * 60 * 1000,
  max: 10
});

app.use('/api/', apiLimiter);

// 로그인 엄격하게
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true  // 성공 시 카운트 안 함
});

app.post('/api/auth/login', loginLimiter, async (req, res) => {
  // 로그인 로직
});
```

---

## 27.4 고급 Rate Limiting

### Redis 기반

```javascript
const RedisStore = require('rate-limit-redis');
const { createClient } = require('redis');

const redisClient = createClient({
  host: 'localhost',
  port: 6379
});

redisClient.connect();

const limiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:'
  }),
  windowMs: 15 * 60 * 1000,
  max: 100
});

// 다중 서버 환경에서도 동작
```

### IP별 + 사용자별 제한

```javascript
// IP별 제한
const ipLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  keyGenerator: (req) => req.ip
});

// 사용자별 제한 (로그인 후)
const userLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 1000,  // 인증 사용자는 더 많이
  keyGenerator: (req) => req.user?.id || req.ip
});

app.use('/api', ipLimiter, userLimiter);
```

### 동적 제한

```javascript
const dynamicLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: (req) => {
    // 관리자는 제한 없음
    if (req.user?.role === 'admin') {
      return 0;  // 무제한
    }

    // 유료 회원은 더 많이
    if (req.user?.plan === 'premium') {
      return 1000;
    }

    // 일반 사용자
    return 100;
  }
});
```

---

## 27.5 Slowloris 공격

### 공격 원리

```
느린 HTTP 요청으로 연결 점유

정상 요청:
GET / HTTP/1.1\r\n
Host: example.com\r\n
\r\n

Slowloris:
GET / HTTP/1.1\r\n
Host: example.com\r\n
X-a: b\r\n
(10초 대기)
X-c: d\r\n
(10초 대기)
...

→ 헤더를 천천히 전송
→ 연결 계속 점유
→ 서버 연결 풀 고갈
```

### 방어

```javascript
// 타임아웃 설정
const server = app.listen(3000);

server.setTimeout(30000);  // 30초
server.headersTimeout = 40000;  // 헤더 40초
server.requestTimeout = 0;

// 연결 수 제한
server.maxRequestsPerSocket = 0;
server.maxConnections = 1000;
```

---

## 27.6 AWS Shield & WAF

### AWS Shield

```
자동 DDoS 방어

Shield Standard (무료):
- L3/L4 공격 방어
- SYN Flood, UDP Flood 등
- 자동 활성화

Shield Advanced (유료, $3000/월):
- L7 공격 방어
- 24/7 DDoS 대응팀
- 비용 보호
```

### AWS WAF Rate Limiting

```hcl
resource "aws_wafv2_web_acl" "main" {
  name  = "rate-limit-acl"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # IP별 Rate Limit
  rule {
    name     = "rate-limit-per-ip"
    priority = 1

    statement {
      rate_based_statement {
        limit              = 2000  # 5분당 2000 요청
        aggregate_key_type = "IP"
      }
    }

    action {
      block {
        custom_response {
          response_code = 429
          custom_response_body_key = "too_many_requests"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "WebACL"
    sampled_requests_enabled   = true
  }
}

# Custom Response Body
resource "aws_wafv2_web_acl_association" "main" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## 27.7 Cloudflare DDoS 방어

### Cloudflare 기능

```
무료 플랜:
✅ DDoS 방어 (Unlimited)
✅ WAF
✅ Rate Limiting (유료)
✅ Bot 감지

설정:
1. Cloudflare에 도메인 추가
2. 네임서버 변경
3. Security → DDoS → High

→ 자동으로 공격 차단
```

### Rate Limiting (유료)

```
Cloudflare Dashboard:
Security → WAF → Rate limiting rules

규칙:
- URL: /api/*
- Method: POST
- Rate: 10 요청/분
- Action: Block (1시간)
```

---

## 27.8 nginx Rate Limiting

```nginx
# nginx.conf

# Zone 정의 (IP별 10MB 메모리)
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;

# 연결 수 제한
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

server {
    listen 80;
    server_name example.com;

    # API Rate Limiting
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        # 초당 10개, 버스트 20개
        # nodelay: 큐 없이 즉시 처리

        proxy_pass http://backend;
    }

    # 로그인 Rate Limiting
    location /api/auth/login {
        limit_req zone=login_limit burst=5 nodelay;
        # 분당 5개

        proxy_pass http://backend;
    }

    # 연결 수 제한
    location / {
        limit_conn conn_limit 10;  # IP당 10개 연결
        proxy_pass http://backend;
    }

    # 429 에러 커스터마이징
    error_page 429 = @rate_limit;

    location @rate_limit {
        return 429 '{"error": "Too Many Requests"}';
        add_header Content-Type application/json always;
    }
}
```

---

## 27.9 모니터링 및 알람

### CloudWatch 메트릭

```hcl
# DDoS 공격 감지 알람
resource "aws_cloudwatch_metric_alarm" "ddos_detected" {
  alarm_name          = "ddos-attack-detected"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DDoSDetected"
  namespace           = "AWS/DDoSProtection"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "DDoS 공격 감지됨"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}

# WAF 차단 급증 알람
resource "aws_cloudwatch_metric_alarm" "waf_blocked" {
  alarm_name          = "waf-blocked-requests-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "BlockedRequests"
  namespace           = "AWS/WAFV2"
  period              = 300
  statistic           = "Sum"
  threshold           = 1000
  alarm_description   = "WAF 차단 요청 급증"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

### 로깅

```javascript
// Rate Limit 초과 로그
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  handler: (req, res) => {
    console.warn('Rate limit exceeded:', {
      ip: req.ip,
      path: req.path,
      userAgent: req.headers['user-agent']
    });

    res.status(429).json({
      error: '너무 많은 요청입니다'
    });
  }
});
```

---

## 27.10 실전 대응 전략

### 계층적 방어

```
1단계: CDN/프록시 (Cloudflare)
       → DDoS 트래픽 흡수

2단계: WAF (AWS WAF)
       → 악성 패턴 차단

3단계: Load Balancer
       → 트래픽 분산

4단계: 웹 서버 (nginx)
       → Rate Limiting

5단계: 애플리케이션
       → 로직 최적화

6단계: 데이터베이스
       → 쿼리 최적화, 캐싱
```

### 공격 시 대응

```
1. 감지
   - 갑작스러운 트래픽 증가
   - 응답 시간 급증
   - 에러율 증가

2. 분석
   - 공격 유형 파악
   - 공격 소스 IP 분석
   - 패턴 분석

3. 차단
   - IP 블랙리스트
   - Rate Limiting 강화
   - Geo-blocking (필요 시)

4. 복구
   - 캐시 퍼지
   - 서버 재시작 (필요 시)
   - 정상화 확인

5. 사후 분석
   - 로그 분석
   - 대응 개선
```

---

## 핵심 요약

### DDoS 공격
```
Volumetric: 대역폭 소진
Protocol: SYN Flood
Application: HTTP Flood
```

### 방어
```
Rate Limiting: 요청 제한
CDN: 트래픽 분산
WAF: 악성 패턴 차단
Shield: AWS DDoS 방어
```

### Rate Limiting
```
express-rate-limit: 앱 레벨
nginx: 웹서버 레벨
WAF: 네트워크 레벨
Redis: 다중 서버
```

### 모니터링
```
CloudWatch Alarm
로깅 및 분석
즉각 대응 체계
```

---

## 다음 단계

Part 5 (보안) 완료! 이제 **웹 성능 최적화** 파트로 이동합니다!

👉 [Chapter 28. 브라우저 렌더링 과정](../part06-웹성능/28-브라우저-렌더링.md)

---

## 실습 과제

### 과제 1: Rate Limiting 구현

```javascript
// 1. express-rate-limit 설치
// 2. 전역 limiter (100 req/15min)
// 3. API limiter (10 req/min)
// 4. 로그인 limiter (5 req/15min)
// 5. Redis store 적용
```

### 과제 2: nginx Rate Limiting

```nginx
// 1. limit_req_zone 설정
// 2. API 경로별 제한
// 3. 429 에러 커스터마이징
// 4. 테스트 (ab, wrk)
```

### 과제 3: AWS WAF Rate Rule

```hcl
// 1. WAF Web ACL 생성
// 2. Rate Limiting Rule 추가
// 3. ALB 연결
// 4. CloudWatch 메트릭 확인
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3시간
**대상**: 백엔드 개발자, DevOps
**중요도**: ⭐⭐⭐ 매우 중요!