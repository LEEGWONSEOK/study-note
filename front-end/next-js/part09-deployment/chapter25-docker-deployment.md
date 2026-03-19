# Chapter 25: Docker 배포

## 1. Docker란?

### 1.1 개념

Docker는 애플리케이션을 컨테이너로 패키징하여 어디서든 동일하게 실행할 수 있게 합니다.

```
로컬 개발 환경 = 스테이징 = 프로덕션
```

**장점:**
- 환경 일관성
- 쉬운 배포
- 확장성
- 독립성

### 1.2 용어

```
Image:     실행 파일 (레시피)
Container: 실행 중인 인스턴스 (요리)
Dockerfile: 이미지 빌드 명세
```

## 2. Next.js Dockerfile

### 2.1 기본 Dockerfile

```dockerfile
# Dockerfile
FROM node:18-alpine AS base

# 1. Dependencies 설치
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm ci

# 2. 빌드
FROM base AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# 환경 변수 (빌드 타임)
ENV NEXT_TELEMETRY_DISABLED 1

RUN npm run build

# 3. 프로덕션 이미지
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# 빌드 결과물만 복사
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

### 2.2 next.config.js 설정

```javascript
// next.config.js
module.exports = {
  output: 'standalone', // Docker용 최적화
};
```

### 2.3 .dockerignore

```
# .dockerignore
node_modules
.next
.git
.env*.local
Dockerfile
.dockerignore
README.md
```

## 3. Docker 이미지 빌드

### 3.1 빌드

```bash
docker build -t my-nextjs-app .
```

**플래그:**
```bash
-t: 이미지 이름 (tag)
.:  Dockerfile 위치
```

### 3.2 멀티 플랫폼 빌드

```bash
# ARM64 (Apple Silicon) + AMD64
docker buildx build --platform linux/amd64,linux/arm64 -t my-nextjs-app .
```

### 3.3 빌드 캐시 활용

```bash
# 캐시 사용
docker build -t my-nextjs-app .

# 캐시 무시
docker build --no-cache -t my-nextjs-app .
```

## 4. 컨테이너 실행

### 4.1 기본 실행

```bash
docker run -p 3000:3000 my-nextjs-app
```

**플래그:**
```bash
-p: 포트 매핑 (호스트:컨테이너)
-d: 백그라운드 실행
--name: 컨테이너 이름
```

### 4.2 환경 변수

```bash
# 방법 1: 직접 전달
docker run -p 3000:3000 \
  -e DATABASE_URL="postgresql://..." \
  -e NEXT_PUBLIC_API_URL="https://api.example.com" \
  my-nextjs-app

# 방법 2: 파일 사용
docker run -p 3000:3000 --env-file .env.production my-nextjs-app
```

### 4.3 볼륨 마운트

```bash
# 로그 파일 저장
docker run -p 3000:3000 \
  -v /host/logs:/app/logs \
  my-nextjs-app
```

## 5. Docker Compose

### 5.1 docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

### 5.2 실행

```bash
# 시작
docker-compose up

# 백그라운드 실행
docker-compose up -d

# 중지
docker-compose down

# 재빌드 + 시작
docker-compose up --build
```

### 5.3 개발 환경

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
      - /app/.next
    environment:
      - NODE_ENV=development
    command: npm run dev
```

**Dockerfile.dev:**
```dockerfile
# Dockerfile.dev
FROM node:18-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

**실행:**
```bash
docker-compose -f docker-compose.dev.yml up
```

## 6. 최적화

### 6.1 멀티 스테이지 빌드

```dockerfile
# 단계별 빌드로 최종 이미지 크기 축소
FROM node:18-alpine AS deps
# ...

FROM node:18-alpine AS builder
# ...

FROM node:18-alpine AS runner
# 필요한 파일만 복사
```

**효과:**
```
단일 스테이지:  1.2GB
멀티 스테이지:  150MB (90% 감소)
```

### 6.2 레이어 캐싱

```dockerfile
# ❌ 나쁜 예 (전체 재빌드)
COPY . .
RUN npm install

# ✅ 좋은 예 (package.json 변경 시만 재설치)
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

### 6.3 Alpine Linux 사용

```dockerfile
# 경량 이미지
FROM node:18-alpine  # ~50MB

# vs 일반 이미지
FROM node:18         # ~900MB
```

## 7. 프로덕션 배포

### 7.1 AWS ECS

**1. ECR에 이미지 푸시**
```bash
# ECR 로그인
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# 이미지 태그
docker tag my-nextjs-app:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/my-nextjs-app:latest

# 푸시
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/my-nextjs-app:latest
```

**2. ECS 태스크 정의**
```json
{
  "family": "my-nextjs-app",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-nextjs-app:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:db-url"
        }
      ]
    }
  ]
}
```

### 7.2 Docker Hub

```bash
# 로그인
docker login

# 태그
docker tag my-nextjs-app:latest username/my-nextjs-app:latest

# 푸시
docker push username/my-nextjs-app:latest

# 다른 서버에서 실행
docker pull username/my-nextjs-app:latest
docker run -p 3000:3000 username/my-nextjs-app:latest
```

### 7.3 자체 서버

**VPS (Ubuntu) 배포:**
```bash
# 1. Docker 설치
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 2. 이미지 받기
docker pull username/my-nextjs-app:latest

# 3. 실행
docker run -d \
  --name my-app \
  -p 80:3000 \
  --restart unless-stopped \
  --env-file .env.production \
  username/my-nextjs-app:latest

# 4. Nginx 리버스 프록시 (선택)
sudo apt install nginx

# /etc/nginx/sites-available/my-app
server {
  listen 80;
  server_name example.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
}
```

## 8. CI/CD 파이프라인

### 8.1 GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

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
          push: true
          tags: username/my-nextjs-app:latest
          cache-from: type=registry,ref=username/my-nextjs-app:buildcache
          cache-to: type=registry,ref=username/my-nextjs-app:buildcache,mode=max

      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull username/my-nextjs-app:latest
            docker stop my-app || true
            docker rm my-app || true
            docker run -d \
              --name my-app \
              -p 80:3000 \
              --restart unless-stopped \
              --env-file /home/user/.env.production \
              username/my-nextjs-app:latest
```

## 9. 모니터링

### 9.1 헬스체크

```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD node -e "require('http').get('http://localhost:3000/api/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"
```

```typescript
// app/api/health/route.ts
export async function GET() {
  // 데이터베이스 연결 확인
  try {
    await db.$queryRaw`SELECT 1`;
    return Response.json({ status: 'ok' });
  } catch (error) {
    return Response.json({ status: 'error' }, { status: 500 });
  }
}
```

### 9.2 로그 수집

```bash
# 로그 확인
docker logs my-app

# 실시간 로그
docker logs -f my-app

# 최근 100줄
docker logs --tail 100 my-app
```

## 10. 실습 문제

### 문제 1: 멀티 환경 Docker
Development, Staging, Production 환경별로 다른 설정을 사용하는 Docker 이미지를 만드세요.

### 문제 2: 무중단 배포
Blue-Green 배포 방식을 Docker Compose로 구현하세요.

### 문제 3: 모니터링 스택
Prometheus + Grafana를 추가하여 모니터링 시스템을 구축하세요.

## 11. 실습 해답

### 문제 1 해답

**Dockerfile (멀티 스테이지):**
```dockerfile
FROM node:18-alpine AS base
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM base AS development
ENV NODE_ENV=development
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS builder
COPY . .
RUN npm run build

FROM node:18-alpine AS production
ENV NODE_ENV=production
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  app-dev:
    build:
      context: .
      target: development
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db-dev:5432/mydb_dev
    profiles: ["dev"]

  app-staging:
    build:
      context: .
      target: production
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@db-staging:5432/mydb_staging
    profiles: ["staging"]

  app-prod:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@db-prod:5432/mydb_prod
    restart: unless-stopped
    profiles: ["prod"]

  db-dev:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_dev:/var/lib/postgresql/data
    profiles: ["dev"]

  db-staging:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb_staging
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_staging:/var/lib/postgresql/data
    profiles: ["staging"]

  db-prod:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb_prod
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_prod:/var/lib/postgresql/data
    profiles: ["prod"]

volumes:
  postgres_dev:
  postgres_staging:
  postgres_prod:
```

**실행:**
```bash
# Development
docker-compose --profile dev up

# Staging
docker-compose --profile staging up

# Production
docker-compose --profile prod up
```

### 문제 2 해답

```yaml
# docker-compose.blue-green.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app-blue
      - app-green

  app-blue:
    build: .
    environment:
      - NODE_ENV=production
      - VERSION=blue
    expose:
      - "3000"

  app-green:
    build: .
    environment:
      - NODE_ENV=production
      - VERSION=green
    expose:
      - "3000"
```

**nginx.conf:**
```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server app-blue:3000;
        # server app-green:3000; # 전환 시 활성화
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
        }
    }
}
```

**배포 스크립트:**
```bash
#!/bin/bash
# deploy.sh

CURRENT=$(docker-compose ps --services --filter "status=running" | grep "app-" | head -n 1)

if [ "$CURRENT" = "app-blue" ]; then
  NEW="green"
  OLD="blue"
else
  NEW="blue"
  OLD="green"
fi

echo "Current: app-$OLD"
echo "Deploying: app-$NEW"

# 1. 새 버전 시작
docker-compose up -d app-$NEW

# 2. 헬스체크
echo "Waiting for app-$NEW to be healthy..."
until [ "$(docker inspect --format='{{.State.Health.Status}}' $(docker-compose ps -q app-$NEW))" = "healthy" ]; do
  sleep 1
done

# 3. Nginx 설정 변경
sed -i "s/app-$OLD/app-$NEW/g" nginx.conf
docker-compose exec nginx nginx -s reload

# 4. 이전 버전 중지
echo "Switching traffic to app-$NEW"
sleep 5
docker-compose stop app-$OLD

echo "Deployment complete!"
```

### 문제 3 해답

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
    volumes:
      - '/:/host:ro,rslave'

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus_data:
  grafana_data:
```

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'nextjs-app'
    static_configs:
      - targets: ['app:3000']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

**Next.js 메트릭 엔드포인트:**
```typescript
// app/api/metrics/route.ts
import { NextResponse } from 'next/server';

let requestCount = 0;
let errorCount = 0;

export async function GET() {
  requestCount++;

  const metrics = `
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total ${requestCount}

# HELP http_errors_total Total HTTP errors
# TYPE http_errors_total counter
http_errors_total ${errorCount}

# HELP nodejs_memory_usage Node.js memory usage
# TYPE nodejs_memory_usage gauge
nodejs_memory_usage ${process.memoryUsage().heapUsed}
  `.trim();

  return new NextResponse(metrics, {
    headers: {
      'Content-Type': 'text/plain; charset=utf-8'
    }
  });
}
```

**실행:**
```bash
docker-compose -f docker-compose.monitoring.yml up -d

# Prometheus: http://localhost:9090
# Grafana: http://localhost:3001 (admin/admin)
```

## 핵심 요약

1. **Docker**: 환경 일관성, 쉬운 배포
2. **멀티 스테이지 빌드**: 이미지 크기 최소화
3. **Docker Compose**: 여러 컨테이너 관리
4. **프로덕션**: ECR/ECS, Docker Hub, 자체 서버
5. **CI/CD**: GitHub Actions로 자동 빌드 및 배포

[← Chapter 24: Vercel 배포](./chapter24-vercel-deployment.md) | [Part 10: 실전 프로젝트 →](../part10-projects/README.md)