# Chapter 29: 클라우드 배포 (Cloud Deployment)

## 학습 목표
- AWS에 FastAPI 배포 (EC2, ECS, Lambda)
- Google Cloud Platform 배포 (Cloud Run, App Engine)
- Azure 배포 (App Service)
- 클라우드 서비스 비교
- 비용 최적화 전략
- Spring Boot 클라우드 배포와 비교

## 1. AWS 배포

### 1.1 AWS EC2 배포

**EC2 인스턴스 설정:**
```bash
# 1. EC2 인스턴스 생성 (Ubuntu 22.04)
# 2. SSH 접속
ssh -i key.pem ubuntu@your-ec2-ip

# 3. 시스템 업데이트
sudo apt update && sudo apt upgrade -y

# 4. Docker 설치
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu

# 5. Docker Compose 설치
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 6. 애플리케이션 클론
git clone https://github.com/your-repo/fastapi-app.git
cd fastapi-app

# 7. 환경 변수 설정
cp .env.example .env
nano .env

# 8. 실행
docker-compose -f docker-compose.prod.yml up -d

# 9. Nginx 설치 (선택적)
sudo apt install nginx
sudo nano /etc/nginx/sites-available/fastapi
```

**Nginx 설정 (/etc/nginx/sites-available/fastapi):**
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**SSL 설정 (Let's Encrypt):**
```bash
# Certbot 설치
sudo apt install certbot python3-certbot-nginx

# SSL 인증서 발급
sudo certbot --nginx -d your-domain.com

# 자동 갱신 테스트
sudo certbot renew --dry-run
```

### 1.2 AWS ECS (Elastic Container Service)

**작업 정의 (Task Definition):**
```json
{
  "family": "fastapi-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "fastapi-container",
      "image": "your-ecr-repo/fastapi-app:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:db-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fastapi-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

**ECS CLI로 배포:**
```bash
# AWS CLI 설치 및 설정
pip install awscli
aws configure

# ECR에 이미지 푸시
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin your-account.dkr.ecr.us-east-1.amazonaws.com

docker build -t fastapi-app .
docker tag fastapi-app:latest your-account.dkr.ecr.us-east-1.amazonaws.com/fastapi-app:latest
docker push your-account.dkr.ecr.us-east-1.amazonaws.com/fastapi-app:latest

# ECS 서비스 업데이트
aws ecs update-service \
  --cluster fastapi-cluster \
  --service fastapi-service \
  --force-new-deployment
```

**Terraform으로 인프라 코드화:**
```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

# ECS 클러스터
resource "aws_ecs_cluster" "fastapi_cluster" {
  name = "fastapi-cluster"
}

# ECS 작업 정의
resource "aws_ecs_task_definition" "fastapi_task" {
  family                   = "fastapi-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn

  container_definitions = jsonencode([
    {
      name  = "fastapi-container"
      image = "your-ecr-repo/fastapi-app:latest"
      portMappings = [
        {
          containerPort = 8000
          protocol      = "tcp"
        }
      ]
      environment = [
        {
          name  = "ENVIRONMENT"
          value = "production"
        }
      ]
    }
  ])
}

# ECS 서비스
resource "aws_ecs_service" "fastapi_service" {
  name            = "fastapi-service"
  cluster         = aws_ecs_cluster.fastapi_cluster.id
  task_definition = aws_ecs_task_definition.fastapi_task.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private.*.id
    security_groups = [aws_security_group.ecs_sg.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.fastapi_tg.arn
    container_name   = "fastapi-container"
    container_port   = 8000
  }
}

# Application Load Balancer
resource "aws_lb" "fastapi_alb" {
  name               = "fastapi-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = aws_subnet.public.*.id
}

resource "aws_lb_target_group" "fastapi_tg" {
  name     = "fastapi-tg"
  port     = 8000
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  target_type = "ip"

  health_check {
    path                = "/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener" "fastapi_listener" {
  load_balancer_arn = aws_lb.fastapi_alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.fastapi_tg.arn
  }
}
```

### 1.3 AWS Lambda (서버리스)

```bash
pip install mangum
```

**Lambda 핸들러:**
```python
# app/main.py
from fastapi import FastAPI
from mangum import Mangum

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from Lambda!"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id, "name": f"User {user_id}"}

# Lambda 핸들러
handler = Mangum(app)
```

**serverless.yml (Serverless Framework):**
```yaml
service: fastapi-lambda

provider:
  name: aws
  runtime: python3.11
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  memorySize: 512
  timeout: 30

functions:
  api:
    handler: app.main.handler
    events:
      - http:
          path: /{proxy+}
          method: ANY
          cors: true
      - http:
          path: /
          method: ANY
          cors: true

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true
    slim: true
```

**배포:**
```bash
# Serverless Framework 설치
npm install -g serverless

# 배포
serverless deploy

# 로그 확인
serverless logs -f api -t
```

### 1.4 AWS RDS 및 기타 서비스 연동

```python
# app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # RDS
    DATABASE_URL: str = "postgresql://user:pass@fastapi-db.xxxx.us-east-1.rds.amazonaws.com/mydb"

    # ElastiCache (Redis)
    REDIS_URL: str = "redis://fastapi-cache.xxxx.cache.amazonaws.com:6379"

    # S3
    AWS_S3_BUCKET: str = "fastapi-uploads"
    AWS_REGION: str = "us-east-1"

    # SES (이메일)
    AWS_SES_REGION: str = "us-east-1"
    FROM_EMAIL: str = "noreply@example.com"

# S3 업로드 예시
import boto3

s3_client = boto3.client('s3', region_name=settings.AWS_REGION)

@app.post("/upload")
async def upload_file(file: UploadFile):
    s3_client.upload_fileobj(
        file.file,
        settings.AWS_S3_BUCKET,
        file.filename
    )
    return {"filename": file.filename}
```

## 2. Google Cloud Platform (GCP)

### 2.1 Cloud Run 배포

**Dockerfile (Cloud Run 최적화):**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Cloud Run은 PORT 환경 변수를 사용
CMD exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}
```

**gcloud CLI로 배포:**
```bash
# gcloud CLI 설치 및 인증
gcloud auth login
gcloud config set project your-project-id

# Cloud Run에 배포
gcloud run deploy fastapi-app \
  --source . \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --timeout 300 \
  --set-env-vars "ENVIRONMENT=production" \
  --set-secrets "DATABASE_URL=database-url:latest"

# 배포된 URL 확인
gcloud run services describe fastapi-app --region us-central1 --format "value(status.url)"
```

**cloudbuild.yaml (CI/CD):**
```yaml
steps:
  # Docker 이미지 빌드
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA', '.']

  # Container Registry에 푸시
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA']

  # Cloud Run에 배포
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'fastapi-app'
      - '--image'
      - 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA'
      - '--region'
      - 'us-central1'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'

images:
  - 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA'
```

### 2.2 App Engine 배포

**app.yaml:**
```yaml
runtime: python311
entrypoint: gunicorn -w 4 -k uvicorn.workers.UvicornWorker app.main:app

instance_class: F2

automatic_scaling:
  target_cpu_utilization: 0.65
  min_instances: 1
  max_instances: 10

env_variables:
  ENVIRONMENT: 'production'

handlers:
  - url: /.*
    script: auto
    secure: always
    redirect_http_response_code: 301
```

**배포:**
```bash
# App Engine 초기화
gcloud app create --region=us-central

# 배포
gcloud app deploy

# 로그 확인
gcloud app logs tail -s default
```

### 2.3 Cloud SQL 및 기타 서비스

```python
# Cloud SQL 연결
from google.cloud.sql.connector import Connector

connector = Connector()

def get_db_connection():
    conn = connector.connect(
        "project:region:instance-name",
        "pg8000",
        user="user",
        password="password",
        db="dbname"
    )
    return conn

# Cloud Storage
from google.cloud import storage

storage_client = storage.Client()
bucket = storage_client.bucket('fastapi-uploads')

@app.post("/upload")
async def upload_to_gcs(file: UploadFile):
    blob = bucket.blob(file.filename)
    blob.upload_from_file(file.file)
    return {"filename": file.filename}
```

## 3. Microsoft Azure

### 3.1 Azure App Service 배포

**azure-pipelines.yml:**
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  dockerRegistryServiceConnection: 'your-docker-registry'
  imageRepository: 'fastapi-app'
  containerRegistry: 'yourregistry.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        steps:
          - task: Docker@2
            displayName: Build and push image
            inputs:
              command: buildAndPush
              repository: $(imageRepository)
              dockerfile: $(dockerfilePath)
              containerRegistry: $(dockerRegistryServiceConnection)
              tags: |
                $(tag)
                latest

  - stage: Deploy
    jobs:
      - job: DeployToAppService
        steps:
          - task: AzureWebAppContainer@1
            displayName: Deploy to Azure App Service
            inputs:
              azureSubscription: 'your-subscription'
              appName: 'fastapi-app'
              containers: '$(containerRegistry)/$(imageRepository):$(tag)'
```

**Azure CLI로 배포:**
```bash
# Azure CLI 로그인
az login

# 리소스 그룹 생성
az group create --name fastapi-rg --location eastus

# App Service Plan 생성
az appservice plan create \
  --name fastapi-plan \
  --resource-group fastapi-rg \
  --is-linux \
  --sku B1

# Web App 생성
az webapp create \
  --resource-group fastapi-rg \
  --plan fastapi-plan \
  --name fastapi-app \
  --deployment-container-image-name your-dockerhub-user/fastapi-app:latest

# 환경 변수 설정
az webapp config appsettings set \
  --resource-group fastapi-rg \
  --name fastapi-app \
  --settings ENVIRONMENT=production DATABASE_URL="postgresql://..."

# 로그 확인
az webapp log tail --resource-group fastapi-rg --name fastapi-app
```

### 3.2 Azure Container Instances

```bash
# Container Instance 생성
az container create \
  --resource-group fastapi-rg \
  --name fastapi-aci \
  --image your-registry/fastapi-app:latest \
  --dns-name-label fastapi-app \
  --ports 8000 \
  --environment-variables \
    ENVIRONMENT=production \
  --secure-environment-variables \
    DATABASE_URL="postgresql://..." \
  --cpu 1 \
  --memory 1
```

## 4. 클라우드 서비스 비교

### 4.1 주요 서비스 비교

| 서비스 | AWS | GCP | Azure | 특징 |
|--------|-----|-----|-------|------|
| **컨테이너** | ECS/Fargate | Cloud Run | Container Instances | 관리형 컨테이너 |
| **서버리스** | Lambda | Cloud Functions | Azure Functions | 이벤트 기반 실행 |
| **VM** | EC2 | Compute Engine | Virtual Machines | 전통적인 서버 |
| **PaaS** | Elastic Beanstalk | App Engine | App Service | 플랫폼 관리형 |
| **오케스트레이션** | EKS | GKE | AKS | Kubernetes |
| **데이터베이스** | RDS | Cloud SQL | Azure Database | 관리형 DB |
| **스토리지** | S3 | Cloud Storage | Blob Storage | 객체 스토리지 |
| **캐시** | ElastiCache | Memorystore | Azure Cache | Redis/Memcached |

### 4.2 비용 비교 (월 예상)

**소규모 (2 vCPU, 4GB RAM, 트래픽 소량):**
- AWS ECS Fargate: ~$40-60
- GCP Cloud Run: ~$10-30 (요청 기반)
- Azure App Service: ~$50-70
- AWS Lambda: ~$5-20 (요청 기반)

**중규모 (4 vCPU, 8GB RAM, 중간 트래픽):**
- AWS ECS Fargate: ~$80-120
- GCP Cloud Run: ~$50-100
- Azure App Service: ~$100-150

**대규모 (Kubernetes, 고가용성):**
- AWS EKS: ~$300-1000+
- GCP GKE: ~$250-900+
- Azure AKS: ~$300-1000+

## 5. 배포 전략

### 5.1 블루-그린 배포

```yaml
# docker-compose.blue-green.yml
version: '3.8'

services:
  app-blue:
    image: fastapi-app:v1.0
    container_name: app-blue
    ports:
      - "8001:8000"

  app-green:
    image: fastapi-app:v1.1
    container_name: app-green
    ports:
      - "8002:8000"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
```

**Nginx 설정:**
```nginx
upstream backend {
    server app-blue:8000;  # 현재 활성
    # server app-green:8000;  # 대기
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

**배포 스크립트:**
```bash
#!/bin/bash
# deploy.sh

# 1. Green 버전 배포
docker-compose up -d app-green

# 2. 헬스 체크
until curl -f http://localhost:8002/health; do
    sleep 1
done

# 3. Nginx 설정 변경 (Blue → Green)
sed -i 's/app-blue/app-green/' nginx.conf
nginx -s reload

# 4. Blue 버전 중지
docker-compose stop app-blue

# 5. Blue 버전 제거
docker-compose rm -f app-blue
```

### 5.2 카나리 배포

```nginx
# 10%의 트래픽을 새 버전으로
upstream backend {
    server app-v1:8000 weight=9;
    server app-v2:8000 weight=1;
}
```

### 5.3 롤링 업데이트

**Kubernetes:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: fastapi
        image: fastapi-app:v1.1
```

## 6. 모니터링 및 로깅

### 6.1 AWS CloudWatch

```python
import boto3
import logging

cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

def send_metric(metric_name: str, value: float):
    """CloudWatch 메트릭 전송"""
    cloudwatch.put_metric_data(
        Namespace='FastAPI/App',
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': value,
                'Unit': 'Count'
            }
        ]
    )

# 사용
@app.post("/users")
async def create_user(user: UserCreate):
    # ... 사용자 생성 로직 ...
    send_metric('UserCreated', 1)
    return user
```

### 6.2 GCP Cloud Logging

```python
from google.cloud import logging as cloud_logging

logging_client = cloud_logging.Client()
logger = logging_client.logger('fastapi-app')

@app.middleware("http")
async def log_requests(request: Request, call_next):
    response = await call_next(request)

    logger.log_struct({
        "method": request.method,
        "path": request.url.path,
        "status_code": response.status_code,
        "user_agent": request.headers.get("user-agent")
    })

    return response
```

## 7. 비용 최적화

### 7.1 최적화 전략

**1. 적절한 인스턴스 크기:**
```python
# 부하 테스트로 적절한 크기 결정
# 예: 2 vCPU, 4GB RAM이면 충분한데 4 vCPU, 8GB 사용 중이면 낭비
```

**2. 오토 스케일링:**
```yaml
# AWS ECS Service 오토스케일링
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/cluster/service-name \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 10

aws application-autoscaling put-scaling-policy \
  --policy-name cpu-scaling \
  --service-namespace ecs \
  --resource-id service/cluster/service-name \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration \
    '{"TargetValue": 70.0, "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"}}'
```

**3. 스팟 인스턴스 활용 (AWS):**
```hcl
resource "aws_ecs_capacity_provider" "spot" {
  name = "spot-capacity-provider"

  auto_scaling_group_provider {
    auto_scaling_group_arn = aws_autoscaling_group.ecs_spot.arn

    managed_scaling {
      status          = "ENABLED"
      target_capacity = 100
    }
  }
}
```

**4. 캐싱 전략:**
- CloudFront (AWS), Cloud CDN (GCP), Azure CDN으로 정적 파일 캐싱
- Redis로 데이터베이스 쿼리 결과 캐싱

**5. 서버리스 고려:**
- 트래픽이 불규칙하면 Lambda/Cloud Functions가 비용 효율적

## 8. 보안

### 8.1 시크릿 관리

**AWS Secrets Manager:**
```python
import boto3
import json

secrets_client = boto3.client('secretsmanager', region_name='us-east-1')

def get_secret(secret_name: str) -> dict:
    response = secrets_client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# 사용
db_credentials = get_secret('prod/database/credentials')
DATABASE_URL = f"postgresql://{db_credentials['username']}:{db_credentials['password']}@..."
```

**GCP Secret Manager:**
```python
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
name = "projects/PROJECT_ID/secrets/SECRET_NAME/versions/latest"
response = client.access_secret_version(request={"name": name})
secret_value = response.payload.data.decode('UTF-8')
```

### 8.2 네트워크 보안

**AWS Security Group:**
```hcl
resource "aws_security_group" "app_sg" {
  name = "fastapi-sg"

  ingress {
    from_port   = 8000
    to_port     = 8000
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # VPC 내부만
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## 9. 요약

### 핵심 포인트

1. **AWS**
   - EC2: 전통적인 VM
   - ECS/Fargate: 관리형 컨테이너
   - Lambda: 서버리스

2. **GCP**
   - Cloud Run: 완전 관리형 컨테이너
   - App Engine: PaaS
   - 비용 효율적

3. **Azure**
   - App Service: 통합 PaaS
   - Container Instances: 간단한 컨테이너
   - 엔터프라이즈 친화적

4. **배포 전략**
   - 블루-그린
   - 카나리
   - 롤링 업데이트

5. **비용 최적화**
   - 적절한 크기 선택
   - 오토 스케일링
   - 스팟 인스턴스

### 다음 챕터 예고

**Chapter 30: Kubernetes 배포 (Kubernetes Deployment)**

다음 챕터에서는 Kubernetes 배포를 다룹니다:
- Kubernetes 기초
- Deployment, Service, Ingress
- Helm 차트
- CI/CD 파이프라인
- 모니터링 (Prometheus + Grafana)

클라우드로 확장 가능하고 안정적인 서비스를 구축하세요!