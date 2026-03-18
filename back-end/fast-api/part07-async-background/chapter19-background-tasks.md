# Chapter 19: 백그라운드 작업 (Background Tasks)

## 학습 목표
- FastAPI의 BackgroundTasks 사용법 학습
- 이메일 발송, 파일 처리 등 비동기 작업 구현
- Celery를 활용한 분산 작업 큐
- 작업 모니터링과 에러 처리
- Spring Boot의 @Async와 비교

## 1. 백그라운드 작업 개요

### 1.1 백그라운드 작업이란?

백그라운드 작업은 HTTP 응답을 반환한 후 실행되는 작업입니다. 사용자는 기다리지 않고 즉시 응답을 받습니다.

```
Client Request
    ↓
Route Handler (빠른 응답)
    ↓
Response to Client ← 사용자는 여기서 응답 받음
    ↓
Background Task 실행 (이메일 발송, 로깅 등)
```

### 1.2 사용 사례

1. **이메일 발송**: 회원가입 환영 메일, 비밀번호 재설정 메일
2. **파일 처리**: 이미지 리사이징, PDF 생성, 비디오 인코딩
3. **로깅 및 분석**: 사용자 행동 추적, 통계 수집
4. **알림**: 푸시 알림, SMS 발송
5. **데이터 동기화**: 외부 시스템과 데이터 동기화
6. **정리 작업**: 임시 파일 삭제, 캐시 정리

### 1.3 Spring Boot와 비교

| 구분 | Spring Boot | FastAPI |
|------|-------------|---------|
| **비동기 작업** | `@Async` 어노테이션 | `BackgroundTasks` |
| **작업 큐** | Spring Batch, Quartz | Celery, ARQ |
| **스레드 풀** | `TaskExecutor` 설정 | FastAPI 자동 관리 |
| **에러 처리** | `AsyncUncaughtExceptionHandler` | try-except in task |
| **결과 반환** | `CompletableFuture` | Celery AsyncResult |
| **스케줄링** | `@Scheduled` | APScheduler, Celery Beat |

**Spring Boot @Async 예시:**
```java
@Service
public class EmailService {
    @Async
    public void sendEmail(String to, String subject, String body) {
        // 이메일 발송 로직
        System.out.println("Sending email to " + to);
        // 시간이 오래 걸리는 작업...
    }
}

@RestController
public class UserController {
    @Autowired
    private EmailService emailService;

    @PostMapping("/register")
    public ResponseEntity<String> register(@RequestBody UserDTO user) {
        // 사용자 등록
        userRepository.save(user);

        // 백그라운드에서 이메일 발송
        emailService.sendEmail(user.getEmail(), "Welcome", "Welcome to our service!");

        return ResponseEntity.ok("User registered");
    }
}
```

## 2. BackgroundTasks 기본 사용법

### 2.1 간단한 백그라운드 작업

```python
from fastapi import FastAPI, BackgroundTasks
import time

app = FastAPI()

def write_log(message: str):
    """백그라운드에서 실행될 함수"""
    time.sleep(2)  # 2초 걸리는 작업
    with open("log.txt", "a") as f:
        f.write(f"{message}\n")
    print(f"Log written: {message}")

@app.post("/send-notification/{email}")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    # 백그라운드 작업 추가
    background_tasks.add_task(write_log, f"Email sent to {email}")

    # 즉시 응답 반환 (로그 작성을 기다리지 않음)
    return {"message": "Notification sent in the background"}
```

**동작 흐름:**
1. 클라이언트가 POST 요청
2. 서버가 즉시 응답 반환 (빠름)
3. 응답 후 `write_log` 함수 실행 (백그라운드)
4. 클라이언트는 기다리지 않음

### 2.2 파라미터 전달

백그라운드 함수에 여러 파라미터를 전달할 수 있습니다.

```python
from fastapi import FastAPI, BackgroundTasks
from typing import Optional

app = FastAPI()

def send_email(email: str, subject: str, body: str, cc: Optional[str] = None):
    """이메일 발송 함수"""
    print(f"Sending email to {email}")
    print(f"Subject: {subject}")
    print(f"Body: {body}")
    if cc:
        print(f"CC: {cc}")
    # 실제 이메일 발송 로직...
    time.sleep(3)  # 3초 소요
    print("Email sent!")

@app.post("/register")
async def register(
    email: str,
    username: str,
    background_tasks: BackgroundTasks
):
    # 사용자 등록 (빠른 작업)
    print(f"Registering user: {username}")

    # 백그라운드에서 환영 이메일 발송
    background_tasks.add_task(
        send_email,
        email=email,
        subject="Welcome!",
        body=f"Welcome {username} to our service!",
        cc="admin@example.com"
    )

    return {"message": "User registered", "username": username}
```

### 2.3 여러 백그라운드 작업 추가

하나의 요청에 여러 백그라운드 작업을 추가할 수 있습니다.

```python
from fastapi import FastAPI, BackgroundTasks
import time

app = FastAPI()

def send_welcome_email(email: str):
    time.sleep(2)
    print(f"Welcome email sent to {email}")

def log_registration(email: str):
    time.sleep(1)
    print(f"Logged registration for {email}")

def update_statistics():
    time.sleep(1)
    print("Statistics updated")

@app.post("/register")
async def register(email: str, background_tasks: BackgroundTasks):
    print("Registering user...")

    # 여러 백그라운드 작업 추가
    background_tasks.add_task(send_welcome_email, email)
    background_tasks.add_task(log_registration, email)
    background_tasks.add_task(update_statistics)

    return {"message": "User registered"}

# 실행 순서:
# 1. "Registering user..." 출력
# 2. 즉시 응답 반환
# 3. send_welcome_email 실행 (2초)
# 4. log_registration 실행 (1초)
# 5. update_statistics 실행 (1초)
# 총 백그라운드 작업 시간: 4초 (순차 실행)
```

## 3. 실전 예제: 이메일 발송

### 3.1 SMTP 이메일 발송

실제 이메일을 발송하는 예제입니다.

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel, EmailStr
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import os

app = FastAPI()

# 이메일 설정
SMTP_SERVER = os.getenv("SMTP_SERVER", "smtp.gmail.com")
SMTP_PORT = int(os.getenv("SMTP_PORT", "587"))
SMTP_USERNAME = os.getenv("SMTP_USERNAME")
SMTP_PASSWORD = os.getenv("SMTP_PASSWORD")
FROM_EMAIL = os.getenv("FROM_EMAIL", "noreply@example.com")

def send_email(to_email: str, subject: str, body: str):
    """SMTP를 통한 이메일 발송"""
    try:
        # 이메일 메시지 생성
        message = MIMEMultipart()
        message["From"] = FROM_EMAIL
        message["To"] = to_email
        message["Subject"] = subject

        # 본문 추가
        message.attach(MIMEText(body, "plain"))

        # SMTP 서버 연결 및 발송
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()  # TLS 시작
            server.login(SMTP_USERNAME, SMTP_PASSWORD)
            server.send_message(message)

        print(f"Email sent successfully to {to_email}")
    except Exception as e:
        print(f"Failed to send email: {e}")
        # 실제로는 로깅하고 재시도 큐에 추가

class UserRegister(BaseModel):
    username: str
    email: EmailStr
    password: str

@app.post("/register")
async def register(user: UserRegister, background_tasks: BackgroundTasks):
    # 사용자 등록 (데이터베이스 저장)
    print(f"Registering user: {user.username}")

    # 환영 이메일 발송 (백그라운드)
    welcome_email_body = f"""
    Hello {user.username}!

    Welcome to our service. We're glad to have you on board!

    Best regards,
    The Team
    """

    background_tasks.add_task(
        send_email,
        to_email=user.email,
        subject="Welcome to Our Service!",
        body=welcome_email_body
    )

    return {"message": "User registered", "username": user.username}
```

### 3.2 HTML 이메일 발송

더 풍부한 HTML 이메일을 발송하는 예제입니다.

```python
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

def send_html_email(to_email: str, subject: str, html_body: str):
    """HTML 이메일 발송"""
    try:
        message = MIMEMultipart("alternative")
        message["From"] = FROM_EMAIL
        message["To"] = to_email
        message["Subject"] = subject

        # HTML 본문
        html_part = MIMEText(html_body, "html")
        message.attach(html_part)

        # 발송
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()
            server.login(SMTP_USERNAME, SMTP_PASSWORD)
            server.send_message(message)

        print(f"HTML email sent to {to_email}")
    except Exception as e:
        print(f"Failed to send HTML email: {e}")

@app.post("/send-report")
async def send_report(email: EmailStr, background_tasks: BackgroundTasks):
    html_content = """
    <html>
      <body>
        <h1>Monthly Report</h1>
        <p>Dear User,</p>
        <p>Here is your monthly report:</p>
        <ul>
          <li>Total Users: 1,234</li>
          <li>New Signups: 56</li>
          <li>Revenue: $12,345</li>
        </ul>
        <p>Thank you for using our service!</p>
      </body>
    </html>
    """

    background_tasks.add_task(
        send_html_email,
        to_email=email,
        subject="Your Monthly Report",
        html_body=html_content
    )

    return {"message": "Report will be sent to your email"}
```

## 4. 파일 처리

### 4.1 이미지 리사이징

사용자가 업로드한 이미지를 백그라운드에서 리사이징합니다.

```python
from fastapi import FastAPI, BackgroundTasks, UploadFile, File
from PIL import Image
import io
import os

app = FastAPI()

def resize_image(image_path: str, sizes: list):
    """이미지를 여러 크기로 리사이징"""
    try:
        with Image.open(image_path) as img:
            for size in sizes:
                resized = img.resize((size, size), Image.LANCZOS)
                output_path = f"{image_path}_thumb_{size}.jpg"
                resized.save(output_path, "JPEG")
                print(f"Created thumbnail: {output_path}")
    except Exception as e:
        print(f"Failed to resize image: {e}")

@app.post("/upload-image")
async def upload_image(
    file: UploadFile = File(...),
    background_tasks: BackgroundTasks = None
):
    # 파일 저장
    upload_dir = "uploads"
    os.makedirs(upload_dir, exist_ok=True)

    file_path = f"{upload_dir}/{file.filename}"
    with open(file_path, "wb") as f:
        content = await file.read()
        f.write(content)

    # 백그라운드에서 썸네일 생성
    background_tasks.add_task(
        resize_image,
        image_path=file_path,
        sizes=[128, 256, 512]  # 3가지 크기
    )

    return {
        "message": "Image uploaded",
        "filename": file.filename,
        "thumbnails": "will be created in background"
    }
```

### 4.2 CSV 파일 생성 및 다운로드

대량의 데이터를 CSV로 내보내는 작업을 백그라운드에서 처리합니다.

```python
from fastapi import FastAPI, BackgroundTasks
import csv
from datetime import datetime

app = FastAPI()

def generate_csv_report(user_id: int, filename: str):
    """CSV 리포트 생성"""
    try:
        # 데이터 조회 (시간이 오래 걸릴 수 있음)
        data = [
            {"id": 1, "name": "Alice", "email": "alice@example.com"},
            {"id": 2, "name": "Bob", "email": "bob@example.com"},
            # ... 수천 개의 레코드
        ]

        # CSV 파일 작성
        with open(filename, "w", newline="") as csvfile:
            fieldnames = ["id", "name", "email"]
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)

            writer.writeheader()
            for row in data:
                writer.writerow(row)

        print(f"CSV report generated: {filename}")

        # 이메일로 다운로드 링크 발송 (선택적)
        # send_email(user_email, "Report Ready", f"Download: {filename}")

    except Exception as e:
        print(f"Failed to generate CSV: {e}")

@app.post("/export-users")
async def export_users(user_id: int, background_tasks: BackgroundTasks):
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = f"reports/users_{timestamp}.csv"

    # 백그라운드에서 CSV 생성
    background_tasks.add_task(generate_csv_report, user_id, filename)

    return {
        "message": "Report generation started",
        "note": "You will receive an email when it's ready"
    }
```

## 5. 로깅 및 분석

### 5.1 사용자 행동 로깅

```python
from fastapi import FastAPI, BackgroundTasks, Request
from datetime import datetime
import json

app = FastAPI()

def log_user_activity(
    user_id: int,
    action: str,
    details: dict,
    ip_address: str
):
    """사용자 활동 로깅"""
    log_entry = {
        "user_id": user_id,
        "action": action,
        "details": details,
        "ip_address": ip_address,
        "timestamp": datetime.utcnow().isoformat()
    }

    # 파일에 로그 저장
    with open("activity_log.jsonl", "a") as f:
        f.write(json.dumps(log_entry) + "\n")

    print(f"Activity logged: {action} by user {user_id}")

@app.post("/products/{product_id}/purchase")
async def purchase_product(
    product_id: int,
    user_id: int,
    quantity: int,
    request: Request,
    background_tasks: BackgroundTasks
):
    # 구매 처리
    total_price = quantity * 100  # 가격 계산
    print(f"Processing purchase: {quantity} x Product {product_id}")

    # 백그라운드에서 활동 로깅
    background_tasks.add_task(
        log_user_activity,
        user_id=user_id,
        action="purchase",
        details={
            "product_id": product_id,
            "quantity": quantity,
            "total_price": total_price
        },
        ip_address=request.client.host
    )

    return {"message": "Purchase successful", "total": total_price}
```

### 5.2 통계 업데이트

```python
from fastapi import FastAPI, BackgroundTasks
from collections import defaultdict

app = FastAPI()

# 메모리 내 통계 (실제로는 Redis나 DB 사용)
statistics = defaultdict(int)

def update_statistics(metric: str, value: int = 1):
    """통계 업데이트"""
    statistics[metric] += value
    print(f"Statistics updated: {metric} = {statistics[metric]}")

@app.get("/articles/{article_id}")
async def get_article(article_id: int, background_tasks: BackgroundTasks):
    # 기사 조회
    article = {"id": article_id, "title": "FastAPI Tutorial", "views": 1234}

    # 백그라운드에서 조회수 증가
    background_tasks.add_task(update_statistics, f"article_{article_id}_views")
    background_tasks.add_task(update_statistics, "total_views")

    return article

@app.get("/statistics")
async def get_statistics():
    """통계 조회"""
    return dict(statistics)
```

## 6. 에러 처리

### 6.1 백그라운드 작업 에러 처리

백그라운드 작업에서 발생한 에러는 응답에 영향을 주지 않지만, 적절히 로깅해야 합니다.

```python
from fastapi import FastAPI, BackgroundTasks
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

def risky_task(task_id: int):
    """에러가 발생할 수 있는 작업"""
    try:
        print(f"Starting risky task {task_id}")

        # 에러가 발생할 수 있는 작업
        if task_id == 13:
            raise ValueError("Unlucky number!")

        # 정상 작업
        print(f"Task {task_id} completed successfully")

    except Exception as e:
        # 에러 로깅
        logger.error(
            f"Background task failed: {e}",
            exc_info=True,
            extra={"task_id": task_id}
        )

        # 실패한 작업을 재시도 큐에 추가하거나
        # 관리자에게 알림 발송 등

@app.post("/start-task/{task_id}")
async def start_task(task_id: int, background_tasks: BackgroundTasks):
    background_tasks.add_task(risky_task, task_id)
    return {"message": "Task started", "task_id": task_id}
```

### 6.2 재시도 로직

실패한 작업을 자동으로 재시도하는 로직입니다.

```python
from fastapi import FastAPI, BackgroundTasks
import time
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

def retry_task(func, max_retries: int = 3, delay: int = 1, *args, **kwargs):
    """재시도 로직이 포함된 작업 실행"""
    for attempt in range(max_retries):
        try:
            func(*args, **kwargs)
            logger.info(f"Task succeeded on attempt {attempt + 1}")
            return
        except Exception as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")

            if attempt < max_retries - 1:
                time.sleep(delay * (attempt + 1))  # 지수 백오프
            else:
                logger.error(f"Task failed after {max_retries} attempts", exc_info=True)

def send_notification(user_id: int):
    """알림 발송 (실패할 수 있음)"""
    import random
    if random.random() < 0.5:  # 50% 확률로 실패
        raise ConnectionError("Notification service unavailable")
    print(f"Notification sent to user {user_id}")

@app.post("/notify/{user_id}")
async def notify_user(user_id: int, background_tasks: BackgroundTasks):
    background_tasks.add_task(
        retry_task,
        func=send_notification,
        max_retries=3,
        delay=2,
        user_id=user_id
    )

    return {"message": "Notification will be sent"}
```

## 7. Celery를 활용한 분산 작업 큐

BackgroundTasks는 간단한 작업에 적합하지만, 더 복잡한 시나리오에는 Celery가 적합합니다.

### 7.1 Celery 설치 및 설정

```bash
pip install celery redis
```

**celery_app.py:**
```python
from celery import Celery

# Celery 앱 생성
celery_app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",  # 메시지 브로커
    backend="redis://localhost:6379/0"  # 결과 저장소
)

# Celery 설정
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
)

# 작업 정의
@celery_app.task
def send_email_task(to_email: str, subject: str, body: str):
    """Celery 작업: 이메일 발송"""
    import time
    print(f"Sending email to {to_email}")
    time.sleep(5)  # 시뮬레이션
    print(f"Email sent to {to_email}")
    return {"status": "sent", "to": to_email}

@celery_app.task
def process_image_task(image_path: str):
    """Celery 작업: 이미지 처리"""
    from PIL import Image
    print(f"Processing image: {image_path}")

    with Image.open(image_path) as img:
        # 처리 로직
        img.thumbnail((800, 800))
        output_path = f"{image_path}_processed.jpg"
        img.save(output_path)

    print(f"Image processed: {output_path}")
    return {"status": "processed", "output": output_path}
```

**main.py (FastAPI):**
```python
from fastapi import FastAPI
from celery_app import send_email_task, process_image_task

app = FastAPI()

@app.post("/send-email")
async def send_email(to_email: str, subject: str, body: str):
    # Celery 작업을 큐에 추가
    task = send_email_task.delay(to_email, subject, body)

    return {
        "message": "Email task queued",
        "task_id": task.id  # 작업 ID 반환
    }

@app.get("/task-status/{task_id}")
async def get_task_status(task_id: str):
    """작업 상태 조회"""
    from celery.result import AsyncResult

    task_result = AsyncResult(task_id, app=celery_app)

    return {
        "task_id": task_id,
        "status": task_result.status,  # PENDING, SUCCESS, FAILURE 등
        "result": task_result.result if task_result.ready() else None
    }
```

**Celery Worker 실행:**
```bash
# Worker 실행
celery -A celery_app worker --loglevel=info

# Flower (모니터링 도구) 실행
celery -A celery_app flower
# http://localhost:5555 에서 모니터링 가능
```

### 7.2 BackgroundTasks vs Celery

| 기능 | BackgroundTasks | Celery |
|------|-----------------|--------|
| **설치** | 불필요 (FastAPI 내장) | 별도 설치 필요 |
| **인프라** | 불필요 | Redis/RabbitMQ 필요 |
| **분산 처리** | 불가 (단일 프로세스) | 가능 (여러 워커) |
| **작업 상태 추적** | 불가 | 가능 (task ID로 조회) |
| **재시도** | 수동 구현 | 자동 지원 |
| **스케줄링** | 불가 | 가능 (Celery Beat) |
| **우선순위** | 불가 | 가능 |
| **적합한 경우** | 간단한 작업, 작은 규모 | 복잡한 작업, 대규모 시스템 |

## 8. 주기적 작업 (Scheduled Tasks)

### 8.1 APScheduler 사용

```bash
pip install apscheduler
```

```python
from fastapi import FastAPI
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from datetime import datetime

app = FastAPI()
scheduler = AsyncIOScheduler()

def cleanup_temp_files():
    """임시 파일 정리 (주기적 실행)"""
    print(f"[{datetime.now()}] Cleaning up temporary files...")
    # 파일 삭제 로직

def send_daily_report():
    """일일 리포트 발송"""
    print(f"[{datetime.now()}] Sending daily report...")
    # 리포트 생성 및 발송

@app.on_event("startup")
async def start_scheduler():
    """애플리케이션 시작 시 스케줄러 시작"""
    # 매 시간마다 실행
    scheduler.add_job(cleanup_temp_files, "interval", hours=1)

    # 매일 오전 9시 실행
    scheduler.add_job(send_daily_report, "cron", hour=9, minute=0)

    scheduler.start()
    print("Scheduler started")

@app.on_event("shutdown")
async def shutdown_scheduler():
    """애플리케이션 종료 시 스케줄러 종료"""
    scheduler.shutdown()
    print("Scheduler shutdown")

@app.get("/")
async def root():
    return {"message": "Scheduler is running"}
```

### 8.2 Celery Beat (주기적 작업)

**celery_app.py:**
```python
from celery import Celery
from celery.schedules import crontab

celery_app = Celery("tasks", broker="redis://localhost:6379/0")

# 주기적 작업 스케줄
celery_app.conf.beat_schedule = {
    # 매 10분마다 실행
    "cleanup-every-10-minutes": {
        "task": "celery_app.cleanup_task",
        "schedule": 600.0,  # 초
    },
    # 매일 자정 실행
    "send-report-daily": {
        "task": "celery_app.daily_report_task",
        "schedule": crontab(hour=0, minute=0),
    },
    # 매주 월요일 오전 9시 실행
    "weekly-summary": {
        "task": "celery_app.weekly_summary_task",
        "schedule": crontab(day_of_week=1, hour=9, minute=0),
    },
}

@celery_app.task
def cleanup_task():
    print("Cleaning up...")

@celery_app.task
def daily_report_task():
    print("Sending daily report...")

@celery_app.task
def weekly_summary_task():
    print("Sending weekly summary...")
```

**Celery Beat 실행:**
```bash
# Beat 스케줄러 실행
celery -A celery_app beat --loglevel=info

# Worker와 함께 실행
celery -A celery_app worker --beat --loglevel=info
```

## 9. 실전 예제: 완전한 백그라운드 작업 시스템

```python
from fastapi import FastAPI, BackgroundTasks, HTTPException
from pydantic import BaseModel, EmailStr
from typing import Optional
import logging
import smtplib
from email.mime.text import MIMEText
from datetime import datetime
import json

# 로깅 설정
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Background Tasks System")

# ===== 모델 =====
class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class EmailNotification(BaseModel):
    to_email: EmailStr
    subject: str
    body: str

# ===== 백그라운드 작업 함수 =====

def send_email_with_retry(
    to_email: str,
    subject: str,
    body: str,
    max_retries: int = 3
):
    """재시도 로직이 포함된 이메일 발송"""
    for attempt in range(max_retries):
        try:
            # SMTP 이메일 발송 (실제 구현)
            logger.info(f"Sending email to {to_email} (attempt {attempt + 1})")

            # 여기에 실제 SMTP 코드
            # ...

            logger.info(f"Email sent successfully to {to_email}")
            return

        except Exception as e:
            logger.warning(f"Email send attempt {attempt + 1} failed: {e}")

            if attempt == max_retries - 1:
                logger.error(f"Failed to send email to {to_email} after {max_retries} attempts")

def log_activity(user_id: int, action: str, details: dict):
    """사용자 활동 로깅"""
    try:
        log_entry = {
            "user_id": user_id,
            "action": action,
            "details": details,
            "timestamp": datetime.utcnow().isoformat()
        }

        with open("activity.log", "a") as f:
            f.write(json.dumps(log_entry) + "\n")

        logger.info(f"Activity logged: {action} for user {user_id}")

    except Exception as e:
        logger.error(f"Failed to log activity: {e}")

def update_user_statistics(user_id: int):
    """사용자 통계 업데이트"""
    try:
        logger.info(f"Updating statistics for user {user_id}")
        # 데이터베이스 업데이트 로직
        # ...
        logger.info(f"Statistics updated for user {user_id}")

    except Exception as e:
        logger.error(f"Failed to update statistics: {e}")

def send_welcome_package(username: str, email: str):
    """환영 패키지 발송 (여러 작업 조합)"""
    try:
        # 1. 환영 이메일
        send_email_with_retry(
            to_email=email,
            subject=f"Welcome {username}!",
            body=f"Hello {username}, welcome to our platform!"
        )

        # 2. 관리자에게 알림
        send_email_with_retry(
            to_email="admin@example.com",
            subject="New User Registered",
            body=f"New user: {username} ({email})"
        )

        logger.info(f"Welcome package sent to {username}")

    except Exception as e:
        logger.error(f"Failed to send welcome package: {e}")

# ===== 엔드포인트 =====

@app.post("/register", status_code=201)
async def register_user(user: UserCreate, background_tasks: BackgroundTasks):
    """사용자 등록 (여러 백그라운드 작업 포함)"""
    # 1. 사용자 등록 (빠른 작업)
    user_id = 123  # 실제로는 DB에 저장 후 ID 받아옴
    logger.info(f"User registered: {user.username}")

    # 2. 백그라운드 작업 추가
    # 환영 패키지 발송
    background_tasks.add_task(
        send_welcome_package,
        username=user.username,
        email=user.email
    )

    # 활동 로깅
    background_tasks.add_task(
        log_activity,
        user_id=user_id,
        action="register",
        details={"username": user.username, "email": user.email}
    )

    # 통계 업데이트
    background_tasks.add_task(update_user_statistics, user_id)

    return {
        "message": "User registered successfully",
        "user_id": user_id,
        "username": user.username
    }

@app.post("/send-notification")
async def send_notification(
    notification: EmailNotification,
    background_tasks: BackgroundTasks
):
    """알림 발송"""
    background_tasks.add_task(
        send_email_with_retry,
        to_email=notification.to_email,
        subject=notification.subject,
        body=notification.body
    )

    return {"message": "Notification will be sent"}

@app.post("/users/{user_id}/actions/{action}")
async def log_user_action(
    user_id: int,
    action: str,
    details: Optional[dict] = None,
    background_tasks: BackgroundTasks = None
):
    """사용자 액션 로깅"""
    background_tasks.add_task(
        log_activity,
        user_id=user_id,
        action=action,
        details=details or {}
    )

    return {"message": "Action logged"}

@app.get("/")
async def root():
    return {"message": "Background Tasks API"}
```

## 10. 모범 사례

### 10.1 백그라운드 작업 설계 원칙

1. **빠른 응답 우선**: 사용자가 기다릴 필요 없는 작업만 백그라운드로
2. **멱등성**: 같은 작업을 여러 번 실행해도 결과가 같도록 설계
3. **에러 처리**: 백그라운드 작업의 에러는 로깅 및 모니터링
4. **재시도 로직**: 실패 가능한 작업은 자동 재시도 구현
5. **타임아웃**: 무한 실행 방지를 위한 타임아웃 설정

### 10.2 언제 BackgroundTasks를 사용할까?

**적합한 경우:**
- 이메일 발송
- 로깅 및 분석
- 간단한 파일 처리
- 알림 발송
- 통계 업데이트

**부적합한 경우 (Celery 사용 권장):**
- 장시간 실행되는 작업 (5분 이상)
- 대량의 작업 처리
- 분산 처리가 필요한 경우
- 작업 상태 추적이 필요한 경우
- 주기적 실행이 필요한 경우

### 10.3 성능 고려사항

```python
# ❌ 나쁜 예: 너무 무거운 작업
@app.post("/process")
async def process(background_tasks: BackgroundTasks):
    # 1시간 걸리는 작업을 BackgroundTasks로 실행
    background_tasks.add_task(very_heavy_task)  # 부적절!
    return {"message": "Processing"}

# ✅ 좋은 예: Celery 사용
from celery_app import heavy_task

@app.post("/process")
async def process():
    task = heavy_task.delay()  # Celery 사용
    return {"message": "Processing", "task_id": task.id}
```

## 11. 연습 문제

### 11.1 기본 연습

1. 사용자가 비밀번호 재설정을 요청하면, 백그라운드에서 재설정 링크를 이메일로 발송하는 엔드포인트를 작성하세요.

2. 파일 업로드 후 백그라운드에서 파일을 압축하여 저장하는 기능을 구현하세요.

3. 사용자 로그인 시 백그라운드에서 로그인 기록을 남기는 기능을 작성하세요.

### 11.2 중급 연습

4. 여러 이메일을 한 번에 발송하는 엔드포인트를 작성하고, 각 이메일 발송을 백그라운드 작업으로 처리하세요.

5. 백그라운드 작업이 실패하면 자동으로 3번까지 재시도하는 데코레이터를 구현하세요.

6. APScheduler를 사용하여 매일 자정에 일일 리포트를 생성하는 기능을 구현하세요.

### 11.3 고급 연습

7. Celery를 설정하고, 이미지 처리 작업을 Celery로 분산 처리하는 시스템을 구축하세요. Flower로 모니터링도 추가하세요.

8. 백그라운드 작업의 진행 상황을 WebSocket으로 실시간 전송하는 시스템을 구현하세요.

9. 백그라운드 작업 실패 시 관리자에게 알림을 보내고, 재시도 큐에 추가하는 완전한 에러 처리 시스템을 구현하세요.

## 12. 요약

### 핵심 포인트

1. **BackgroundTasks**: FastAPI 내장 백그라운드 작업
   - `background_tasks.add_task(func, *args, **kwargs)`
   - 응답 후 실행
   - 간단한 작업에 적합

2. **사용 사례**
   - 이메일 발송
   - 파일 처리
   - 로깅 및 분석
   - 알림 발송

3. **에러 처리**
   - try-except로 에러 캐치
   - 로깅 필수
   - 재시도 로직 구현

4. **Celery**: 대규모 분산 작업 큐
   - Redis/RabbitMQ 필요
   - 작업 상태 추적
   - 자동 재시도
   - Celery Beat으로 스케줄링

5. **Spring Boot 비교**
   - `@Async` ≈ `BackgroundTasks`
   - Spring Batch ≈ Celery
   - `@Scheduled` ≈ APScheduler/Celery Beat

6. **선택 기준**
   - 간단한 작업 → BackgroundTasks
   - 복잡한 작업 → Celery
   - 주기적 작업 → APScheduler or Celery Beat

### 다음 챕터 예고

**Chapter 20: 비동기 프로그래밍 (Async Programming)**

다음 챕터에서는 FastAPI의 핵심인 비동기 프로그래밍을 깊이 다룹니다:
- async/await 심화
- 동시성 vs 병렬성
- 비동기 데이터베이스 쿼리
- 비동기 HTTP 클라이언트 (httpx)
- asyncio 패턴과 모범 사례
- Spring WebFlux와 비교

백그라운드 작업과 비동기 프로그래밍을 함께 활용하면 고성능 API를 구축할 수 있습니다!
