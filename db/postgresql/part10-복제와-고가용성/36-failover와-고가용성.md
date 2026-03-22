# Chapter 36. Failover와 고가용성

## 🎯 이 챕터의 목표
- **Failover**의 개념과 유형 이해하기
- **수동 Failover** 절차 마스터하기
- **자동 Failover** 도구 활용하기 (Patroni, pg_auto_failover)
- **로드밸런서** 설정하기 (HAProxy, PgBouncer)
- **VIP 관리** 구현하기
- **고가용성 시스템** 완성하기

---

## 1. Failover란? (비유로 이해하기)

### 🎭 연극 공연에 비유하면

```
Failover = 주연 배우 교체
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

정상 상황:
  주연 (Primary) 🎭
    ↓
  공연 진행 중...

장애 발생:
  주연 (Primary) 🤕 갑자기 쓰러짐!
    ↓
  대기 배우 (Standby) 🎭 무대로!
    ↓
  공연 계속! (관객은 차이 못 느낌)

Failover:
  1. 주연 장애 감지
  2. 대기 배우를 주연으로 승격
  3. 관객(클라이언트)을 새 주연으로 안내
  4. 공연 재개

목표: 다운타임 최소화!
```

---

## 2. Failover 유형

### 🔧 수동 Failover (Manual Failover)

```
수동 Failover = 사람이 직접 전환
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

절차:
  1. Primary 장애 확인 (모니터링 알림)
  2. DBA가 상황 판단
  3. Standby를 Primary로 수동 승격
  4. 애플리케이션 연결 변경
  5. 서비스 재개

장점:
✅ 완전한 제어
✅ 오판 가능성 낮음
✅ 단순한 구조

단점:
❌ 느림 (사람 개입 필요)
❌ 야간/주말 대응 어려움
❌ RTO 길어짐 (수십 분)

사용 사례:
- 계획된 유지보수
- 소규모 시스템
- 24/7 대응팀 없음
```

### 🤖 자동 Failover (Automatic Failover)

```
자동 Failover = 시스템이 자동 전환
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

절차:
  1. 모니터링 시스템이 Primary 장애 감지
  2. 자동으로 Standby 승격 명령
  3. VIP 자동 이전 또는 DNS 업데이트
  4. 애플리케이션 자동 재연결
  5. 서비스 재개

장점:
✅ 매우 빠름 (수 초~수 분)
✅ 24/7 자동 대응
✅ RTO 최소화

단점:
❌ Split-brain 위험 (양쪽 모두 Primary)
❌ 복잡한 구조
❌ 오판 가능성 (네트워크 순간 단절)

사용 사례:
- 프로덕션 시스템
- 높은 가용성 요구
- 24/7 서비스
```

---

## 3. 수동 Failover 절차

### 📋 단계별 가이드

#### 1단계: Primary 장애 확인

```bash
# Primary 서버 접속 시도
ssh primary-db

# 연결 실패 시 ping 테스트
ping primary-db

# PostgreSQL 상태 확인
sudo systemctl status postgresql

# 로그 확인
sudo tail -100 /var/log/postgresql/postgresql-14-main.log
```

#### 2단계: Standby 상태 확인

```bash
# Standby 서버 접속
ssh standby-db

# PostgreSQL 실행 중인지 확인
sudo systemctl status postgresql

# Standby 모드인지 확인
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
#  pg_is_in_recovery
# -------------------
#  t
# (true = Standby 모드)

# 복제 지연 확인
sudo -u postgres psql -c "
SELECT
  pg_last_wal_receive_lsn() AS receive,
  pg_last_wal_replay_lsn() AS replay,
  pg_wal_lsn_diff(
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn()
  ) AS lag_bytes;
"
```

#### 3단계: Standby를 Primary로 승격

```bash
# 방법 1: pg_ctl promote (권장)
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/14/main

# 또는
sudo -u postgres /usr/lib/postgresql/14/bin/pg_ctl promote \
  -D /var/lib/postgresql/14/main

# 방법 2: SQL 함수 사용
sudo -u postgres psql -c "SELECT pg_promote();"

# 방법 3: 트리거 파일 생성 (PostgreSQL 11 이하)
sudo -u postgres touch /var/lib/postgresql/14/main/promote

# 로그 확인 (승격 과정 모니터링)
sudo tail -f /var/log/postgresql/postgresql-14-main.log
# 2024-01-15 15:30:00 KST [1234]: LOG:  received promote request
# 2024-01-15 15:30:01 KST [1234]: LOG:  redo done at 0/5000000
# 2024-01-15 15:30:02 KST [1234]: LOG:  selected new timeline ID: 2
# 2024-01-15 15:30:03 KST [1234]: LOG:  archive recovery complete
# 2024-01-15 15:30:04 KST [1234]: LOG:  database system is ready to accept connections
```

#### 4단계: 승격 확인

```sql
-- PostgreSQL 접속
sudo -u postgres psql

-- Primary 모드 확인
SELECT pg_is_in_recovery();
--  pg_is_in_recovery
-- -------------------
--  f
-- (false = Primary 모드) ✅

-- 쓰기 테스트
CREATE TABLE failover_test (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO failover_test DEFAULT VALUES;

SELECT * FROM failover_test;
--  id |         created_at
-- ----+----------------------------
--   1 | 2024-01-15 15:31:00+09
-- ✅ 쓰기 가능!

-- 타임라인 확인
SELECT timeline_id, redo_lsn FROM pg_control_checkpoint();
--  timeline_id | redo_lsn
-- -------------+-----------
--            2 | 0/5000000
-- (Timeline 2로 변경됨)
```

#### 5단계: 애플리케이션 연결 변경

```javascript
// 방법 1: 설정 파일 수동 변경
// config.js (Before)
const dbConfig = {
  host: '10.0.1.10',  // 구 Primary
  port: 5432,
  database: 'mydb',
  user: 'app_user',
  password: 'password'
};

// config.js (After)
const dbConfig = {
  host: '10.0.1.20',  // 새 Primary (구 Standby)
  port: 5432,
  database: 'mydb',
  user: 'app_user',
  password: 'password'
};

// 방법 2: 환경변수 사용 (권장)
// .env
DB_HOST=10.0.1.20

// 애플리케이션 재시작
pm2 restart app
```

#### 6단계: 구 Primary 재구축 (복구 후)

```bash
# 구 Primary 서버 복구 후
ssh 10.0.1.10

# PostgreSQL 중지
sudo systemctl stop postgresql

# 기존 데이터 백업
sudo mv /var/lib/postgresql/14/main /var/lib/postgresql/14/main.old

# 새 Primary에서 베이스 백업
sudo -u postgres pg_basebackup \
  -h 10.0.1.20 \
  -U replication \
  -D /var/lib/postgresql/14/main \
  -Fp -Xs -P -R

# PostgreSQL 시작 (새로운 Standby로)
sudo systemctl start postgresql

# 확인
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
#  pg_is_in_recovery
# -------------------
#  t
# ✅ Standby로 재구축 완료!
```

---

## 4. 자동 Failover: Patroni

### 🤖 Patroni란?

```
Patroni = PostgreSQL용 고가용성 솔루션
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

구성 요소:
  ├─ Patroni: PostgreSQL 관리 에이전트
  ├─ etcd/Consul/ZooKeeper: 분산 설정 저장소
  ├─ HAProxy: 로드밸런서
  └─ VIP (Virtual IP): 단일 엔드포인트

동작 원리:
  1. Patroni가 PostgreSQL 상태 모니터링
  2. Primary 장애 감지
  3. etcd를 통해 클러스터 협의
  4. Standby를 Primary로 자동 승격
  5. HAProxy가 트래픽 자동 전환
```

### 🛠️ Patroni 설치 (Ubuntu)

```bash
# 1. Python 및 pip 설치
sudo apt update
sudo apt install -y python3 python3-pip python3-dev

# 2. Patroni 설치
sudo pip3 install patroni[etcd]

# 3. etcd 설치 (분산 설정 저장소)
sudo apt install -y etcd

# 4. HAProxy 설치 (로드밸런서)
sudo apt install -y haproxy
```

### ⚙️ Patroni 설정

```yaml
# /etc/patroni/patroni.yml

scope: postgres-cluster
namespace: /db/
name: node1  # 각 노드마다 다른 이름 (node1, node2, node3)

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.10:8008

etcd:
  hosts: 10.0.1.100:2379  # etcd 서버 주소

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replication 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.10:5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  authentication:
    replication:
      username: replication
      password: replication_password
    superuser:
      username: postgres
      password: postgres_password

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

### 🚀 Patroni 시작

```bash
# Patroni 서비스 파일 생성
sudo nano /etc/systemd/system/patroni.service
```

```ini
[Unit]
Description=Patroni PostgreSQL HA
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
```

```bash
# 서비스 활성화 및 시작
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni

# 상태 확인
sudo systemctl status patroni

# 로그 확인
sudo journalctl -u patroni -f
```

### 📊 Patroni 클러스터 확인

```bash
# patronictl을 사용한 클러스터 상태 확인
patronictl -c /etc/patroni/patroni.yml list

# 출력 예시:
# + Cluster: postgres-cluster -----+----+-----------+
# | Member | Host       | Role    | State   | TL | Lag in MB |
# +--------+------------+---------+---------+----+-----------+
# | node1  | 10.0.1.10  | Leader  | running |  1 |           |
# | node2  | 10.0.1.20  | Replica | running |  1 |         0 |
# | node3  | 10.0.1.30  | Replica | running |  1 |         0 |
# +--------+------------+---------+---------+----+-----------+
```

### 🧪 자동 Failover 테스트

```bash
# Primary (Leader) 노드에서 PostgreSQL 강제 종료
sudo systemctl stop postgresql

# 10~30초 대기 후 클러스터 상태 확인
patronictl -c /etc/patroni/patroni.yml list

# 출력 예시 (자동 Failover 발생):
# + Cluster: postgres-cluster -----+----+-----------+
# | Member | Host       | Role    | State   | TL | Lag in MB |
# +--------+------------+---------+---------+----+-----------+
# | node1  | 10.0.1.10  | Replica | stopped |  1 |           |
# | node2  | 10.0.1.20  | Leader  | running |  2 |           | ← 새 Leader!
# | node3  | 10.0.1.30  | Replica | running |  2 |         0 |
# +--------+------------+---------+---------+----+-----------+

# node2가 자동으로 Leader로 승격됨! ✅
```

---

## 5. 로드밸런서: HAProxy

### 🔀 HAProxy 설정

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

```ini
# /etc/haproxy/haproxy.cfg

global
    maxconn 100
    log 127.0.0.1 local0

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

# Patroni REST API를 통한 헬스체크
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

# Primary (쓰기 전용)
listen primary
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.1.10:5432 maxconn 100 check port 8008
    server node2 10.0.1.20:5432 maxconn 100 check port 8008
    server node3 10.0.1.30:5432 maxconn 100 check port 8008

# Replicas (읽기 전용, 부하 분산)
listen replicas
    bind *:5001
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.1.10:5432 maxconn 100 check port 8008
    server node2 10.0.1.20:5432 maxconn 100 check port 8008
    server node3 10.0.1.30:5432 maxconn 100 check port 8008
```

```bash
# HAProxy 재시작
sudo systemctl restart haproxy

# 상태 확인
sudo systemctl status haproxy

# 통계 페이지 접속
# http://haproxy-server:7000
```

### 🔌 애플리케이션 연결

```javascript
// Node.js 애플리케이션
const { Pool } = require('pg');

// Primary 연결 (쓰기)
const primaryPool = new Pool({
  host: 'haproxy-server',
  port: 5000,  // HAProxy Primary 포트
  database: 'mydb',
  user: 'app_user',
  password: 'password',
  max: 20
});

// Replica 연결 (읽기, 부하 분산)
const replicaPool = new Pool({
  host: 'haproxy-server',
  port: 5001,  // HAProxy Replica 포트
  database: 'mydb',
  user: 'app_user',
  password: 'password',
  max: 50
});

// 쓰기
async function createUser(name) {
  return await primaryPool.query(
    'INSERT INTO users (name) VALUES ($1) RETURNING id',
    [name]
  );
}

// 읽기 (자동으로 여러 Replica에 분산)
async function getUsers() {
  return await replicaPool.query('SELECT * FROM users');
}
```

---

## 6. VIP (Virtual IP) 관리

### 💡 VIP란?

```
VIP = 가상 IP 주소
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

정상 상황:
  VIP: 192.168.1.100
    ↓
  Primary: 10.0.1.10 (VIP 소유)
  Standby: 10.0.1.20

Failover 발생:
  VIP: 192.168.1.100
    ↓
  구 Primary: 10.0.1.10 (VIP 해제)
  새 Primary: 10.0.1.20 (VIP 획득) ✅

애플리케이션:
  항상 192.168.1.100으로 연결
  → Primary 변경 시에도 재설정 불필요!
```

### 🛠️ Keepalived를 사용한 VIP 관리

```bash
# Keepalived 설치
sudo apt install -y keepalived

# 설정 파일
sudo nano /etc/keepalived/keepalived.conf
```

```ini
# /etc/keepalived/keepalived.conf

vrrp_script chk_patroni {
    script "/usr/local/bin/check_patroni.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100  # Primary는 101, Standby는 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret
    }

    virtual_ipaddress {
        192.168.1.100/24
    }

    track_script {
        chk_patroni
    }
}
```

```bash
# Patroni 상태 확인 스크립트
sudo nano /usr/local/bin/check_patroni.sh
```

```bash
#!/bin/bash
# check_patroni.sh

# Patroni REST API로 현재 노드가 Leader인지 확인
ROLE=$(curl -s http://localhost:8008 | jq -r '.role')

if [ "$ROLE" = "master" ]; then
    exit 0  # Leader면 성공
else
    exit 1  # Replica면 실패
fi
```

```bash
# 실행 권한 부여
sudo chmod +x /usr/local/bin/check_patroni.sh

# Keepalived 시작
sudo systemctl enable keepalived
sudo systemctl start keepalived

# VIP 확인
ip addr show eth0
# eth0: ...
#     inet 192.168.1.100/24 scope global secondary eth0
# ✅ VIP 활성화됨!
```

---

## 7. 고가용성 아키텍처 완성

### 🏗️ 완전한 HA 아키텍처

```
                    [클라이언트]
                         ↓
                   [VIP: 192.168.1.100]
                         ↓
                    [HAProxy]
                    (포트 5000/5001)
                         ↓
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
    [Patroni 1]      [Patroni 2]      [Patroni 3]
        ↓                ↓                ↓
  [PostgreSQL]     [PostgreSQL]     [PostgreSQL]
   (Leader)         (Replica)         (Replica)
    10.0.1.10        10.0.1.20         10.0.1.30
        ↓                ↓                ↓
                    [etcd Cluster]
                  (설정 저장 및 리더 선출)

데이터 흐름:
1. 쓰기: VIP → HAProxy:5000 → Leader
2. 읽기: VIP → HAProxy:5001 → Replicas (Round Robin)

Failover 시:
1. Leader 장애 감지
2. Patroni가 새 Leader 선출
3. HAProxy가 트래픽 자동 전환
4. VIP 이전 (Keepalived)
5. 클라이언트는 변화 감지 못 함!
```

### 📊 모니터링 대시보드

```sql
-- 클러스터 상태 종합 뷰
CREATE OR REPLACE VIEW v_cluster_status AS
SELECT
  application_name AS node_name,
  client_addr AS node_ip,
  state,
  sync_state,
  CASE
    WHEN sync_state = 'sync' THEN 'Primary'
    ELSE 'Replica'
  END AS role,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag_size,
  replay_lag,
  CASE
    WHEN state != 'streaming' THEN '🔴 DOWN'
    WHEN replay_lag > interval '1 minute' THEN '🟡 LAGGING'
    ELSE '🟢 OK'
  END AS health
FROM pg_stat_replication

UNION ALL

SELECT
  'Current Node' AS node_name,
  inet_server_addr() AS node_ip,
  CASE
    WHEN pg_is_in_recovery() THEN 'standby'
    ELSE 'primary'
  END AS state,
  CASE
    WHEN pg_is_in_recovery() THEN 'replica'
    ELSE 'sync'
  END AS sync_state,
  CASE
    WHEN pg_is_in_recovery() THEN 'Replica (Self)'
    ELSE 'Primary (Self)'
  END AS role,
  0 AS lag_bytes,
  '0 bytes' AS lag_size,
  interval '0' AS replay_lag,
  '🟢 OK' AS health;

-- 사용
SELECT * FROM v_cluster_status;
```

---

## 8. Failover 테스트 시나리오

### 🧪 테스트 1: Primary 장애 시뮬레이션

```bash
# 1. 현재 클러스터 상태 확인
patronictl -c /etc/patroni/patroni.yml list

# 2. Primary 노드에서 PostgreSQL 중지
# (Primary 노드에서)
sudo systemctl stop postgresql

# 3. Failover 진행 확인 (10~30초 대기)
watch -n 1 'patronictl -c /etc/patroni/patroni.yml list'

# 4. 애플리케이션 영향 확인
# (클라이언트에서)
while true; do
  psql -h haproxy-server -p 5000 -U app_user -d mydb \
    -c "SELECT now(), pg_is_in_recovery();" || echo "FAILED"
  sleep 1
done

# 예상 결과:
# - 몇 초간 연결 실패 (Failover 중)
# - 자동으로 새 Primary 연결
# - 서비스 재개 ✅
```

### 🧪 테스트 2: 네트워크 분리 (Split-brain 방지)

```bash
# 1. Primary 노드의 네트워크 차단
# (Primary 노드에서)
sudo iptables -A INPUT -s 10.0.1.0/24 -j DROP
sudo iptables -A OUTPUT -d 10.0.1.0/24 -j DROP

# 2. 클러스터 상태 확인
# (다른 노드에서)
patronictl -c /etc/patroni/patroni.yml list

# 예상 결과:
# - 격리된 노드는 자동으로 Standby로 강등
# - 다른 노드가 새 Primary로 승격
# - Split-brain 방지 ✅

# 3. 네트워크 복구
# (Primary였던 노드에서)
sudo iptables -F  # 모든 규칙 삭제

# 4. 자동으로 Replica로 재합류 확인
patronictl -c /etc/patroni/patroni.yml list
```

### 🧪 테스트 3: 계획된 Switchover

```bash
# 수동으로 Primary 변경 (다운타임 없이)
patronictl -c /etc/patroni/patroni.yml switchover \
  --master node1 \
  --candidate node2

# 진행 과정:
# 1. node1에서 새 연결 차단
# 2. 기존 트랜잭션 완료 대기
# 3. node2를 Primary로 승격
# 4. node1을 Replica로 강등
# 5. 완료!

# 다운타임: 거의 없음 (수 초)
```

---

## 9. 고가용성 체크리스트

### ✅ 프로덕션 HA 시스템 체크리스트

```
[ ] 복제 구성
    [ ] Primary + 최소 2개 Standby
    [ ] Synchronous Replication (중요 시스템)
    [ ] Replication Slot 사용
    [ ] 복제 지연 모니터링

[ ] 자동 Failover
    [ ] Patroni 또는 pg_auto_failover 설치
    [ ] etcd/Consul 클러스터 (최소 3대)
    [ ] Split-brain 방지 설정
    [ ] Failover 시간 < 30초

[ ] 로드밸런싱
    [ ] HAProxy 또는 PgBouncer
    [ ] Primary 포트 (쓰기)
    [ ] Replica 포트 (읽기)
    [ ] 헬스체크 설정

[ ] 네트워크
    [ ] VIP 또는 DNS 자동 업데이트
    [ ] 방화벽 규칙
    [ ] 네트워크 분리 (데이터/관리)

[ ] 모니터링
    [ ] 클러스터 상태 모니터링
    [ ] 복제 지연 알림 (>10초)
    [ ] Failover 이벤트 알림
    [ ] 디스크 공간 모니터링

[ ] 백업
    [ ] 별도 백업 시스템 (Replication ≠ 백업!)
    [ ] 정기 백업 (일일/주간)
    [ ] 백업 검증 (복원 테스트)

[ ] 테스트
    [ ] 정기 Failover 훈련 (월 1회)
    [ ] 재해 복구 훈련 (분기 1회)
    [ ] 부하 테스트
    [ ] 문서화 및 플레이북

[ ] 보안
    [ ] SSL/TLS 연결
    [ ] 최소 권한 원칙
    [ ] 네트워크 암호화
    [ ] 감사 로깅
```

---

## 10. 실습 과제

### 🎯 과제 1: 수동 Failover 연습
로컬 환경에서 수동 Failover 수행:

```
1. Primary + Standby 구축
2. Primary에 테스트 데이터 삽입
3. Primary 중지
4. Standby 승격 (pg_ctl promote)
5. 쓰기 가능 확인
6. 구 Primary를 새 Standby로 재구축
```

### 🎯 과제 2: HAProxy 설정
HAProxy로 읽기/쓰기 분리:

```
1. HAProxy 설치
2. Primary 포트 5000 설정
3. Replica 포트 5001 설정 (Round Robin)
4. 헬스체크 설정
5. 애플리케이션 연결 테스트
```

### 🎯 과제 3: Failover 시간 측정
자동 Failover의 RTO 측정:

```
1. Patroni 클러스터 구축
2. 계속 쿼리를 보내는 스크립트 작성
3. Primary 강제 종료
4. 첫 번째 실패부터 재연결까지 시간 측정
5. 목표: RTO < 30초
```

---

## 11. 다음 단계

Failover와 고가용성을 마스터했으니, Part 10이 완료되었습니다!

이제 Part 11: 모니터링과 튜닝으로 넘어갑니다!

### 다음 챕터 미리보기
```
Chapter 37: 시스템 카탈로그
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 배울 내용:
   ├─ 시스템 카탈로그의 구조
   ├─ pg_class, pg_attribute, pg_index
   ├─ 메타데이터 쿼리 작성
   ├─ 테이블/인덱스 정보 조회
   ├─ 의존성 추적
   └─ 시스템 뷰 활용

💡 실습:
   ├─ 전체 테이블 크기 확인
   ├─ 인덱스 사용률 분석
   ├─ 외래 키 의존성 추적
   ├─ 컬럼 타입 변경 영향 분석
   └─ 커스텀 모니터링 뷰 작성
```

---

## 📖 정리

이번 챕터에서 배운 내용:

| 개념 | 핵심 내용 |
|------|----------|
| **수동 Failover** | pg_ctl promote, 완전한 제어, 느림 |
| **자동 Failover** | Patroni, 빠른 복구 (수 초~분), 24/7 |
| **Patroni** | PostgreSQL HA 솔루션, etcd 기반 |
| **HAProxy** | 로드밸런서, 읽기/쓰기 분리, 헬스체크 |
| **VIP** | 가상 IP, Keepalived, 애플리케이션 재설정 불필요 |
| **HA 아키텍처** | 복제 + Failover + 로드밸런싱 + 모니터링 |

### 핵심 명령어

```bash
# 수동 Failover
pg_ctl promote -D /data

# Patroni 클러스터 상태
patronictl list

# 수동 Switchover
patronictl switchover --master node1 --candidate node2

# HAProxy 통계
http://haproxy:7000
```

### 가용성 수준

```
단일 서버:                     99.9%   (연간 8.76시간 다운)
복제 + 수동 Failover:          99.95%  (연간 4.38시간)
복제 + 자동 Failover:          99.99%  (연간 52.56분)
복제 + 자동 Failover + 다중AZ: 99.999% (연간 5.26분)
```

**기억하세요**: "고가용성은 기술만으로 달성되지 않습니다. 정기적인 훈련, 명확한 문서화, 자동화된 테스트가 함께 있어야 진정한 고가용성을 보장할 수 있습니다!"

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐⭐⭐ (전문가)
**학습 시간**: 150분
**대상**: 데브옵스 엔지니어, DBA, 시스템 아키텍트
**중요도**: 🔥🔥🔥🔥🔥 (매우 높음)

이전: [Chapter 35. Streaming Replication](35-streaming-replication.md)
다음: [Chapter 37. 시스템 카탈로그](../part11-모니터링과-튜닝/37-시스템-카탈로그.md)