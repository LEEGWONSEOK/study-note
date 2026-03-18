# Chapter 28: Docker와 컨테이너화 (Docker and Containerization)

## 학습 목표
- Dockerfile 작성 및 최적화
- Docker Compose로 멀티 컨테이너 관리
- 멀티 스테이지 빌드
- 프로덕션 환경 설정
- Docker 이미지 최적화
- Spring Boot Docker 배포와 비교

## 1. Dockerfile 기초

### 1.1 기본 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일 복사
COPY requirements.txt .

# 의존성 설치
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 포트 노출
EXPOSE 8000

# 애플리케이션 실행
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**빌드 및 실행:**
```bash
# 이미지 빌드
docker build -t fastapi-app:latest .

# 컨테이너 실행
docker run -d -p 8000:8000 --name my-fastapi-app fastapi-app:latest

# 로그 확인
docker logs -f my-fastapi-app

# 중지 및 제거
docker stop my-fastapi-app
docker rm my-fastapi-app
```

### 1.2 멀티 스테이지 빌드

이미지 크기를 줄이고 보안을 강화합니다.

```dockerfile
# Dockerfile.multistage

# ===== Stage 1: Builder =====
FROM python:3.11-slim as builder

WORKDIR /app

# 빌드 의존성 설치
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Python 의존성 설치
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ===== Stage 2: Runtime =====
FROM python:3.11-slim

WORKDIR /app

# 시스템 의존성 (런타임만)
RUN apt-get update && apt-get install -y \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Builder 스테이지에서 설치된 패키지 복사
COPY --from=builder /root/.local /root/.local

# PATH 업데이트
ENV PATH=/root/.local/bin:$PATH

# 애플리케이션 코드 복사
COPY . .

# 비루트 사용자 생성
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**이미지 크기 비교:**
```bash
# 기본 Dockerfile: ~800MB
# 멀티 스테이지: ~200MB
```

### 1.3 최적화된 Dockerfile

```dockerfile
# Dockerfile.optimized
FROM python:3.11-slim

# 메타데이터
LABEL maintainer="your-email@example.com"
LABEL version="1.0"
LABEL description="FastAPI Application"

WORKDIR /app

# 시스템 의존성 설치 (레이어 캐싱 최적화)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Python 의존성 먼저 복사 (캐싱 활용)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY ./app ./app

# 환경 변수
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# 비루트 사용자
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# 헬스 체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000

# Gunicorn으로 실행 (프로덕션)
CMD ["gunicorn", "app.main:app", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "--keep-alive", "5"]
```

### 1.4 .dockerignore

불필요한 파일을 이미지에서 제외합니다.

```
# .dockerignore
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.so
*.egg
*.egg-info
dist
build
.venv
venv/
.env
.env.*
.git
.github
.gitignore
.idea
.vscode
*.md
Dockerfile*
docker-compose*.yml
tests/
*.log
.pytest_cache
.coverage
htmlcov/
```

### 1.5 Spring Boot와 비교

**Spring Boot Dockerfile:**
```dockerfile
# Dockerfile (Spring Boot)
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# JAR 파일 복사
COPY target/*.jar app.jar

# 비루트 사용자
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]

# 또는 멀티 스테이지
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 2. Docker Compose

### 2.1 기본 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # FastAPI 애플리케이션
  app:
    build: .
    container_name: fastapi-app
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  # PostgreSQL 데이터베이스
  db:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped

  # Redis 캐시
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:
```

**실행:**
```bash
# 서비스 시작
docker-compose up -d

# 로그 확인
docker-compose logs -f app

# 특정 서비스만 재시작
docker-compose restart app

# 중지 및 제거
docker-compose down

# 볼륨까지 제거
docker-compose down -v
```

### 2.2 프로덕션 Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.optimized
    image: fastapi-app:${VERSION:-latest}
    container_name: fastapi-app-prod
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SECRET_KEY=${SECRET_KEY}
    env_file:
      - .env.production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    networks:
      - backend
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    restart: always
    networks:
      - backend

  # Nginx 리버스 프록시
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    restart: always
    networks:
      - backend

volumes:
  postgres-data:
  redis-data:

networks:
  backend:
    driver: bridge
```

### 2.3 개발 환경 vs 프로덕션 환경

**docker-compose.dev.yml:**
```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app  # 코드 변경 즉시 반영
    environment:
      - ENVIRONMENT=development
      - DEBUG=True
    ports:
      - "8000:8000"
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=dev
      - POSTGRES_PASSWORD=dev
      - POSTGRES_DB=dev_db
    ports:
      - "5432:5432"
```

**사용:**
```bash
# 개발 환경
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

# 프로덕션 환경
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## 3. 환경 변수 관리

### 3.1 .env 파일

```bash
# .env.example
# Database
DATABASE_URL=postgresql://user:password@localhost/dbname
DB_USER=user
DB_PASSWORD=password
DB_NAME=mydb

# Redis
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# Application
SECRET_KEY=your-secret-key-here
ENVIRONMENT=development
DEBUG=True

# CORS
BACKEND_CORS_ORIGINS=["http://localhost:3000"]

# Monitoring
SENTRY_DSN=
```

### 3.2 환경별 설정

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from typing import Literal

class Settings(BaseSettings):
    ENVIRONMENT: Literal["development", "production", "test"] = "development"
    DATABASE_URL: str
    REDIS_URL: str
    SECRET_KEY: str

    class Config:
        env_file = ".env"

settings = Settings()
```

## 4. Nginx 설정

### 4.1 리버스 프록시 설정

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream fastapi_backend {
        server app:8000;
    }

    server {
        listen 80;
        server_name api.example.com;

        # 최대 업로드 크기
        client_max_body_size 100M;

        # 로깅
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        # API 프록시
        location / {
            proxy_pass http://fastapi_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 타임아웃
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # 정적 파일
        location /static/ {
            alias /app/static/;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }

        # 헬스 체크 (로깅 제외)
        location /health {
            proxy_pass http://fastapi_backend;
            access_log off;
        }
    }
}
```

### 4.2 HTTPS 설정 (Let's Encrypt)

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL 인증서
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # SSL 설정
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://fastapi_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP를 HTTPS로 리다이렉트
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$server_name$request_uri;
}
```

## 5. 데이터베이스 마이그레이션

### 5.1 초기화 스크립트

```sql
-- init.sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_email ON users(email);
```

### 5.2 Alembic 마이그레이션

```dockerfile
# Dockerfile에 추가
COPY alembic.ini .
COPY alembic ./alembic

# 시작 스크립트
COPY entrypoint.sh .
RUN chmod +x entrypoint.sh

ENTRYPOINT ["./entrypoint.sh"]
```

**entrypoint.sh:**
```bash
#!/bin/bash
set -e

# DB 연결 대기
echo "Waiting for database..."
while ! nc -z db 5432; do
  sleep 0.1
done
echo "Database is ready!"

# 마이그레이션 실행
echo "Running migrations..."
alembic upgrade head

# 애플리케이션 시작
echo "Starting application..."
exec "$@"
```

## 6. 로깅과 모니터링

### 6.1 컨테이너 로그

```yaml
# docker-compose.yml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "app=fastapi"
```

### 6.2 중앙 집중식 로깅

```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  app:
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: fastapi.app

  fluentd:
    image: fluent/fluentd:v1.16-1
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224"

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
```

## 7. CI/CD 파이프라인

### 7.1 GitHub Actions

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Dockerfile.optimized
        push: true
        tags: |
          ${{ secrets.DOCKER_USERNAME }}/fastapi-app:latest
          ${{ secrets.DOCKER_USERNAME }}/fastapi-app:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Run tests
      run: |
        docker run --rm ${{ secrets.DOCKER_USERNAME }}/fastapi-app:latest \
          pytest tests/
```

### 7.2 자동 배포

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Deploy to server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          cd /opt/fastapi-app
          docker-compose pull
          docker-compose up -d
          docker system prune -f
```

## 8. 보안 강화

### 8.1 비밀 정보 관리

```bash
# Docker Secrets 사용
echo "my-secret-password" | docker secret create db_password -

# docker-compose.yml에서 사용
services:
  db:
    image: postgres:15
    secrets:
      - db_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    external: true
```

### 8.2 보안 스캔

```bash
# Trivy로 이미지 스캔
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image fastapi-app:latest

# Docker Scout
docker scout cves fastapi-app:latest
```

## 9. 성능 최적화

### 9.1 레이어 캐싱

```dockerfile
# ❌ 나쁜 예: 매번 모든 의존성 재설치
COPY . .
RUN pip install -r requirements.txt

# ✅ 좋은 예: requirements.txt가 변경될 때만 재설치
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### 9.2 이미지 최소화

```dockerfile
# Alpine 이미지 사용
FROM python:3.11-alpine

# 불필요한 파일 제거
RUN pip install --no-cache-dir -r requirements.txt \
    && rm -rf /root/.cache

# 멀티 스테이지 빌드로 빌드 도구 제외
```

## 10. 실전 배포 체크리스트

### 10.1 빌드 전

- [ ] .dockerignore 설정
- [ ] 멀티 스테이지 빌드 적용
- [ ] 비루트 사용자 설정
- [ ] 헬스 체크 추가
- [ ] 환경 변수 분리

### 10.2 실행 전

- [ ] 데이터베이스 마이그레이션
- [ ] 환경 변수 확인
- [ ] 볼륨 마운트 확인
- [ ] 네트워크 설정
- [ ] 리소스 제한 설정

### 10.3 배포 후

- [ ] 헬스 체크 확인
- [ ] 로그 확인
- [ ] 메트릭 모니터링
- [ ] 백업 설정
- [ ] 자동 재시작 설정

## 11. 요약

### 핵심 포인트

1. **Dockerfile**
   - 멀티 스테이지 빌드
   - 레이어 캐싱 최적화
   - 비루트 사용자

2. **Docker Compose**
   - 멀티 컨테이너 관리
   - 환경별 설정
   - 네트워크와 볼륨

3. **프로덕션 배포**
   - Nginx 리버스 프록시
   - HTTPS 설정
   - 로깅과 모니터링

4. **CI/CD**
   - 자동 빌드
   - 자동 배포
   - 테스트 자동화

5. **보안**
   - 비밀 정보 관리
   - 이미지 스캔
   - 최소 권한 원칙

### 다음 챕터 예고

**Chapter 29: 클라우드 배포 (Cloud Deployment)**

다음 챕터에서는 클라우드 플랫폼 배포를 다룹니다:
- AWS (ECS, Lambda)
- Google Cloud Platform (Cloud Run)
- Azure (App Service)
- Kubernetes 배포
- 서버리스 배포

Docker로 애플리케이션을 컨테이너화하여 어디서든 동일하게 실행하세요!
