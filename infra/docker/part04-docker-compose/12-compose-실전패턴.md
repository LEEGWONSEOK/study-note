# Chapter 12. Compose 실전 패턴

## 12.1 웹앱 + DB 패턴 (Node.js + PostgreSQL)

### 프로젝트 구조

```
myapp/
├── docker-compose.yml
├── .env
├── .env.example
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── index.js
└── sql/
    └── init.sql
```

### .env 파일

```bash
# .env
POSTGRES_DB=myapp_db
POSTGRES_USER=myapp_user
POSTGRES_PASSWORD=securepassword123
APP_PORT=3000
NODE_ENV=development
```

### docker-compose.yml

```yaml
services:
  app:
    build:
      context: ./backend
      target: development
    ports:
      - "${APP_PORT:-3000}:3000"
    environment:
      NODE_ENV: ${NODE_ENV:-development}
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    volumes:
      - ./backend/src:/app/src    # 개발 시 소스 코드 실시간 반영
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-net

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - app-net

  adminer:
    image: adminer
    profiles:
      - tools
    ports:
      - "8080:8080"
    networks:
      - app-net

volumes:
  postgres-data:

networks:
  app-net:
    driver: bridge
```

```bash
# 개발 실행
docker compose up -d

# DB 관리 도구 포함 실행
docker compose --profile tools up -d
```

---

## 12.2 웹앱 + DB + 캐시 패턴 (Spring Boot + MySQL + Redis)

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: springapp:latest
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/${MYSQL_DATABASE}?useSSL=false&allowPublicKeyRetrieval=true
      SPRING_DATASOURCE_USERNAME: ${MYSQL_USER}
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_PASSWORD}
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
      SPRING_DATA_REDIS_PASSWORD: ${REDIS_PASSWORD}
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./sql:/docker-entrypoint-initdb.d
    command: --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
    restart: unless-stopped

volumes:
  mysql-data:
  redis-data:
```

---

## 12.3 개발/운영 환경 분리

### 파일 구조

```
project/
├── docker-compose.yml          # 기본 (공통)
├── docker-compose.override.yml # 개발 환경 (자동 병합)
├── docker-compose.prod.yml     # 운영 환경
└── .env
```

### docker-compose.yml (공통 설정)

```yaml
services:
  app:
    image: myapp:${APP_VERSION:-latest}
    environment:
      DB_HOST: db
      DB_NAME: ${DB_NAME}
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

### docker-compose.override.yml (개발용, 자동 병합)

```yaml
# 개발 환경 추가/재정의 설정
services:
  app:
    build: .                    # 개발에서는 직접 빌드
    volumes:
      - .:/app                  # 소스 코드 마운트 (핫리로드)
      - /app/node_modules       # node_modules 제외
    environment:
      NODE_ENV: development
      DEBUG: "app:*"
    ports:
      - "3000:3000"
      - "9229:9229"             # 디버거 포트

  db:
    ports:
      - "3306:3306"             # 개발 시 DB 직접 접근
    environment:
      MYSQL_ROOT_PASSWORD: devpassword    # 개발용 간단한 비밀번호
```

### docker-compose.prod.yml (운영용)

```yaml
services:
  app:
    image: registry.company.com/myapp:${APP_VERSION}
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G
    environment:
      NODE_ENV: production
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "5"

  db:
    restart: unless-stopped
    # 운영에서는 포트 외부 노출 안 함
```

```bash
# 개발 실행 (override.yml 자동 병합)
docker compose up -d

# 운영 실행 (base + prod.yml 병합)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 12.4 Nginx 리버스 프록시 패턴

```yaml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/logs:/var/log/nginx
    depends_on:
      - app1
      - app2
    restart: unless-stopped
    networks:
      - frontend

  app1:
    image: myapp1:latest
    expose:
      - "3000"        # expose만 (외부 직접 접근 불가)
    restart: unless-stopped
    networks:
      - frontend
      - backend

  app2:
    image: myapp2:latest
    expose:
      - "4000"
    restart: unless-stopped
    networks:
      - frontend
      - backend

  db:
    image: mysql:8.0
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - backend    # DB는 backend 네트워크에만 (외부 접근 차단)
    restart: unless-stopped

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # 외부 인터넷 접근 차단

volumes:
  db-data:
```

### nginx.conf 예시

```nginx
# nginx/conf.d/default.conf
upstream app1 {
    server app1:3000;
}

upstream app2 {
    server app2:4000;
}

server {
    listen 80;
    server_name _;

    # 앱 1 (경로 기반 라우팅)
    location /api/ {
        proxy_pass http://app1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 앱 2 (도메인 기반)
    location /app2/ {
        proxy_pass http://app2/;
        proxy_set_header Host $host;
    }
}
```

---

## 12.5 로깅 설정 패턴

### 중앙화된 로깅 설정

```yaml
# 공통 로깅 설정을 앵커(anchor)로 정의
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
    labels: "service"
    env: "NODE_ENV,APP_VERSION"

services:
  app:
    image: myapp
    logging: *default-logging    # 앵커 참조
    labels:
      service: app

  db:
    image: mysql:8.0
    logging: *default-logging
    labels:
      service: db
```

### Fluentd 중앙 로깅

```yaml
services:
  app:
    image: myapp
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.{{.Name}}

  fluentd:
    image: fluent/fluentd:v1.16-1
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - ./fluentd/log:/output
    ports:
      - "24224:24224"
      - "24224:24224/udp"
```

---

## 12.6 Extension fields (YAML 앵커 활용)

Compose 파일에서 중복을 줄이는 방법입니다:

```yaml
# 공통 설정 정의
x-common-env: &common-env
  TZ: Asia/Seoul
  LOG_LEVEL: info

x-common-labels: &common-labels
  labels:
    project: myapp
    env: production

x-healthcheck: &default-healthcheck
  interval: 30s
  timeout: 10s
  retries: 3

services:
  app1:
    image: service1
    environment:
      <<: *common-env         # 공통 환경변수 병합
      SERVICE_NAME: app1
    <<: *common-labels        # 공통 레이블 병합
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      <<: *default-healthcheck  # 공통 헬스체크 병합

  app2:
    image: service2
    environment:
      <<: *common-env
      SERVICE_NAME: app2
    <<: *common-labels
```

---

## 핵심 요약

1. **환경 분리**: `docker-compose.override.yml`로 개발/운영 설정 분리
2. **Nginx 리버스 프록시**: 외부는 80/443만 노출, 내부는 네트워크로 통신
3. **internal 네트워크**: DB 등 민감 서비스를 외부 인터넷에서 차단
4. **YAML 앵커**: 공통 설정 재사용으로 중복 제거
5. **profiles**: 환경별 선택적 서비스 실행
6. **로깅 설정**: max-size/max-file로 로그 용량 관리

---

## 다음 단계

[Chapter 13. 네트워크 →](../part05-네트워크와-볼륨/13-네트워크.md)

---

## 실습 과제

1. Node.js + PostgreSQL Compose 파일 작성 후 실행하기
2. 개발/운영 환경 분리 (`override.yml`) 구현하기
3. Nginx를 리버스 프록시로 두고 앱 서버에 외부 포트 노출 없애기
4. YAML 앵커로 공통 환경변수 설정 재사용하기
5. `internal: true` 네트워크로 DB 외부 접근 차단 확인하기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자