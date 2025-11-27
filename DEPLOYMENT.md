# 배포 및 인프라 가이드

## 1. 환경 분리 전략

### 1.1 환경 구성

```
┌─────────────────────────────────────────────────────────┐
│                    환경 분리 구조                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Development (개발)          Staging (스테이징)         │
│  ├── 로컬 개발 서버           ├── 테스트 서버            │
│  ├── PostgreSQL (Docker)     ├── PostgreSQL (RDS)       │
│  ├── Redis (Docker)          ├── Redis (ElastiCache)    │
│  └── .env.development        └── GitHub Secrets         │
│                                                          │
│  Production (운영)                                       │
│  ├── 운영 서버 (다중 인스턴스)                           │
│  ├── PostgreSQL (managed)                               │
│  ├── Redis (managed)                                    │
│  └── GitHub Secrets                                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 1.2 환경별 특징

#### Development (개발 환경)
- **용도**: 로컬 개발, 기능 구현 및 디버깅
- **서버**: localhost:8000
- **데이터베이스**: Docker Compose로 실행되는 PostgreSQL
- **캐시**: Docker Compose로 실행되는 Redis
- **환경변수**: `.env.development` 파일
- **DEBUG**: True
- **특징**:
  - django-debug-toolbar 활성화
  - 상세한 에러 메시지
  - 핫 리로드 지원
  - 테스트 데이터 사용

#### Staging (스테이징 환경) - 선택사항
- **용도**: 프로덕션 배포 전 최종 테스트
- **서버**: staging.yourdomain.com
- **데이터베이스**: AWS RDS, GCP Cloud SQL 등
- **캐시**: AWS ElastiCache, GCP Memorystore 등
- **환경변수**: GitHub Secrets
- **DEBUG**: False
- **특징**:
  - 프로덕션과 동일한 환경
  - 테스트 결제 연동
  - QA 테스트 수행

#### Production (운영 환경)
- **용도**: 실제 서비스 운영
- **서버**: www.yourdomain.com
- **데이터베이스**: Managed PostgreSQL (RDS, Cloud SQL 등)
- **캐시**: Managed Redis (ElastiCache, Memorystore 등)
- **환경변수**: GitHub Secrets
- **DEBUG**: False
- **특징**:
  - 고가용성 (Multi-AZ, 로드 밸런싱)
  - 자동 백업
  - 모니터링 및 알림
  - 실 결제 연동

---

## 2. 환경변수 관리

### 2.1 환경변수 구조

#### .env.development (로컬 개발용)
```bash
# Django Settings
DJANGO_SETTINGS_MODULE=config.settings.development
SECRET_KEY=your-secret-key-for-development
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1

# Database
DB_NAME=ecommerce_dev
DB_USER=postgres
DB_PASSWORD=postgres
DB_HOST=localhost
DB_PORT=5432

# Redis
REDIS_URL=redis://localhost:6379/0

# Email (개발용 - 콘솔 출력)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend

# AWS S3 (선택사항)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_STORAGE_BUCKET_NAME=

# Payment (테스트 키)
PORTONE_API_KEY=test_api_key
PORTONE_API_SECRET=test_api_secret

# Social Login (개발용 키)
KAKAO_CLIENT_ID=
KAKAO_SECRET_KEY=
NAVER_CLIENT_ID=
NAVER_SECRET_KEY=
GOOGLE_CLIENT_ID=
GOOGLE_SECRET_KEY=
```

#### GitHub Secrets (Staging/Production)
```
DJANGO_SECRET_KEY
DB_NAME
DB_USER
DB_PASSWORD
DB_HOST
DB_PORT
REDIS_URL
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_STORAGE_BUCKET_NAME
PORTONE_API_KEY
PORTONE_API_SECRET
KAKAO_CLIENT_ID
KAKAO_SECRET_KEY
NAVER_CLIENT_ID
NAVER_SECRET_KEY
GOOGLE_CLIENT_ID
GOOGLE_SECRET_KEY
SENTRY_DSN
```

### 2.2 환경변수 설정 방법

**Django Settings 분리**:
```python
# config/settings/base.py
import os
from pathlib import Path
import environ

env = environ.Env()
BASE_DIR = Path(__file__).resolve().parent.parent.parent

# config/settings/development.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME', default='ecommerce_dev'),
        'USER': env('DB_USER', default='postgres'),
        'PASSWORD': env('DB_PASSWORD', default='postgres'),
        'HOST': env('DB_HOST', default='localhost'),
        'PORT': env('DB_PORT', default='5432'),
    }
}

# config/settings/production.py
from .base import *

DEBUG = False
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT', default='5432'),
    }
}

# 보안 설정
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
```

---

## 3. GitHub Actions CI/CD

### 3.1 CI 워크플로우 (테스트 및 코드 품질 검사)

**`.github/workflows/ci.yml`**:
```yaml
name: CI - Test and Lint

on:
  pull_request:
    branches: [ main, dev ]
  push:
    branches: [ main, dev ]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'

    - name: Install dependencies
      run: |
        pip install --upgrade pip
        pip install -r requirements/development.txt

    - name: Run migrations
      env:
        DJANGO_SETTINGS_MODULE: config.settings.development
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: test_db
        DB_USER: postgres
        DB_PASSWORD: postgres
        SECRET_KEY: test-secret-key-for-ci
      run: |
        python manage.py migrate --noinput

    - name: Run tests with coverage
      env:
        DJANGO_SETTINGS_MODULE: config.settings.development
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: test_db
        DB_USER: postgres
        DB_PASSWORD: postgres
        SECRET_KEY: test-secret-key-for-ci
        REDIS_URL: redis://localhost:6379/0
      run: |
        pytest --cov=apps --cov-report=xml --cov-report=term

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        fail_ci_if_error: false

  lint:
    name: Code Quality Checks
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'

    - name: Install dependencies
      run: |
        pip install --upgrade pip
        pip install -r requirements/development.txt

    - name: Run Black (formatting check)
      run: |
        black --check .

    - name: Run isort (import sorting check)
      run: |
        isort --check-only .

    - name: Run flake8 (linting)
      run: |
        flake8 apps config

    - name: Run mypy (type checking)
      run: |
        mypy apps config
      continue-on-error: true  # 초기에는 경고만

  security:
    name: Security Scan
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run Bandit (security linter)
      run: |
        pip install bandit
        bandit -r apps config -f json -o bandit-report.json
      continue-on-error: true

    - name: Check for known vulnerabilities
      run: |
        pip install safety
        safety check --json
      continue-on-error: true
```

### 3.2 CD 워크플로우 - Development 배포

**`.github/workflows/deploy-dev.yml`**:
```yaml
name: Deploy to Development

on:
  push:
    branches: [ dev ]

jobs:
  deploy:
    name: Deploy to Dev Server
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Deploy to Development Server
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.DEV_SERVER_HOST }}
        username: ${{ secrets.DEV_SERVER_USER }}
        key: ${{ secrets.DEV_SERVER_SSH_KEY }}
        script: |
          cd /var/www/ecommerce-dev
          git pull origin dev
          source venv/bin/activate
          pip install -r requirements/development.txt
          python manage.py migrate --noinput
          python manage.py collectstatic --noinput
          sudo systemctl restart gunicorn-dev
          sudo systemctl restart celery-dev
```

### 3.3 CD 워크플로우 - Production 배포

**`.github/workflows/deploy-prod.yml`**:
```yaml
name: Deploy to Production

on:
  push:
    branches: [ main ]
  workflow_dispatch:  # 수동 트리거 허용

jobs:
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production  # 승인 필요

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Run Final Tests
      run: |
        pip install -r requirements/production.txt
        pytest --maxfail=1

    - name: Build Docker Image
      run: |
        docker build -t ecommerce-app:${{ github.sha }} .
        docker tag ecommerce-app:${{ github.sha }} ecommerce-app:latest

    - name: Push to Container Registry
      env:
        REGISTRY_URL: ${{ secrets.CONTAINER_REGISTRY_URL }}
        REGISTRY_USER: ${{ secrets.CONTAINER_REGISTRY_USER }}
        REGISTRY_PASSWORD: ${{ secrets.CONTAINER_REGISTRY_PASSWORD }}
      run: |
        echo $REGISTRY_PASSWORD | docker login $REGISTRY_URL -u $REGISTRY_USER --password-stdin
        docker push ecommerce-app:${{ github.sha }}
        docker push ecommerce-app:latest

    - name: Deploy to Production
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.PROD_SERVER_HOST }}
        username: ${{ secrets.PROD_SERVER_USER }}
        key: ${{ secrets.PROD_SERVER_SSH_KEY }}
        script: |
          cd /var/www/ecommerce-prod
          docker-compose pull
          docker-compose up -d --force-recreate
          docker-compose exec -T web python manage.py migrate --noinput
          docker-compose exec -T web python manage.py collectstatic --noinput

    - name: Send Deployment Notification
      if: always()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: 'Production deployment ${{ job.status }}'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 4. Docker 배포 설정

### 4.1 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

# 환경 변수 설정
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# 작업 디렉토리 설정
WORKDIR /app

# 시스템 패키지 설치
RUN apt-get update && apt-get install -y \
    postgresql-client \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Python 의존성 설치
COPY requirements/production.txt requirements.txt
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 정적 파일 디렉토리 생성
RUN mkdir -p /app/staticfiles /app/media

# Gunicorn 설정
EXPOSE 8000

# 엔트리포인트 스크립트
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "--timeout", "120", "config.wsgi:application"]
```

### 4.2 Docker Compose (개발 환경)

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ecommerce_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env.development
    depends_on:
      - db
      - redis

  celery:
    build: .
    command: celery -A config worker --loglevel=info
    volumes:
      - .:/app
    env_file:
      - .env.development
    depends_on:
      - db
      - redis

  celery-beat:
    build: .
    command: celery -A config beat --loglevel=info
    volumes:
      - .:/app
    env_file:
      - .env.development
    depends_on:
      - db
      - redis

volumes:
  postgres_data:
```

### 4.3 Docker Compose (프로덕션)

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  web:
    image: ecommerce-app:latest
    restart: always
    volumes:
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    environment:
      - DJANGO_SETTINGS_MODULE=config.settings.production
    env_file:
      - .env.production
    depends_on:
      - redis

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - static_volume:/static
      - media_volume:/media
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - web

  redis:
    image: redis:7
    restart: always

  celery:
    image: ecommerce-app:latest
    command: celery -A config worker --loglevel=info
    restart: always
    env_file:
      - .env.production
    depends_on:
      - redis

  celery-beat:
    image: ecommerce-app:latest
    command: celery -A config beat --loglevel=info
    restart: always
    env_file:
      - .env.production
    depends_on:
      - redis

volumes:
  static_volume:
  media_volume:
```

### 4.4 Entrypoint Script

```bash
#!/bin/bash
# docker-entrypoint.sh

set -e

echo "Waiting for postgres..."
while ! nc -z $DB_HOST $DB_PORT; do
  sleep 0.1
done
echo "PostgreSQL started"

# 마이그레이션 실행
python manage.py migrate --noinput

# 정적 파일 수집
python manage.py collectstatic --noinput

# Gunicorn 실행
exec "$@"
```

---

## 5. 데이터베이스 마이그레이션 전략

### 5.1 마이그레이션 원칙

1. **무중단 배포 지원**
   - 하위 호환성 유지
   - 2단계 배포 (스키마 추가 → 코드 배포 → 스키마 삭제)

2. **롤백 가능성**
   - 모든 마이그레이션은 롤백 가능해야 함
   - 데이터 손실 방지

3. **성능 고려**
   - 대용량 테이블 마이그레이션 시 시간 측정
   - 필요 시 인덱스 생성을 별도 마이그레이션으로 분리

### 5.2 마이그레이션 체크리스트

```bash
# 마이그레이션 전 체크리스트
- [ ] 로컬에서 마이그레이션 테스트
- [ ] 스테이징 환경에서 테스트
- [ ] 프로덕션 데이터베이스 백업
- [ ] 마이그레이션 실행 시간 예측
- [ ] 롤백 계획 수립

# 마이그레이션 실행
python manage.py migrate --plan  # 실행 계획 확인
python manage.py migrate         # 실행

# 롤백 (필요 시)
python manage.py migrate app_name migration_name
```

### 5.3 마이그레이션 자동화 (GitHub Actions)

```yaml
# .github/workflows/migration-check.yml
name: Migration Check

on:
  pull_request:
    paths:
      - '**/migrations/**'

jobs:
  check-migrations:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Check for migration conflicts
      run: |
        python manage.py makemigrations --check --dry-run

    - name: Test migrations
      run: |
        python manage.py migrate
```

---

## 6. 모니터링 및 로깅

### 6.1 Sentry 설정 (에러 트래킹)

```python
# config/settings/production.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration

sentry_sdk.init(
    dsn=env('SENTRY_DSN'),
    integrations=[
        DjangoIntegration(),
        CeleryIntegration(),
    ],
    traces_sample_rate=0.1,  # 10%의 트랜잭션 추적
    send_default_pii=False,  # 개인정보 전송 비활성화
    environment=env('DJANGO_ENV', default='production'),
)
```

### 6.2 로깅 설정

```python
# config/settings/base.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': 'logs/django.log',
            'maxBytes': 1024 * 1024 * 10,  # 10MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console', 'file'],
        'level': 'INFO',
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
        'apps': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}
```

### 6.3 헬스 체크 엔드포인트

```python
# apps/core/views.py
from django.http import JsonResponse
from django.db import connection
from django.core.cache import cache

def health_check(request):
    """헬스 체크 엔드포인트"""
    try:
        # DB 연결 확인
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")

        # Redis 연결 확인
        cache.set('health_check', 'ok', 10)
        cache.get('health_check')

        return JsonResponse({
            'status': 'healthy',
            'database': 'ok',
            'cache': 'ok',
        })
    except Exception as e:
        return JsonResponse({
            'status': 'unhealthy',
            'error': str(e),
        }, status=503)
```

---

## 7. 백업 및 복구 전략

### 7.1 데이터베이스 백업

**자동 백업 스크립트**:
```bash
#!/bin/bash
# backup_db.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/postgres"
DB_NAME="ecommerce_prod"

# 백업 실행
pg_dump -U postgres -h localhost $DB_NAME | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# 7일 이상 된 백업 삭제
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: backup_$DATE.sql.gz"
```

**Cron 설정** (매일 새벽 2시):
```bash
0 2 * * * /scripts/backup_db.sh >> /var/log/backup.log 2>&1
```

### 7.2 미디어 파일 백업

```bash
#!/bin/bash
# backup_media.sh

DATE=$(date +%Y%m%d)
MEDIA_DIR="/var/www/ecommerce/media"
BACKUP_DIR="/backups/media"

# rsync로 증분 백업
rsync -avz --delete $MEDIA_DIR/ $BACKUP_DIR/latest/

# S3 업로드 (선택사항)
aws s3 sync $BACKUP_DIR/latest/ s3://your-bucket/media-backup/$DATE/

echo "Media backup completed"
```

---

## 8. 배포 체크리스트

### 8.1 배포 전 체크리스트

```markdown
- [ ] 모든 테스트 통과 확인
- [ ] 코드 리뷰 완료
- [ ] 마이그레이션 파일 확인
- [ ] 환경변수 설정 확인
- [ ] 데이터베이스 백업 완료
- [ ] 미디어 파일 백업 완료
- [ ] 정적 파일 수집 확인
- [ ] SSL 인증서 유효성 확인
- [ ] 모니터링 설정 확인
- [ ] 롤백 계획 수립
```

### 8.2 배포 후 체크리스트

```markdown
- [ ] 헬스 체크 엔드포인트 확인
- [ ] 주요 API 엔드포인트 테스트
- [ ] Admin 페이지 접근 확인
- [ ] 결제 기능 테스트 (테스트 결제)
- [ ] 소셜 로그인 테스트
- [ ] 이메일 발송 테스트
- [ ] Celery 작업 확인
- [ ] 에러 로그 확인
- [ ] 성능 모니터링 확인
- [ ] 사용자 피드백 모니터링
```

---

## 9. 성능 최적화

### 9.1 Nginx 설정

```nginx
# nginx.conf
upstream django {
    server web:8000;
}

server {
    listen 80;
    server_name yourdomain.com;

    # HTTPS로 리다이렉트
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    client_max_body_size 20M;

    # 정적 파일
    location /static/ {
        alias /static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 미디어 파일
    location /media/ {
        alias /media/;
        expires 7d;
    }

    # Django 애플리케이션
    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 120s;
        proxy_read_timeout 120s;
    }
}
```

### 9.2 Gunicorn 최적화

```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "sync"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 50
timeout = 120
keepalive = 5

# 로깅
accesslog = "-"
errorlog = "-"
loglevel = "info"
```

---

## 10. 보안 설정

### 10.1 Django 보안 설정 (production)

```python
# config/settings/production.py

# HTTPS 강제
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# HSTS (HTTP Strict Transport Security)
SECURE_HSTS_SECONDS = 31536000  # 1년
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# 쿠키 보안
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True

# X-Frame-Options
X_FRAME_OPTIONS = 'DENY'

# Content Security Policy
CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "'unsafe-inline'", "https://cdn.jsdelivr.net")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'", "https://fonts.googleapis.com")

# 기타 보안
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
```

### 10.2 방화벽 및 네트워크 보안

```bash
# UFW 방화벽 설정
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# PostgreSQL은 내부 네트워크만 허용
sudo ufw allow from 10.0.0.0/8 to any port 5432
```

---

## 11. 스케일링 전략

### 11.1 수평 확장

```
                    Load Balancer
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   Web Server 1    Web Server 2    Web Server 3
        │                │                │
        └────────────────┼────────────────┘
                         │
                    ┌────┴────┐
                    ▼         ▼
              PostgreSQL   Redis
              (Primary)    (Cache)
```

### 11.2 데이터베이스 읽기 복제본

```python
# config/settings/production.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT'),
    },
    'read_replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_READ_USER'),
        'PASSWORD': env('DB_READ_PASSWORD'),
        'HOST': env('DB_READ_HOST'),
        'PORT': env('DB_PORT'),
    }
}

DATABASE_ROUTERS = ['config.db_router.ReadWriteRouter']
```

---

## 12. 참고 자료

- [Django Deployment Checklist](https://docs.djangoproject.com/en/5.0/howto/deployment/checklist/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker Documentation](https://docs.docker.com/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [AWS Deployment Best Practices](https://aws.amazon.com/architecture/well-architected/)
