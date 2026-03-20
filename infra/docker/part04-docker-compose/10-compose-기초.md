# Chapter 10. Docker Compose 기초

## 10.1 Docker Compose란?

Docker Compose는 **여러 컨테이너를 하나의 파일로 정의하고 관리**하는 도구입니다.

> **핵심**: `docker-compose.yml` 파일 하나로 여러 서비스를 한번에 실행합니다.

**비유**: Docker Compose는 **악보(Score)** 와 같습니다.
- 여러 악기(컨테이너)의 연주를 하나의 악보로 관리
- `docker compose up` 한 번으로 전체 앙상블 시작

### 사용 배경

실제 앱은 여러 서비스로 구성됩니다:

```
웹 앱 = 앱 서버 + DB + 캐시 + 메시지 큐 + ...
```

---

## 10.2 Compose vs 개별 docker run 비교

### 개별 docker run 방식 (불편함)

```bash
# 네트워크 생성
docker network create myapp-network

# 볼륨 생성
docker volume create mysql-data

# MySQL 실행
docker run -d \
  --name mysql \
  --network myapp-network \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=mydb \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

# Redis 실행
docker run -d \
  --name redis \
  --network myapp-network \
  redis:7-alpine

# 앱 실행
docker run -d \
  --name myapp \
  --network myapp-network \
  -p 8080:8080 \
  -e DB_HOST=mysql \
  -e REDIS_HOST=redis \
  myapp:latest
```

### Docker Compose 방식 (편리함)

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: mydb
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:7-alpine

volumes:
  mysql-data:
```

```bash
# 한 번에 실행!
docker compose up -d
```

---

## 10.3 docker-compose.yml 기본 구조

```yaml
# Compose 파일 기본 구조
services:          # 필수: 서비스 정의
  서비스명1:
    image: ...
    ...
  서비스명2:
    ...

networks:          # 선택: 네트워크 정의
  네트워크명:
    ...

volumes:           # 선택: 볼륨 정의
  볼륨명:
    ...

configs:           # 선택: 설정 파일
  ...

secrets:           # 선택: 시크릿 관리
  ...
```

---

## 10.4 version 필드 안내

최신 Compose(v2)에서는 `version` 필드가 deprecated 되었습니다:

```yaml
# 구 버전 (deprecated)
version: "3.8"
services:
  ...

# 현재 권장 (version 생략)
services:
  ...
```

> **참고**: `version` 필드는 여전히 쓸 수 있지만, Docker Compose v2에서는 무시됩니다.

---

## 10.5 services, networks, volumes 섹션

### services 섹션

각 서비스(컨테이너)를 정의합니다:

```yaml
services:
  webapp:
    image: nginx:latest           # 사용할 이미지
    build: .                      # 또는 빌드
    container_name: my-webapp     # 컨테이너 이름 (선택)
    ports:
      - "80:80"
    environment:
      - NODE_ENV=production
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - frontend
    depends_on:
      - db
    restart: unless-stopped
```

### networks 섹션

```yaml
networks:
  frontend:              # 사용자 정의 네트워크
    driver: bridge
  backend:
    driver: bridge
    internal: true       # 외부 접근 차단
```

### volumes 섹션

```yaml
volumes:
  mysql-data:           # Named volume
    driver: local
  redis-data:
  app-logs:
    driver: local
    driver_opts:
      type: none
      device: /host/path/logs
      o: bind
```

---

## 10.6 기본 명령어

### docker compose up

```bash
# 서비스 시작 (포그라운드)
docker compose up

# 백그라운드 실행
docker compose up -d

# 특정 서비스만 시작
docker compose up -d webapp

# 이미지 강제 재빌드
docker compose up -d --build

# 이미지 항상 pull
docker compose up -d --pull always

# 실패 시 컨테이너 제거
docker compose up --abort-on-container-exit
```

### docker compose down

```bash
# 서비스 중지 및 컨테이너 제거
docker compose down

# 볼륨까지 제거 (데이터 삭제!)
docker compose down -v

# 이미지까지 제거
docker compose down --rmi all

# 네트워크도 제거 (기본으로 제거됨)
docker compose down --remove-orphans
```

### docker compose start / stop

```bash
# 기존 컨테이너 시작 (down 없이 일시 중지 후 재시작)
docker compose start
docker compose start webapp

# 컨테이너 중지 (제거하지 않음)
docker compose stop
docker compose stop webapp

# 재시작
docker compose restart
docker compose restart webapp
```

### docker compose ps

```bash
# 서비스 상태 확인
docker compose ps

# 출력 예시
# NAME          IMAGE        COMMAND    SERVICE   STATUS    PORTS
# myapp-db-1    mysql:8.0    ...        db        running   3306/tcp
# myapp-web-1   nginx        ...        web       running   0.0.0.0:80->80/tcp
```

### docker compose logs

```bash
# 모든 서비스 로그
docker compose logs

# 실시간 로그
docker compose logs -f

# 특정 서비스 로그
docker compose logs -f webapp

# 마지막 N줄
docker compose logs --tail 100 webapp
```

### docker compose exec

```bash
# 실행 중인 서비스 컨테이너에 명령어 실행
docker compose exec webapp bash
docker compose exec webapp sh  # Alpine

# 특정 명령어 실행
docker compose exec db mysql -u root -p
docker compose exec webapp node --version
```

---

## 10.7 의존성 관리 (depends_on)

```yaml
services:
  app:
    image: myapp
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0

  redis:
    image: redis:7-alpine
```

### 헬스체크 기반 의존성 (권장)

```yaml
services:
  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy   # DB가 healthy 상태일 때
      redis:
        condition: service_started   # Redis가 시작됐을 때

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
```

> **주의**: `depends_on`만으로는 서비스가 "준비됨(ready)" 상태를 보장하지 않습니다.
> DB가 완전히 시작되기 전에 앱이 연결 시도할 수 있습니다.
> `condition: service_healthy`를 사용하거나 앱에서 재시도 로직을 구현하세요.

---

## 10.8 환경 변수 (.env 파일)

### .env 파일 기본

```bash
# .env 파일
DB_PASSWORD=mysecretpassword
DB_NAME=myapp_db
APP_PORT=8080
IMAGE_TAG=latest
```

```yaml
# docker-compose.yml에서 자동으로 읽힘
services:
  app:
    image: myapp:${IMAGE_TAG}
    ports:
      - "${APP_PORT}:8080"

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
```

### 여러 환경 파일

```bash
# 특정 env 파일 지정
docker compose --env-file .env.prod up -d
```

### .env 파일 구성 예시

```bash
# .env (기본, 개발용)
APP_PORT=8080
DB_PASSWORD=devpassword
DB_NAME=myapp_dev
REDIS_PASSWORD=

# .env.prod (운영용)
APP_PORT=80
DB_PASSWORD=very-secure-password-here
DB_NAME=myapp_prod
REDIS_PASSWORD=redis-secret
```

---

## 핵심 요약

1. **Docker Compose**: 여러 컨테이너를 YAML 파일로 정의
2. **`docker compose up -d`**: 전체 스택 백그라운드 실행
3. **`docker compose down`**: 전체 스택 중지 및 정리
4. **`depends_on`**: 서비스 시작 순서 제어
5. **`.env` 파일**: 환경별 변수 관리
6. **version 생략**: Compose v2에서는 version 필드 불필요

---

## 다음 단계

[Chapter 11. Compose 파일 심화 →](11-compose-파일-심화.md)

---

## 실습 과제

1. Node.js + MySQL Compose 파일 작성하고 실행하기
2. `depends_on`으로 서비스 시작 순서 제어해보기
3. `.env` 파일로 비밀번호 관리하기
4. `docker compose logs -f`로 실시간 로그 확인하기
5. `docker compose down -v`로 볼륨까지 정리해보기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자