version: "3.8"

services:
  api-gateway:
    extends:
      file: docker-compose.yml
      service: api-gateway
    container_name: lims-api-gateway-local
    ports:
      - "3001:80"
    environment:
      - ENV=local
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      auth-service:
        condition: service_healthy
      crm:
        condition: service_healthy
      test-packages:
        condition: service_healthy
      inward:
        condition: service_healthy
      notification-service:
        condition: service_healthy
      user-service:
        condition: service_healthy
      sample-management:
        condition: service_healthy
      test-management:
        condition: service_healthy
      efiling:
        condition: service_healthy

  postgres:
    extends:
      file: docker-compose.yml
      service: postgres
    container_name: lims-postgres-local
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-lims_db_local}
      - POSTGRES_USER=${POSTGRES_USER:-lims_user}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-lims_password_local}
    ports:
      - "${POSTGRES_PORT:-5434}:5432"
    volumes:
      - postgres_data_local:/var/lib/postgresql/data
      - ./init-db:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-lims_user} -d ${POSTGRES_DB:-lims_db_local} && psql -U ${POSTGRES_USER:-lims_user} -d ${POSTGRES_DB:-lims_db_local} -tAc 'SELECT 1' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 60s

  # CRM Service
  crm:
    build:
      context: ../services/crm
      dockerfile: Dockerfile
    container_name: lims-crm-local
    expose:
      - "8004"
    environment:
      - SERVICE_NAME=crm
      - SERVICE_VERSION=1.0.0
      - CRM_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/crm_lims
      - LIMS_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/${POSTGRES_DB:-lims_db_local}
      - RABBITMQ_URL=${RABBITMQ_URL:-amqp://${RABBITMQ_DEFAULT_USER:-lims_user}:${RABBITMQ_DEFAULT_PASS:-lims_password}@rabbitmq:5672/}
      - RABBITMQ_EXCHANGE_NAME=${RABBITMQ_EXCHANGE_NAME:-lims_notifications_exchange}
      - RABBITMQ_ROUTING_KEY_PREFIX=${RABBITMQ_ROUTING_KEY_PREFIX:-lims_notification}
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_DEFAULT_USER:-lims_user}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_DEFAULT_PASS:-lims_password}
      # MinIO Configuration
      - MINIO_ENDPOINT=${MINIO_ENDPOINT:-minio:9000}
      - MINIO_PUBLIC_ENDPOINT=${MINIO_PUBLIC_ENDPOINT:-localhost:9090}
      - MINIO_ACCESS_KEY=${MINIO_ROOT_USER:-minioadmin}
      - MINIO_SECRET_KEY=${MINIO_ROOT_PASSWORD:-minioadmin123}
      - MINIO_BUCKET=${MINIO_BUCKET:-lims-uploads}
      - USE_MINIO=${USE_MINIO:-true}
      - UPLOAD_DIR=${UPLOAD_DIR:-uploads}
    volumes:
      - ${UPLOADS_HOST_PATH:-./uploads}:/data/uploads
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      minio:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8004/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # Authentication Service
  auth-service:
    build:
      context: ./services/auth
      dockerfile: Dockerfile
    container_name: lims-auth-service-local
    expose:
      - "8006"
    environment:
      - AZURE_AD_TENANT_ID=${AZURE_AD_TENANT_ID:-29073e22-b2a3-4fce-8513-092b4718ac62}
      - API_SCOPE_DEV=${API_SCOPE_DEV:-api://api.dev.euroasiasci.com/access_api}
      - API_SCOPE_PROD=${API_SCOPE_PROD:-api://api.euroasiasci.com/access_api}
      - ENV=local
    restart: unless-stopped
    networks:
      - gateway-net
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8006/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # Test Packages Service
  test-packages:
    build:
      context: ../services/test-packages
      dockerfile: Dockerfile
    container_name: lims-test-packages-local
    expose:
      - "8005"
    environment:
      - SERVICE_NAME=test-packages
      - SERVICE_VERSION=1.0.0
      - LIMS_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/${POSTGRES_DB:-lims_db_local}
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8005/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # Inward Service
  inward:
    build:
      context: ../services/inward
      dockerfile: Dockerfile
    container_name: lims-inward-local
    expose:
      - "8007"
    environment:
      - SERVICE_NAME=inward
      - SERVICE_VERSION=1.0.0
      - LIMS_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/${POSTGRES_DB:-lims_db_local}
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8007/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # Notification Service
  notification-service:
    build:
      context: ../services/notification
      dockerfile: Dockerfile
    container_name: lims-notification-local
    expose:
      - "8000"
    environment:
      - SERVICE_NAME=notification-service
      - SERVICE_VERSION=1.0.0
      - NOTIFICATION_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/crm_lims
      # Use default vhost "/" (same as crm); "/lims" requires creating that vhost in RabbitMQ first
      - RABBITMQ_URL=${RABBITMQ_URL:-amqp://${RABBITMQ_DEFAULT_USER:-lims_user}:${RABBITMQ_DEFAULT_PASS:-lims_password}@rabbitmq:5672/}
      - RABBITMQ_EXCHANGE_NAME=${RABBITMQ_EXCHANGE_NAME:-lims_notifications_exchange}
      - RABBITMQ_QUEUE_NAME=${RABBITMQ_QUEUE_NAME:-lims_notification_queue}
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8000/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: lims-rabbitmq-local
    expose:
      - "5672"
      - "15672"
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_DEFAULT_USER:-lims_user}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_DEFAULT_PASS:-lims_password}
    restart: unless-stopped
    networks:
      - gateway-net
    healthcheck:
      test: [ "CMD", "rabbitmq-diagnostics", "ping" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # MailHog (for email testing)
  mailhog:
    image: mailhog/mailhog:latest
    container_name: lims-mailhog-local
    expose:
      - "1025"
      - "8025"
    restart: unless-stopped
    networks:
      - gateway-net

  # User Service
  user-service:
    build:
      context: ../services/user
      dockerfile: Dockerfile
    container_name: lims-user-local
    expose:
      - "8008"
    environment:
      - SERVICE_NAME=user-service
      - SERVICE_VERSION=1.0.0
      - USER_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/crm_lims
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8008/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # Sample Management Service
  sample-management:
    build:
      context: ../services/sample-management
      dockerfile: Dockerfile
    container_name: lims-sample-management-local
    expose:
      - "8009"
    environment:
      - SERVICE_NAME=sample-management-service
      - SERVICE_VERSION=1.0.0
      - LIMS_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/${POSTGRES_DB:-lims_db_local}
      - INWARD_SERVICE_URL=http://inward:8007
      - TEST_MANAGEMENT_SERVICE_URL=http://test-management:8010
      - UVICORN_WORKERS=2
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8009/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # CPSC eFiling Service
  efiling:
    build:
      context: ../services/efiling
      dockerfile: Dockerfile
    container_name: lims-efiling-local
    expose:
      - "8011"
    environment:
      - SERVICE_NAME=efiling
      - SERVICE_VERSION=1.0.0
      - LIMS_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/${POSTGRES_DB:-lims_db_local}
      - CRM_SERVICE_URL=http://crm:8004/api/v1
      - CPSC_EFILING_BASE_URL=${CPSC_EFILING_BASE_URL:-https://efiling.saferproducts.gov/efiling/api}
      - EFILING_MOCK_MODE=${EFILING_MOCK_MODE:-false}
      - EFILING_ENCRYPTION_KEY=${EFILING_ENCRYPTION_KEY:-lims-efiling-local-key}
      - CPSC_DEFAULT_LAB_ALTERNATE_ID=${CPSC_DEFAULT_LAB_ALTERNATE_ID:-darshan.chaudhary@euroasiasci.com}
      - CPSC_USE_BEARER_AUTH=${CPSC_USE_BEARER_AUTH:-false}
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
      crm:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8011/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # Test Management Service
  test-management:
    build:
      context: ../services/test-management
      dockerfile: Dockerfile
    container_name: lims-test-management-local
    expose:
      - "8010"
    environment:
      - SERVICE_NAME=test-management-service
      - SERVICE_VERSION=1.0.0
      - LIMS_DATABASE_URL=postgresql://${POSTGRES_USER:-lims_user}:${POSTGRES_PASSWORD:-lims_password_local}@postgres:5432/${POSTGRES_DB:-lims_db_local}
      - SAMPLE_MANAGEMENT_SERVICE_URL=http://sample-management:8009
      - TEST_PACKAGES_SERVICE_URL=http://test-packages:8005
      - UVICORN_WORKERS=2
    restart: unless-stopped
    networks:
      - gateway-net
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "python", "-c", "import requests; r=requests.get('http://localhost:8010/health', timeout=5); exit(0 if r.status_code==200 else 1)" ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # MinIO - Object Storage for File Uploads (Local Development)
  # IMPORTANT: Files are stored on host (local path), NOT in Docker container
  # Volume mount: ${UPLOADS_HOST_PATH} (host path) -> /data (container path)
  minio:
    image: minio/minio:latest
    container_name: lims-minio-local
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER:-minioadmin}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:-minioadmin123}
    ports:
      - "${MINIO_API_PORT:-9090}:9000"  # S3 API (mapped to 9090 to avoid conflicts)
      - "${MINIO_CONSOLE_PORT:-9091}:9001"  # Web Console (mapped to 9091)
    volumes:
      # Bind mount: Host path -> Container path (files stored on host, not in container)
      - ${UPLOADS_HOST_PATH:-./uploads}:/data
    networks:
      - gateway-net
    healthcheck:
      test: ["CMD", "sh", "-c", "curl -f http://localhost:9000/minio/health/live || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    restart: unless-stopped

volumes:
  postgres_data_local:
    driver: local

networks:
  gateway-net:
    driver: bridge
