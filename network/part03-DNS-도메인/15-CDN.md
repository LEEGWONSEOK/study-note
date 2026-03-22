# Chapter 15. CDN과 DNS

## 15.1 CDN이란?

### 정의

**CDN** (Content Delivery Network)
전 세계에 분산된 서버 네트워크를 통해 콘텐츠를 빠르게 전달

### 비유: 편의점 vs 물류창고

```
물류창고 모델 (CDN 없음):
┌──────────┐                      ┌──────────┐
│ 서울 고객 │ ──── 5000km ────→  │ 미국 서버 │
└──────────┘      느림!           └──────────┘

편의점 모델 (CDN):
┌──────────┐      ┌──────────┐
│ 서울 고객 │ ──→ │ 서울 CDN │  ← 빠름!
└──────────┘      └──────────┘
                       ↓
                  (원본 서버는
                   미국에 있음)
```

### CDN 동작 원리

```
┌─────────────────────────────────────────┐
│  1. 최초 요청 (Cache Miss)               │
│                                         │
│  사용자 → Edge Server (캐시 없음)       │
│         ← Origin Server                 │
│         → 사용자에게 전달 + 캐시 저장   │
│                                         │
│  2. 이후 요청 (Cache Hit)                │
│                                         │
│  사용자 → Edge Server (캐시 있음) ⚡     │
│         → 즉시 전달 (오리진 서버 안 감) │
└─────────────────────────────────────────┘
```

---

## 15.2 CDN의 장점

### 1. 속도 향상 ⚡

```
Without CDN:
사용자 (서울) → 서버 (미국)
RTT: 200ms
TTFB: 500ms

With CDN:
사용자 (서울) → CDN (서울)
RTT: 10ms
TTFB: 50ms

→ 10배 빠름!
```

### 2. 서버 부하 감소

```
Before CDN:
모든 요청 → Origin Server
┌──────┐
│Origin│ ← ← ← ← ← (100 requests)
└──────┘
부하: 100%

After CDN:
95% → Edge Server (캐시)
 5% → Origin Server (신규/변경)
┌──────┐
│Origin│ ← (5 requests)
└──────┘
부하: 5%

→ 95% 부하 감소!
```

### 3. 대역폭 비용 절감

```
이미지 1개: 1MB
요청: 10,000회/일

Without CDN:
10,000 × 1MB = 10GB/일
→ Origin에서 모두 전송
→ AWS 송신 비용: $0.09/GB
→ 월 비용: 10 × 30 × 0.09 = $27

With CDN:
95% Cache Hit
500 × 1MB = 500MB/일 (Origin)
→ 월 비용: 0.5 × 30 × 0.09 = $1.35

→ 95% 절감!
```

### 4. 가용성 향상

```
Origin 서버 다운 시:
CDN이 캐시된 콘텐츠 계속 제공
→ 일부 기능은 작동

DDoS 공격:
CDN이 트래픽 흡수
→ Origin 서버 보호
```

---

## 15.3 CDN 제공 업체

### 주요 CDN 서비스

```
┌────────────────┬─────────────┬──────────────┐
│   서비스        │  특징        │  가격         │
├────────────────┼─────────────┼──────────────┤
│ CloudFront     │ AWS 통합    │ $0.085/GB    │
│ (AWS)          │             │              │
├────────────────┼─────────────┼──────────────┤
│ Cloud CDN      │ GCP 통합    │ $0.08/GB     │
│ (Google)       │             │              │
├────────────────┼─────────────┼──────────────┤
│ Cloudflare     │ 무료 플랜   │ 무료/$20/월  │
│                │ DDoS 보호   │              │
├────────────────┼─────────────┼──────────────┤
│ Fastly         │ 실시간 퍼지 │ $0.12/GB     │
├────────────────┼─────────────┼──────────────┤
│ Akamai         │ 엔터프라이즈│ 협의         │
└────────────────┴─────────────┴──────────────┘
```

---

## 15.4 CloudFront 설정 (AWS)

### Terraform 예시

```hcl
# S3 버킷 (Origin)
resource "aws_s3_bucket" "static" {
  bucket = "myproject-static"
}

resource "aws_s3_bucket_public_access_block" "static" {
  bucket = aws_s3_bucket.static.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# CloudFront OAC (Origin Access Control)
resource "aws_cloudfront_origin_access_control" "static" {
  name                              = "myproject-static-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "static" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Static assets CDN"
  default_root_object = "index.html"
  aliases             = ["cdn.myproject.com"]

  # Origin (S3)
  origin {
    domain_name              = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id                = "S3-myproject-static"
    origin_access_control_id = aws_cloudfront_origin_access_control.static.id
  }

  # Cache Behavior
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-myproject-static"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    compress               = true
  }

  # Cache Behavior for images
  ordered_cache_behavior {
    path_pattern     = "/images/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-myproject-static"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 86400   # 1일
    max_ttl                = 31536000 # 1년
    compress               = true
  }

  # SSL Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cert.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # Restrictions
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Name = "myproject-cdn"
  }
}

# S3 Bucket Policy (CloudFront 접근 허용)
resource "aws_s3_bucket_policy" "static" {
  bucket = aws_s3_bucket.static.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudFrontOAC"
        Effect = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.static.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.static.arn
          }
        }
      }
    ]
  })
}

# Route 53 레코드 (CDN 연결)
resource "aws_route53_record" "cdn" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "cdn.myproject.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.static.domain_name
    zone_id                = aws_cloudfront_distribution.static.hosted_zone_id
    evaluate_target_health = false
  }
}
```

---

## 15.5 Cloudflare 설정 (무료)

### Cloudflare 연결

```
1. Cloudflare 가입
   https://dash.cloudflare.com/sign-up

2. 사이트 추가
   - myproject.com 입력

3. DNS 레코드 스캔
   - 기존 레코드 자동 임포트

4. 네임서버 변경
   - alice.ns.cloudflare.com
   - bob.ns.cloudflare.com
   → 도메인 등록업체에서 변경

5. SSL/TLS 설정
   - Full (strict) 선택
   - Universal SSL 자동 발급

6. 캐싱 설정
   - Cache Everything
   - Browser Cache TTL: 1 month
```

### Cloudflare Page Rules

```
무료 플랜: 3개 규칙

규칙 1: 정적 파일 캐싱
URL: *.myproject.com/static/*
Settings:
- Cache Level: Cache Everything
- Edge Cache TTL: 1 month
- Browser Cache TTL: 1 month

규칙 2: API 캐싱 방지
URL: api.myproject.com/*
Settings:
- Cache Level: Bypass

규칙 3: www 리다이렉트
URL: www.myproject.com/*
Settings:
- Forwarding URL: 301 Permanent Redirect
- Destination: https://myproject.com/$1
```

---

## 15.6 캐싱 전략

### Cache-Control 헤더

```
정적 리소스 (변경 거의 없음):
Cache-Control: public, max-age=31536000, immutable

예: /static/main.abc123.js (해시 포함)
→ 1년 캐싱, 변경 시 파일명 변경

동적 콘텐츠 (자주 변경):
Cache-Control: public, max-age=300, s-maxage=600

max-age=300: 브라우저 5분 캐시
s-maxage=600: CDN 10분 캐시

캐싱 방지:
Cache-Control: no-cache, no-store, must-revalidate
```

### NGINX 설정

```nginx
server {
    listen 80;
    server_name myproject.com;

    # 정적 파일 캐싱 (1년)
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # HTML 파일 (짧은 캐시)
    location ~* \.html$ {
        expires 5m;
        add_header Cache-Control "public, max-age=300";
    }

    # API (캐시 안 함)
    location /api/ {
        proxy_pass http://backend;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }
}
```

---

## 15.7 캐시 무효화 (Cache Invalidation)

### CloudFront Invalidation

```bash
# AWS CLI
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"

# 특정 파일만
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/css/style.css" "/js/app.js"

# 비용: 1000개 경로/월 무료, 이후 $0.005/경로
```

```hcl
# Terraform (배포 시 자동 무효화)
resource "null_resource" "invalidate_cache" {
  triggers = {
    build = timestamp()
  }

  provisioner "local-exec" {
    command = <<EOF
      aws cloudfront create-invalidation \
        --distribution-id ${aws_cloudfront_distribution.static.id} \
        --paths "/*"
    EOF
  }

  depends_on = [aws_s3_object.files]
}
```

### Cloudflare Purge

```bash
# Cloudflare API
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  --data '{"purge_everything":true}'

# 특정 파일만
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  --data '{"files":["https://myproject.com/style.css"]}'
```

---

## 15.8 버전 관리 전략

### 파일명 해시 (권장)

```
Before:
/static/style.css
/static/app.js

After:
/static/style.abc123.css
/static/app.abc123.js

장점:
- 캐시 무효화 불필요
- 파일명이 바뀌면 자동으로 신규 다운로드
- 롤백 쉬움

Webpack 설정:
module.exports = {
  output: {
    filename: '[name].[contenthash].js',
    path: path.resolve(__dirname, 'dist')
  }
};
```

### 쿼리 파라미터 버전

```
/static/style.css?v=1.2.3

문제:
- 일부 CDN에서 쿼리 파라미터 무시
- 캐싱 효율 낮음

→ 파일명 해시 권장
```

---

## 15.9 실전 시나리오

### 시나리오 1: React SPA + API

```
구조:
- React App (정적 파일) → CloudFront
- API 서버 → ALB

1. CloudFront Distribution
   - Origin: S3 (React 빌드 파일)
   - Domain: myproject.com

2. DNS 설정
   myproject.com     → CloudFront (ALIAS)
   api.myproject.com → ALB (ALIAS)

3. 캐싱
   HTML: 5분
   JS/CSS: 1년 (해시)
   이미지: 1년

4. 배포 프로세스
   npm run build
   → aws s3 sync dist/ s3://myproject-static/
   → CloudFront invalidation
```

### 시나리오 2: 이미지 최적화 CDN

```
1. CloudFront + Lambda@Edge
   - 자동 이미지 리사이징
   - WebP 변환
   - 압축

2. URL 형식
   https://cdn.myproject.com/images/photo.jpg?w=800&q=85
   → Lambda@Edge가 처리

3. 원본 이미지
   S3: /images/photo.jpg (원본)
   → Lambda가 리사이징
   → CloudFront가 캐싱
```

### 시나리오 3: 멀티 CDN

```
Primary: CloudFront
Fallback: Cloudflare

DNS Failover:
1. CloudFront 정상 → CloudFront로 라우팅
2. CloudFront 장애 → Cloudflare로 자동 전환

Route 53 Failover 레코드:
myproject.com → Primary: CloudFront
             → Secondary: Cloudflare
```

---

## 15.10 성능 모니터링

### CloudWatch 메트릭

```
주요 지표:
- Requests: 요청 수
- BytesDownloaded: 전송량
- ErrorRate: 오류율
- OriginLatency: 오리진 지연시간

알람 설정:
- ErrorRate > 5%
- OriginLatency > 3000ms
```

### Cloudflare Analytics

```
무료 플랜 제공:
- 트래픽 분석
- 지역별 통계
- Cache Hit Rate
- 위협 차단 수
```

---

## 15.11 보안

### HTTPS 필수

```
CloudFront:
- Viewer Protocol Policy: Redirect HTTP to HTTPS
- Origin Protocol Policy: HTTPS Only

Cloudflare:
- SSL/TLS: Full (strict)
- Always Use HTTPS: ON
```

### Signed URLs (Private Content)

```javascript
// CloudFront Signed URL 생성
const AWS = require('aws-sdk');
const cloudfront = new AWS.CloudFront.Signer(
  keyPairId,
  privateKey
);

const signedUrl = cloudfront.getSignedUrl({
  url: 'https://cdn.myproject.com/private/video.mp4',
  expires: Math.floor(Date.now() / 1000) + 3600  // 1시간
});

// 이 URL은 1시간 후 만료
console.log(signedUrl);
```

---

## 핵심 요약

### CDN이란?
```
전 세계 분산 서버 네트워크
콘텐츠를 사용자와 가까운 곳에서 전달
→ 속도 향상, 부하 감소, 비용 절감
```

### 주요 CDN
```
CloudFront (AWS): $0.085/GB
Cloudflare: 무료 (무제한!)
Cloud CDN (GCP): $0.08/GB
```

### 캐싱 전략
```
정적 파일: 1년 (파일명 해시)
HTML: 5분
API: 캐싱 안 함
```

### 무효화
```
CloudFront: Invalidation (1000개/월 무료)
Cloudflare: Purge (무제한 무료)
파일명 해시: 무효화 불필요 ✅
```

---

## 다음 단계

Part 3 (DNS와 도메인) 완료! 이제 **클라우드 네트워킹** 파트로 이동합니다!

👉 [Chapter 18. 라우팅 테이블](../part04-클라우드네트워킹/18-라우팅테이블.md)

---

## 실습 과제

### 과제 1: CloudFront 설정

```
1. S3 버킷 생성 및 정적 파일 업로드
2. CloudFront Distribution 생성
3. 커스텀 도메인 연결
4. SSL 인증서 발급 (ACM)
5. 캐싱 동작 확인
```

### 과제 2: Cloudflare 무료 CDN

```
1. Cloudflare 가입
2. 도메인 추가
3. 네임서버 변경
4. SSL/TLS Full (strict) 설정
5. Page Rules 설정
6. Analytics 확인
```

### 과제 3: 캐싱 최적화

```
1. Cache-Control 헤더 설정
2. 정적 파일에 해시 추가
3. 캐시 Hit Rate 확인
4. Invalidation 테스트
5. 성능 측정 (Before/After)
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 3-4시간
**대상**: 프론트엔드/백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!