# Chapter 45. AWS RDS PostgreSQL

## 🎯 이 챕터의 목표
- **AWS RDS** 인스턴스 생성하기
- **파라미터 그룹** 설정하기
- **백업 및 복원** 전략 수립하기
- **Multi-AZ와 Read Replica** 구성하기
- **모니터링** (CloudWatch, Performance Insights) 활용하기

---

## 1. RDS 인스턴스 생성

### AWS Console에서 생성

```
1. RDS 대시보드 접속
2. "데이터베이스 생성" 클릭
3. 엔진 선택: PostgreSQL 14.x
4. 템플릿: 프로덕션
5. 인스턴스 클래스: db.t3.medium
6. 스토리지: 범용 SSD (gp3), 100GB
7. 마스터 사용자: postgres
8. 마스터 암호: [강력한 비밀번호]
9. 퍼블릭 액세스: 아니요 (VPC 내부만)
10. 보안 그룹: 새로 생성
11. 초기 데이터베이스 이름: mydb
12. 백업: 자동 백업 7일 보관
13. 생성 완료!
```

### AWS CLI로 생성

```bash
aws rds create-db-instance \
  --db-instance-identifier my-postgres \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 14.7 \
  --master-username postgres \
  --master-user-password MySecretPassword123 \
  --allocated-storage 100 \
  --storage-type gp3 \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --db-subnet-group-name my-db-subnet-group \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --multi-az \
  --storage-encrypted \
  --enable-cloudwatch-logs-exports postgresql upgrade
```

---

## 2. 연결 설정

### 보안 그룹 설정

```bash
# 인바운드 규칙 추가
# Type: PostgreSQL
# Port: 5432
# Source: 애플리케이션 보안 그룹 또는 특정 IP

aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app123456789
```

### 연결 테스트

```bash
# psql로 연결
psql -h my-postgres.c123456789.ap-northeast-2.rds.amazonaws.com \
     -p 5432 \
     -U postgres \
     -d mydb

# Node.js에서 연결
const { Pool } = require('pg');

const pool = new Pool({
  host: 'my-postgres.c123456789.ap-northeast-2.rds.amazonaws.com',
  port: 5432,
  database: 'mydb',
  user: 'postgres',
  password: process.env.DB_PASSWORD,
  ssl: {
    rejectUnauthorized: false,
  },
});
```

---

## 3. 파라미터 그룹

### 커스텀 파라미터 그룹 생성

```bash
# 파라미터 그룹 생성
aws rds create-db-parameter-group \
  --db-parameter-group-name my-postgres-params \
  --db-parameter-group-family postgres14 \
  --description "Custom PostgreSQL parameters"

# 파라미터 수정
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-postgres-params \
  --parameters \
    "ParameterName=max_connections,ParameterValue=200,ApplyMethod=immediate" \
    "ParameterName=shared_buffers,ParameterValue='{DBInstanceClassMemory/4096}',ApplyMethod=pending-reboot"

# 인스턴스에 적용
aws rds modify-db-instance \
  --db-instance-identifier my-postgres \
  --db-parameter-group-name my-postgres-params \
  --apply-immediately
```

### 권장 파라미터

```
max_connections: 200-500 (인스턴스 크기에 따라)
shared_buffers: 메모리의 25%
effective_cache_size: 메모리의 50-75%
work_mem: 16MB-64MB
maintenance_work_mem: 256MB-1GB
checkpoint_completion_target: 0.9
wal_buffers: 16MB
random_page_cost: 1.1 (SSD 기준)
```

---

## 4. 백업 및 복원

### 자동 백업

```
- 일일 자동 백업
- 7-35일 보관 기간 설정 가능
- 백업 윈도우 지정 (트래픽 적은 시간)
- 트랜잭션 로그 아카이빙 (PITR 가능)
```

### 스냅샷 생성

```bash
# 수동 스냅샷 생성
aws rds create-db-snapshot \
  --db-instance-identifier my-postgres \
  --db-snapshot-identifier my-postgres-snapshot-20240115

# 스냅샷 목록
aws rds describe-db-snapshots \
  --db-instance-identifier my-postgres

# 스냅샷으로 복원
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-postgres-restored \
  --db-snapshot-identifier my-postgres-snapshot-20240115
```

### PITR (Point-In-Time Recovery)

```bash
# 특정 시점으로 복원
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier my-postgres \
  --target-db-instance-identifier my-postgres-pitr \
  --restore-time 2024-01-15T14:30:00Z
```

---

## 5. Multi-AZ와 Read Replica

### Multi-AZ (고가용성)

```bash
# Multi-AZ 활성화
aws rds modify-db-instance \
  --db-instance-identifier my-postgres \
  --multi-az \
  --apply-immediately

# 특징:
# - 자동 Failover (수 분 내)
# - 동기 복제 (데이터 손실 없음)
# - 같은 엔드포인트 유지
# - 추가 비용 약 2배
```

### Read Replica (읽기 분산)

```bash
# Read Replica 생성
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-postgres-replica \
  --source-db-instance-identifier my-postgres \
  --db-instance-class db.t3.medium

# 애플리케이션에서 분리
const primaryPool = new Pool({
  host: 'my-postgres.xxx.rds.amazonaws.com',  // 쓰기
});

const replicaPool = new Pool({
  host: 'my-postgres-replica.xxx.rds.amazonaws.com',  // 읽기
});
```

---

## 6. 모니터링

### CloudWatch 메트릭

```
주요 메트릭:
- CPUUtilization: CPU 사용률
- DatabaseConnections: 연결 수
- FreeableMemory: 사용 가능 메모리
- FreeStorageSpace: 디스크 여유 공간
- ReadLatency / WriteLatency: 지연 시간
- ReadThroughput / WriteThroughput: 처리량
```

### Performance Insights

```
1. RDS 콘솔 → Performance Insights 활성화
2. 실시간 쿼리 성능 분석
3. 대기 이벤트 확인
4. Top SQL 쿼리 조회
5. 리소스 사용률 분석
```

### 알람 설정

```bash
# CloudWatch 알람 생성
aws cloudwatch put-metric-alarm \
  --alarm-name rds-cpu-high \
  --alarm-description "RDS CPU over 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=DBInstanceIdentifier,Value=my-postgres
```

---

## 7. 보안

### IAM 인증

```bash
# IAM 인증 활성화
aws rds modify-db-instance \
  --db-instance-identifier my-postgres \
  --enable-iam-database-authentication

# DB 사용자에게 IAM 권한 부여
CREATE USER iam_user WITH LOGIN;
GRANT rds_iam TO iam_user;

# Node.js에서 IAM 토큰 사용
const AWS = require('aws-sdk');
const signer = new AWS.RDS.Signer({
  region: 'ap-northeast-2',
  hostname: 'my-postgres.xxx.rds.amazonaws.com',
  port: 5432,
  username: 'iam_user',
});

const token = signer.getAuthToken();

const pool = new Pool({
  host: 'my-postgres.xxx.rds.amazonaws.com',
  user: 'iam_user',
  password: token,
  database: 'mydb',
  ssl: true,
});
```

### 암호화

```
- 저장 암호화: KMS 키 사용
- 전송 암호화: SSL/TLS 필수
- 스냅샷 암호화: 자동
```

---

## 8. 비용 최적화

```
💰 비용 절감 팁:

1. Reserved Instance: 1-3년 약정 (최대 69% 할인)
2. Aurora Serverless: 간헐적 워크로드
3. 스토리지 최적화: gp3 (gp2보다 저렴)
4. 불필요한 Read Replica 삭제
5. 백업 보관 기간 조정 (7일 vs 35일)
6. 인스턴스 크기 적정화
```

---

## 📖 정리

| 기능 | 용도 |
|------|----------|
| **Multi-AZ** | 고가용성, 자동 Failover |
| **Read Replica** | 읽기 분산, 성능 향상 |
| **자동 백업** | 일일 백업, PITR |
| **Performance Insights** | 쿼리 성능 분석 |

### 프로덕션 체크리스트

```
✅ Multi-AZ 활성화
✅ 자동 백업 7-14일
✅ 암호화 활성화
✅ 보안 그룹 최소 권한
✅ CloudWatch 알람 설정
✅ Performance Insights 활성화
✅ 파라미터 그룹 최적화
✅ 정기 스냅샷 생성
```

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐ (중급)
**학습 시간**: 90분

이전: [Chapter 44. Docker로 PostgreSQL](44-docker로-postgresql.md)

🎉 **PostgreSQL A-Z 학습 완료!** 🎉