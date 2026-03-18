# Chapter 27: 모니터링과 로깅 (Monitoring and Logging)

## 학습 목표
- 구조화된 로깅 구현
- 중앙 집중식 로깅 시스템 (ELK Stack)
- APM (Application Performance Monitoring)
- 메트릭 수집과 대시보드 (Prometheus + Grafana)
- 알림 시스템 구축
- Spring Boot Actuator와 비교

## 1. 구조화된 로깅

### 1.1 기본 로깅 설정

```python
# app/core/logging_config.py
import logging
import sys
from pathlib import Path

def setup_logging():
    """로깅 설정"""
    # 로그 디렉토리 생성
    log_dir = Path("logs")
    log_dir.mkdir(exist_ok=True)

    # 로거 설정
    logger = logging.getLogger("app")
    logger.setLevel(logging.INFO)

    # 콘솔 핸들러
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    console_format = logging.Formatter(
        "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    )
    console_handler.setFormatter(console_format)

    # 파일 핸들러
    file_handler = logging.FileHandler("logs/app.log")
    file_handler.setLevel(logging.INFO)
    file_format = logging.Formatter(
        "%(asctime)s - %(name)s - %(levelname)s - %(funcName)s:%(lineno)d - %(message)s"
    )
    file_handler.setFormatter(file_format)

    # 에러 핸들러
    error_handler = logging.FileHandler("logs/error.log")
    error_handler.setLevel(logging.ERROR)
    error_handler.setFormatter(file_format)

    # 핸들러 등록
    logger.addHandler(console_handler)
    logger.addHandler(file_handler)
    logger.addHandler(error_handler)

    return logger

logger = setup_logging()

# 사용
logger.info("Application started")
logger.error("An error occurred", exc_info=True)
```

### 1.2 JSON 로깅 (구조화)

```bash
pip install python-json-logger
```

```python
# app/core/json_logging.py
from pythonjsonlogger import jsonlogger
import logging
import sys

def setup_json_logging():
    """JSON 형식 로깅"""
    logger = logging.getLogger("app")
    logger.setLevel(logging.INFO)

    # JSON 포맷터
    json_handler = logging.StreamHandler(sys.stdout)
    formatter = jsonlogger.JsonFormatter(
        "%(asctime)s %(name)s %(levelname)s %(funcName)s %(lineno)d %(message)s"
    )
    json_handler.setFormatter(formatter)

    logger.addHandler(json_handler)
    return logger

logger = setup_json_logging()

# 사용
logger.info("User logged in", extra={
    "user_id": 123,
    "username": "john",
    "ip_address": "192.168.1.1"
})

# 출력 (JSON):
# {
#   "asctime": "2024-01-15 10:30:00",
#   "name": "app",
#   "levelname": "INFO",
#   "funcName": "login",
#   "lineno": 42,
#   "message": "User logged in",
#   "user_id": 123,
#   "username": "john",
#   "ip_address": "192.168.1.1"
# }
```

### 1.3 요청 로깅 미들웨어

```python
# app/middleware/logging_middleware.py
from fastapi import Request
import logging
import time
import uuid

logger = logging.getLogger("app")

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    # 요청 ID 생성
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id

    # 시작 시간
    start_time = time.time()

    # 요청 로그
    logger.info("Request started", extra={
        "request_id": request_id,
        "method": request.method,
        "path": request.url.path,
        "query_params": dict(request.query_params),
        "client_ip": request.client.host,
        "user_agent": request.headers.get("user-agent", ""),
    })

    # 요청 처리
    try:
        response = await call_next(request)

        # 응답 로그
        duration = time.time() - start_time
        logger.info("Request completed", extra={
            "request_id": request_id,
            "status_code": response.status_code,
            "duration_ms": round(duration * 1000, 2),
        })

        # 응답 헤더에 request_id 추가
        response.headers["X-Request-ID"] = request_id

        return response

    except Exception as e:
        # 에러 로그
        duration = time.time() - start_time
        logger.error("Request failed", extra={
            "request_id": request_id,
            "error": str(e),
            "duration_ms": round(duration * 1000, 2),
        }, exc_info=True)
        raise
```

### 1.4 Spring Boot와 비교

**Spring Boot Logback 설정:**
```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.log</fileNamePattern>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

## 2. 중앙 집중식 로깅 (ELK Stack)

### 2.1 Elasticsearch + Logstash + Kibana

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5000:5000"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  elasticsearch-data:
```

**Logstash 설정 (logstash/pipeline/logstash.conf):**
```conf
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  # 타임스탬프 파싱
  date {
    match => ["timestamp", "ISO8601"]
  }

  # 필드 추가
  mutate {
    add_field => { "service" => "fastapi-app" }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "fastapi-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

### 2.2 FastAPI에서 Logstash로 전송

```bash
pip install python-logstash-async
```

```python
# app/core/logging_config.py
from logstash_async.handler import AsynchronousLogstashHandler
import logging

def setup_logstash_logging():
    """Logstash 로깅 설정"""
    logger = logging.getLogger("app")
    logger.setLevel(logging.INFO)

    # Logstash 핸들러
    logstash_handler = AsynchronousLogstashHandler(
        host='localhost',
        port=5000,
        database_path='logstash.db'
    )

    logger.addHandler(logstash_handler)
    return logger

logger = setup_logstash_logging()

# 사용
logger.info("User action", extra={
    "user_id": 123,
    "action": "login",
    "status": "success"
})
```

## 3. APM (Application Performance Monitoring)

### 3.1 Elastic APM

```bash
pip install elastic-apm
```

```python
# app/main.py
from fastapi import FastAPI
from elasticapm.contrib.starlette import make_apm_client, ElasticAPM

app = FastAPI()

# Elastic APM 설정
apm_config = {
    'SERVICE_NAME': 'fastapi-app',
    'SERVER_URL': 'http://localhost:8200',
    'ENVIRONMENT': 'production',
}

apm = make_apm_client(apm_config)
app.add_middleware(ElasticAPM, client=apm)

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # APM이 자동으로 트랜잭션 추적
    user = await fetch_user(user_id)
    return user
```

### 3.2 커스텀 스팬 (Span)

```python
from elasticapm import capture_span

@app.get("/dashboard")
async def get_dashboard():
    with capture_span("fetch_user_data"):
        user = await fetch_user_data()

    with capture_span("fetch_stats"):
        stats = await fetch_stats()

    with capture_span("external_api_call"):
        external_data = await call_external_api()

    return {"user": user, "stats": stats, "external": external_data}
```

### 3.3 Sentry (에러 추적)

```bash
pip install sentry-sdk
```

```python
# app/main.py
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration

sentry_sdk.init(
    dsn="https://your-sentry-dsn@sentry.io/project-id",
    integrations=[
        FastApiIntegration(),
        SqlalchemyIntegration(),
    ],
    traces_sample_rate=1.0,  # 100% 트랜잭션 샘플링
    environment="production",
)

app = FastAPI()

@app.get("/error")
async def trigger_error():
    # Sentry가 자동으로 에러 캡처
    raise ValueError("This is a test error")

# 수동 에러 보고
try:
    risky_operation()
except Exception as e:
    sentry_sdk.capture_exception(e)
```

## 4. 메트릭 수집 (Prometheus)

### 4.1 Prometheus 설정

```bash
pip install prometheus-fastapi-instrumentator
```

```python
# app/main.py
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()

# Prometheus 메트릭 자동 수집
instrumentator = Instrumentator(
    should_group_status_codes=False,
    should_ignore_untemplated=True,
    should_respect_env_var=True,
    should_instrument_requests_inprogress=True,
    excluded_handlers=["/metrics"],
    inprogress_name="fastapi_inprogress",
    inprogress_labels=True,
)

instrumentator.instrument(app).expose(app)

# /metrics 엔드포인트에서 메트릭 확인 가능
```

### 4.2 커스텀 메트릭

```python
from prometheus_client import Counter, Histogram, Gauge, Summary

# 카운터: 증가만 가능
user_registrations = Counter(
    'user_registrations_total',
    'Total number of user registrations'
)

# 히스토그램: 분포 측정
request_duration = Histogram(
    'request_duration_seconds',
    'Request duration in seconds',
    ['method', 'endpoint']
)

# 게이지: 증가/감소 가능
active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)

# 서머리: 통계 요약
response_size = Summary(
    'response_size_bytes',
    'Response size in bytes'
)

# 사용
@app.post("/register")
async def register_user(user: UserCreate):
    user_registrations.inc()  # 카운터 증가
    # ... 사용자 등록 로직 ...
    return user

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start_time = time.time()

    active_connections.inc()  # 연결 증가

    response = await call_next(request)

    # 메트릭 기록
    duration = time.time() - start_time
    request_duration.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)

    response_size.observe(len(response.body))

    active_connections.dec()  # 연결 감소

    return response
```

### 4.3 Prometheus 설정 파일

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'fastapi'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'
```

**실행:**
```bash
docker run -d \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

## 5. 대시보드 (Grafana)

### 5.1 Grafana 설정

```bash
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  grafana/grafana
```

**접속:** http://localhost:3000 (admin/admin)

### 5.2 Prometheus 데이터 소스 추가

1. Configuration → Data Sources → Add data source
2. Prometheus 선택
3. URL: http://localhost:9090
4. Save & Test

### 5.3 대시보드 패널 예시

**Request Rate:**
```promql
rate(http_requests_total[5m])
```

**Average Response Time:**
```promql
rate(request_duration_seconds_sum[5m]) / rate(request_duration_seconds_count[5m])
```

**Error Rate:**
```promql
rate(http_requests_total{status=~"5.."}[5m])
```

**Active Users:**
```promql
active_connections
```

### 5.4 JSON 대시보드 예시

```json
{
  "dashboard": {
    "title": "FastAPI Monitoring",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Response Time (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
          }
        ]
      }
    ]
  }
}
```

## 6. 알림 시스템

### 6.1 Prometheus Alertmanager

**alert.rules.yml:**
```yaml
groups:
  - name: fastapi_alerts
    interval: 30s
    rules:
      # 높은 에러율
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} requests per second"

      # 느린 응답
      - alert: SlowResponses
        expr: histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow response time"
          description: "95th percentile response time is {{ $value }}s"

      # 높은 메모리 사용
      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes / 1024 / 1024 > 1024
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value }}MB"
```

### 6.2 Slack 알림

**alertmanager.yml:**
```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

route:
  receiver: 'slack-notifications'
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
```

### 6.3 이메일 알림

```python
# app/monitoring/alerts.py
import smtplib
from email.mime.text import MIMEText
from app.core.config import settings

def send_alert_email(subject: str, body: str):
    """알림 이메일 발송"""
    msg = MIMEText(body)
    msg['Subject'] = f"[ALERT] {subject}"
    msg['From'] = settings.ALERT_FROM_EMAIL
    msg['To'] = settings.ALERT_TO_EMAIL

    with smtplib.SMTP(settings.SMTP_HOST, settings.SMTP_PORT) as server:
        server.starttls()
        server.login(settings.SMTP_USER, settings.SMTP_PASSWORD)
        server.send_message(msg)

# 사용
@app.middleware("http")
async def error_alert_middleware(request: Request, call_next):
    try:
        response = await call_next(request)

        # 에러율 확인
        if response.status_code >= 500:
            error_count.inc()

            # 에러율이 높으면 알림
            if error_count.get() > 10:
                send_alert_email(
                    "High Error Rate",
                    f"Error count: {error_count.get()}"
                )

        return response
    except Exception as e:
        send_alert_email("Unhandled Exception", str(e))
        raise
```

## 7. 헬스 체크

### 7.1 기본 헬스 체크

```python
from fastapi import status
from sqlalchemy import text

@app.get("/health", status_code=status.HTTP_200_OK)
async def health_check():
    """헬스 체크"""
    return {
        "status": "healthy",
        "version": "1.0.0"
    }

@app.get("/health/detailed")
async def detailed_health_check(db: Session = Depends(get_db)):
    """상세 헬스 체크"""
    checks = {}

    # 데이터베이스 체크
    try:
        db.execute(text("SELECT 1"))
        checks["database"] = "healthy"
    except Exception as e:
        checks["database"] = f"unhealthy: {e}"

    # Redis 체크
    try:
        await cache.redis.ping()
        checks["redis"] = "healthy"
    except Exception as e:
        checks["redis"] = f"unhealthy: {e}"

    # 전체 상태
    is_healthy = all(status == "healthy" for status in checks.values())

    return {
        "status": "healthy" if is_healthy else "unhealthy",
        "checks": checks
    }
```

### 7.2 Kubernetes Liveness/Readiness Probe

```python
@app.get("/health/live")
async def liveness():
    """Liveness Probe (앱이 살아있는지)"""
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness(db: Session = Depends(get_db)):
    """Readiness Probe (요청 받을 준비가 되었는지)"""
    try:
        # DB 연결 확인
        db.execute(text("SELECT 1"))
        return {"status": "ready"}
    except Exception:
        raise HTTPException(status_code=503, detail="Not ready")
```

**Kubernetes Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: fastapi-app:latest
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

## 8. 실전 예제: 통합 모니터링 시스템

```python
# app/monitoring/__init__.py
from .logging_config import setup_logging
from .metrics import setup_metrics
from .tracing import setup_tracing

def setup_monitoring(app: FastAPI):
    """모니터링 시스템 초기화"""
    # 로깅
    setup_logging()

    # 메트릭
    setup_metrics(app)

    # 트레이싱
    setup_tracing(app)

    # 헬스 체크
    @app.get("/health")
    async def health():
        return {"status": "healthy"}

    @app.get("/metrics")
    async def metrics():
        # Prometheus 메트릭 노출
        pass

# app/main.py
from app.monitoring import setup_monitoring

app = FastAPI()
setup_monitoring(app)
```

## 9. 모니터링 체크리스트

### 9.1 필수 메트릭

- [ ] Request Rate (요청 빈도)
- [ ] Response Time (응답 시간)
- [ ] Error Rate (에러율)
- [ ] CPU Usage (CPU 사용률)
- [ ] Memory Usage (메모리 사용률)
- [ ] Database Connection Pool
- [ ] Active Connections

### 9.2 로깅

- [ ] 구조화된 로깅 (JSON)
- [ ] 요청/응답 로깅
- [ ] 에러 로깅
- [ ] 중앙 집중식 로깅 (ELK)

### 9.3 알림

- [ ] 높은 에러율 알림
- [ ] 느린 응답 알림
- [ ] 서버 다운 알림
- [ ] 높은 리소스 사용 알림

### 9.4 대시보드

- [ ] Request/Response 차트
- [ ] 에러 추이
- [ ] 응답 시간 분포
- [ ] 리소스 사용 현황

## 10. 요약

### 핵심 포인트

1. **구조화된 로깅**
   - JSON 형식 로깅
   - 요청 ID 추적
   - 중앙 집중식 로그 수집

2. **APM**
   - Elastic APM / Sentry
   - 트랜잭션 추적
   - 에러 모니터링

3. **메트릭**
   - Prometheus 메트릭
   - 커스텀 메트릭
   - 대시보드 (Grafana)

4. **알림**
   - Alertmanager
   - Slack/이메일 알림
   - 임계값 설정

5. **헬스 체크**
   - Liveness/Readiness Probe
   - 의존성 체크

### 다음 챕터 예고

**Chapter 28: Docker와 컨테이너화 (Docker and Containerization)**

다음 챕터에서는 FastAPI 애플리케이션의 컨테이너화와 배포를 다룹니다:
- Dockerfile 작성
- Docker Compose
- 멀티 스테이지 빌드
- 최적화된 이미지 생성
- 컨테이너 오케스트레이션

완벽한 모니터링으로 프로덕션 환경을 안전하게 운영하세요!
