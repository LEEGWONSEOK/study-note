# Chapter 11. Compose 파일 심화

## 11.1 서비스 설정 상세

### build 설정

이미지를 직접 빌드할 때 사용합니다:

```yaml
services:
  app:
    # 간단한 방법 (Dockerfile이 있는 경로)
    build: .

    # 상세 설정
    build:
      context: ./app           # 빌드 컨텍스트 경로
      dockerfile: Dockerfile.prod   # Dockerfile 파일명 지정
      args:                    # 빌드 인수
        NODE_ENV: production
        VERSION: 1.0.0
      target: production       # 멀티스테이지 타겟 스테이지
      cache_from:              # 캐시 소스
        - myapp:latest
      labels:
        - "com.example.version=1.0"
      platforms:               # 멀티플랫폼 빌드
        - linux/amd64
        - linux/arm64
```

### image 설정

```yaml
services:
  web:
    image: nginx:1.25

  app:
    build: .
    image: myapp:1.0    # build와 함께: 빌드 후 이 이름으로 태그
```

### ports 설정

```yaml
services:
  web:
    ports:
      # "호스트포트:컨테이너포트"
      - "80:80"
      - "443:443"

      # 특정 IP 바인딩 (보안)
      - "127.0.0.1:8080:80"

      # 랜덤 호스트 포트
      - "80"

      # 긴 형식 (명확한 설정)
      - target: 80
        host_ip: "0.0.0.0"
        published: "8080"
        protocol: tcp
        mode: host
```

### volumes 설정

```yaml
services:
  app:
    volumes:
      # Named volume
      - app-data:/app/data

      # Bind mount (호스트 경로:컨테이너 경로)
      - ./config:/app/config
      - ./logs:/app/logs

      # 읽기 전용
      - ./config:/app/config:ro

      # 긴 형식
      - type: bind
        source: ./config
        target: /app/config
        read_only: true

      - type: volume
        source: app-data
        target: /app/data
        volume:
          nocopy: true    # 초기 데이터 복사 방지
```

### networks 설정 (서비스 수준)

```yaml
services:
  app:
    networks:
      frontend:
        aliases:
          - myapp         # 이 네트워크에서 myapp으로도 접근 가능
      backend:
        ipv4_address: 172.20.0.10   # 고정 IP 설정
```

### environment 설정

```yaml
services:
  app:
    # 목록 형식
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DB_HOST=db

    # 맵 형식 (권장, 명확함)
    environment:
      NODE_ENV: production
      PORT: 3000
      DB_HOST: db

    # 값 없이 (호스트 환경변수에서 상속)
    environment:
      - AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY

    # env_file로 파일에서 읽기
    env_file:
      - .env
      - .env.local
```

### command 설정

```yaml
services:
  app:
    image: myapp
    # 이미지의 CMD 재정의
    command: node --inspect server.js

    # 배열 형식
    command: ["node", "--inspect", "server.js"]
```

### restart 정책

```yaml
services:
  app:
    restart: no              # 재시작 안 함 (기본)

  db:
    restart: always          # 항상 재시작

  worker:
    restart: on-failure      # 실패 시 재시작
    # 또는
    restart: on-failure:5    # 최대 5번 재시도

  nginx:
    restart: unless-stopped  # 수동으로 중지하지 않는 한 재시작 (권장)
```

| 정책 | 설명 |
|------|------|
| `no` | 절대 재시작 안 함 |
| `always` | 항상 재시작 (수동 중지도 재시작) |
| `on-failure` | 비정상 종료 시만 재시작 |
| `unless-stopped` | 수동 중지 제외하고 항상 재시작 (운영 권장) |

---

## 11.2 헬스체크 설정

```yaml
services:
  db:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s         # 체크 간격
      timeout: 5s           # 타임아웃
      retries: 5            # 실패 허용 횟수
      start_period: 30s     # 시작 대기 시간 (이 기간 실패는 무시)

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  # 헬스체크 비활성화
  legacy-app:
    image: old-app
    healthcheck:
      disable: true
```

---

## 11.3 리소스 제한

### Compose v3 (deploy 사용)

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.5'        # CPU 최대 50%
          memory: 512M       # 메모리 최대 512MB
        reservations:
          cpus: '0.25'       # CPU 예약
          memory: 256M       # 메모리 예약
```

### Compose 직접 설정 (단일 호스트용)

```yaml
services:
  app:
    image: myapp
    mem_limit: 512m          # 메모리 제한 (구 방식, 여전히 동작)
    mem_reservation: 256m    # 메모리 예약
    cpus: 0.5                # CPU 제한
    cpu_shares: 512          # 상대적 CPU 가중치 (기본 1024)
```

---

## 11.4 네트워크 설정 (Compose 수준)

```yaml
services:
  web:
    networks:
      - frontend

  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1

  backend:
    driver: bridge
    internal: true         # 외부 인터넷 접근 차단

  # 외부에서 생성된 네트워크 참조
  existing-network:
    external: true
    name: my-existing-network
```

---

## 11.5 볼륨 설정 (Compose 수준)

```yaml
volumes:
  # 기본 Named Volume
  mysql-data:

  # 로컬 드라이버 상세 설정
  app-logs:
    driver: local
    driver_opts:
      type: none
      device: /var/log/myapp
      o: bind

  # NFS 볼륨 (원격 스토리지)
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: nfsvers=4,addr=nfs-server.company.com
      device: ":/data"

  # 외부에서 생성된 볼륨 참조
  existing-volume:
    external: true
    name: my-existing-volume
```

---

## 11.6 프로파일(Profile) 기능

특정 상황에서만 실행할 서비스를 프로파일로 분리합니다:

```yaml
services:
  # 기본 서비스 (항상 실행)
  app:
    image: myapp

  db:
    image: mysql:8.0

  # 개발 환경에서만 실행
  adminer:
    image: adminer
    profiles:
      - dev
    ports:
      - "8081:8080"

  # 디버깅용
  redis-commander:
    image: rediscommander/redis-commander
    profiles:
      - debug
      - dev

  # 테스트 환경에서만 실행
  test-db:
    image: mysql:8.0
    profiles:
      - test
```

```bash
# 기본 실행 (프로파일 없는 서비스만)
docker compose up -d

# dev 프로파일 포함
docker compose --profile dev up -d

# 여러 프로파일
docker compose --profile dev --profile debug up -d

# 또는 환경변수
COMPOSE_PROFILES=dev,debug docker compose up -d
```

---

## 11.7 기타 유용한 설정

### container_name

```yaml
services:
  db:
    image: mysql:8.0
    container_name: myapp-database    # 고정 컨테이너 이름
    # 주의: 스케일링 불가 (중복 이름 불가)
```

### hostname

```yaml
services:
  app:
    hostname: myapp-server    # 컨테이너 내부 호스트명
```

### extra_hosts

```yaml
services:
  app:
    extra_hosts:
      - "host.docker.internal:host-gateway"  # 호스트 머신 접근
      - "myservice.local:192.168.1.100"
```

### logging 설정

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### stdin_open / tty

```yaml
services:
  debug-app:
    image: myapp
    stdin_open: true    # docker run -i
    tty: true           # docker run -t
```

### privileged / cap_add

```yaml
services:
  system-app:
    image: myapp
    privileged: true    # 전체 권한 (보안 주의)

    # 또는 특정 권한만
    cap_add:
      - NET_ADMIN
      - SYS_PTRACE
    cap_drop:
      - ALL
```

---

## 11.8 전체 예시 (Spring Boot + MySQL + Redis)

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    image: myapp:${APP_VERSION:-latest}
    container_name: myapp
    ports:
      - "${APP_PORT:-8080}:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/${DB_NAME}
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      SPRING_REDIS_HOST: redis
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILE:-prod}
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"

  db:
    image: mysql:8.0
    container_name: myapp-db
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    command: redis-server --requirepass ${REDIS_PASSWORD:-}
    volumes:
      - redis-data:/data
    restart: unless-stopped
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

---

## 핵심 요약

1. **build**: 이미지를 직접 빌드할 때 context, dockerfile, args 설정
2. **restart**: 운영 환경에는 `unless-stopped` 권장
3. **healthcheck**: 서비스 상태 모니터링 + depends_on 연계
4. **deploy.resources**: 컨테이너 리소스 제한
5. **profiles**: 환경별 선택적 서비스 실행
6. **logging**: 로그 로테이션 설정 필수

---

## 다음 단계

[Chapter 12. Compose 실전 패턴 →](12-compose-실전패턴.md)

---

## 실습 과제

1. HEALTHCHECK와 depends_on을 연계하여 DB가 준비된 후 앱 시작하기
2. 리소스 제한(`mem_limit`)으로 컨테이너 메모리 제한하기
3. `profiles`로 개발용 Adminer를 선택적으로 실행하기
4. `restart: unless-stopped` 설정 후 컨테이너 강제 종료 시 자동 재시작 확인하기
5. logging 설정으로 로그 파일 크기 제한하기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자