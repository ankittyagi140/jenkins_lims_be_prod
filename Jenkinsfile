pipeline {
    agent any

    environment {
        VM_HOST = '192.168.1.50'  // TODO: Update with production VM host
        DEPLOY_PATH = '/opt/lims'
        REPO_URL = 'https://github.com/ankittyagi140/euroasia_lims_be.git'
        API_GATEWAY_PORT = '8080'
        API_GATEWAY_HTTPS_PORT = '443'
        MAX_HEALTH_CHECK_RETRIES = '30'
        HEALTH_CHECK_INTERVAL = '5'
    }

    stages {

        /* =========================
           CHECKOUT
        ========================= */
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],  // TODO: Update branch if production uses different branch
                    userRemoteConfigs: [[
                        url: "${REPO_URL}",
                        credentialsId: 'github-pat'
                    ]]
                ])
            }
        }

        /* =========================
           CODE QUALITY CHECKS
        ========================= */
        stage('Code Quality Checks') {
            steps {
                script {
                    def changedFiles = sh(
                        script: "git diff --name-only HEAD~1 HEAD 2>/dev/null || git ls-files",
                        returnStdout: true
                    ).trim()

                    def pythonFilesChanged = changedFiles.split('\n').findAll {
                        it.endsWith('.py') && !it.contains('migrations/') && !it.contains('alembic/versions/')
                    }

                    if (pythonFilesChanged.size() > 0) {
                        echo "Checking Black formatting for ${pythonFilesChanged.size()} Python files..."

                        // Check if Black formatting is correct
                        def blackCheck = sh(
                            script: """
                                cd services/crm && python -m black --check --line-length=120 . 2>&1 || true
                                cd ../notification && python -m black --check --line-length=120 . 2>&1 || true
                                cd ../user && python -m black --check --line-length=120 . 2>&1 || true
                                cd ../inward && python -m black --check --line-length=120 . 2>&1 || true
                                cd ../test-packages && python -m black --check --line-length=120 . 2>&1 || true
                                cd ../sample-management && python -m black --check --line-length=120 . 2>&1 || true
                            """,
                            returnStatus: true
                        )

                        if (blackCheck != 0) {
                            echo "⚠️  Warning: Some files are not formatted with Black"
                            echo "Run: ./scripts/format-all-services.sh to fix"
                        } else {
                            echo "✅ All Python files are properly formatted with Black"
                        }
                    } else {
                        echo "No Python files changed, skipping Black check"
                    }
                }
            }
        }

        /* =========================
           DETECT CHANGED SERVICES
        ========================= */
        stage('Detect Changed Services') {
            steps {
                script {
                    def previousCommit = sh(
                        script: 'git rev-parse HEAD~1 2>/dev/null || echo "HEAD"',
                        returnStdout: true
                    ).trim()

                    def currentCommit = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    env.GIT_COMMIT_SHORT = currentCommit ? currentCommit.take(7) : 'manual'

                    def changedFiles = sh(
                        script: "git diff --name-only ${previousCommit}..${currentCommit}",
                        returnStdout: true
                    ).trim()

                    env.CRM_LIMS_CHANGED = 'false'
                    env.TEST_PACKAGES_CHANGED = 'false'
                    env.AUTH_SERVICE_CHANGED = 'false'
                    env.NOTIFICATION_SERVICE_CHANGED = 'false'
                    env.USER_SERVICE_CHANGED = 'false'
                    env.SAMPLE_MANAGEMENT_CHANGED = 'false'
                    env.API_GATEWAY_CHANGED = 'false'
                    env.REBUILD_ALL = 'false'

                    if (changedFiles.contains('services/common/') || changedFiles == '') {
                        env.CRM_LIMS_CHANGED = 'true'
                        env.TEST_PACKAGES_CHANGED = 'true'
                    }
                    if (changedFiles.contains('services/crm/') || changedFiles == '') {
                        env.CRM_LIMS_CHANGED = 'true'
                    }
                    if (changedFiles.contains('services/test-packages/') || changedFiles == '') {
                        env.TEST_PACKAGES_CHANGED = 'true'
                    }
                    if (changedFiles.contains('services/notification/') || changedFiles == '') {
                        env.NOTIFICATION_SERVICE_CHANGED = 'true'
                    }
                    if (changedFiles.contains('services/user/') || changedFiles == '') {
                        env.USER_SERVICE_CHANGED = 'true'
                    }
                    if (changedFiles.contains('services/sample-management/') || changedFiles == '') {
                        env.SAMPLE_MANAGEMENT_CHANGED = 'true'
                    }
                    if (changedFiles.contains('api-gateway/services/auth/') || changedFiles == '') {
                        env.AUTH_SERVICE_CHANGED = 'true'
                    }
                    if (changedFiles.contains('api-gateway/nginx/') ||
                        changedFiles.contains('api-gateway/docker-compose') ||
                        changedFiles == '') {
                        env.API_GATEWAY_CHANGED = 'true'
                    }
                    if (changedFiles.contains('Jenkinsfile.prod')) {
                        env.REBUILD_ALL = 'true'
                    }

                    def services = []
                    if (env.REBUILD_ALL == 'true' || env.CRM_LIMS_CHANGED == 'true') services.add('crm')
                    if (env.REBUILD_ALL == 'true' || env.TEST_PACKAGES_CHANGED == 'true') services.add('test-packages')
                    if (env.REBUILD_ALL == 'true' || env.AUTH_SERVICE_CHANGED == 'true') services.add('auth-service')
                    if (env.REBUILD_ALL == 'true' || env.NOTIFICATION_SERVICE_CHANGED == 'true') services.add('notification-service')
                    if (env.REBUILD_ALL == 'true' || env.USER_SERVICE_CHANGED == 'true') services.add('user-service')
                    if (env.REBUILD_ALL == 'true' || env.SAMPLE_MANAGEMENT_CHANGED == 'true') services.add('sample-management')
                    if (env.REBUILD_ALL == 'true' || env.API_GATEWAY_CHANGED == 'true') services.add('api-gateway')

                    env.SERVICES_TO_REBUILD = services.join(',')
                    echo "Services to rebuild: ${env.SERVICES_TO_REBUILD}"
                }
            }
        }

        /* =========================
           DEPLOY TO VM
        ========================= */
        stage('Deploy to VM') {
            steps {
                sshagent(credentials: ['lims-ssh-key']) {
                    script {
                        def gitCommitShort = env.GIT_COMMIT_SHORT ?: 'manual'
                        def scriptPart1 = """
                        ssh -o StrictHostKeyChecking=no euroasia-lims@\${VM_HOST} \\
                          "mkdir -p \${DEPLOY_PATH}"

                        echo "Checking if sample-management exists locally..."
                        if [ ! -d "services/sample-management" ]; then
                          echo "⚠️  WARNING: services/sample-management directory not found in source"
                          echo "This may be expected if the service hasn't been committed to git yet."
                          echo "The pipeline will continue, but Docker Compose may fail if the directory is required."
                        else
                          echo "✅ sample-management directory found locally"
                        fi

                        if command -v rsync >/dev/null 2>&1; then
                          echo "Syncing files with rsync..."
                          echo "Ensuring services/sample-management is explicitly included..."
                          rsync -avz --delete \\
                            --exclude='.git' \\
                            --exclude='node_modules' \\
                            --exclude='__pycache__' \\
                            --exclude='api-gateway/logs/' \\
                            --exclude='lims_upload/' \\
                            --exclude='pgdata_stage/' \\
                            --exclude='pgdata_stage/**' \\
                            --exclude='*.log' \\
                            --include='services/sample-management/' \\
                            --include='services/sample-management/**' \\
                            -e "ssh -o StrictHostKeyChecking=no" \\
                            ./ euroasia-lims@\${VM_HOST}:\${DEPLOY_PATH}/
                          
                          echo "Verifying sample-management directory was synced..."
                          if ! ssh -o StrictHostKeyChecking=no euroasia-lims@\${VM_HOST} "test -d \${DEPLOY_PATH}/services/sample-management"; then
                            echo "⚠️  WARNING: sample-management directory not found after rsync"
                            echo "This may be expected if the service hasn't been committed to git yet."
                            echo "Listing remote services directory:"
                            ssh -o StrictHostKeyChecking=no euroasia-lims@\${VM_HOST} "ls -la \${DEPLOY_PATH}/services/ || echo 'services directory does not exist'"
                            echo "Docker Compose will fail if this directory is required for the build."
                          else
                            echo "✅ sample-management directory verified"
                          fi
                        else
                          echo "rsync not found, falling back to tar over ssh"
                          tar --exclude='.git' --exclude='node_modules' --exclude='__pycache__' \\
                              --exclude='api-gateway/logs' --exclude='*.log' -czf - . | \\
                            ssh -o StrictHostKeyChecking=no euroasia-lims@\${VM_HOST} \\
                              "mkdir -p \${DEPLOY_PATH} && tar -xzf - -C \${DEPLOY_PATH}"
                        fi
                        """
                        def sshCmd = "ssh -o StrictHostKeyChecking=no euroasia-lims@\${VM_HOST}"
                        def exportCmd = "export DEPLOY_PATH='\${DEPLOY_PATH}' && export GIT_COMMIT_SHORT='${gitCommitShort}' && /bin/bash -s"
                        def sshCommandLine = "${sshCmd} \"${exportCmd}\" << 'SCRIPT_EOF'\n"
                        def bashScriptContent = '''#!/bin/bash
set -e
cd "${DEPLOY_PATH}/api-gateway" || { echo "Failed to cd to ${DEPLOY_PATH}/api-gateway"; exit 1; }

ENV_TEMPLATE="env.prod.template"
COMPOSE_FILE="docker-compose.prod.yml"
PG_CONTAINER="lims-postgres-prod"
CLEANUP_CONTAINERS="lims-auth-service-prod lims-crm-prod lims-test-packages-prod lims-api-gateway-prod lims-postgres-prod lims-inward-prod lims-notification-prod lims-rabbitmq-prod lims-user-prod lims-sample-management-prod LIMS-API-Gateway-PROD"

if [ ! -f .env ]; then
  echo "Creating .env from ${ENV_TEMPLATE}..."
  if [ ! -f "${ENV_TEMPLATE}" ]; then
    echo "❌ ERROR: ${ENV_TEMPLATE} not found"
    exit 1
  fi
  cp "${ENV_TEMPLATE}" .env
  echo "⚠️  WARNING: .env file created from template. Please verify all production secrets are set!"
fi

# Handle renamed CRM folder (crm-lims -> crm) on the VM
if [ ! -d "${DEPLOY_PATH}/services/crm" ] && [ -d "${DEPLOY_PATH}/services/crm-lims" ]; then
  mv "${DEPLOY_PATH}/services/crm-lims" "${DEPLOY_PATH}/services/crm"
fi

# Handle renamed Inward folder (invards -> inward) on the VM
if [ ! -d "${DEPLOY_PATH}/services/inward" ] && [ -d "${DEPLOY_PATH}/services/invards" ]; then
  mv "${DEPLOY_PATH}/services/invards" "${DEPLOY_PATH}/services/inward"
fi

export $(grep -v '^#' .env | xargs)
PGUSER="${POSTGRES_USER:-lims_user}"
PGPASSWORD="${POSTGRES_PASSWORD}"
LIMS_DB_NAME="${POSTGRES_DB:-lims_db}"
export PGPASSWORD

if [ -z "${PGPASSWORD}" ]; then
  echo "❌ ERROR: POSTGRES_PASSWORD is not set in .env file"
  echo "This is required for production deployment!"
  exit 1
fi

echo "⚠️  PRODUCTION DEPLOYMENT - Stopping existing containers..."
docker-compose -f "${COMPOSE_FILE}" down --remove-orphans || true
for c in ${CLEANUP_CONTAINERS}; do
  docker rm -f "$c" >/dev/null 2>&1 || true
done

echo "Starting PostgreSQL for production..."
if ! docker-compose -f "${COMPOSE_FILE}" up -d postgres; then
  echo "❌ Failed to start PostgreSQL"
  exit 1
fi
sleep 15

echo "Waiting for PostgreSQL to be ready..."
MAX_WAIT=90
WAIT_COUNT=0
until docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" pg_isready -U "${PGUSER}" >/dev/null 2>&1; do
    WAIT_COUNT=$((WAIT_COUNT + 2))
    if [ ${WAIT_COUNT} -ge ${MAX_WAIT} ]; then
      echo "❌ PostgreSQL failed to become ready within ${MAX_WAIT} seconds"
      exit 1
    fi
    echo "Waiting for postgres... (${WAIT_COUNT}s/${MAX_WAIT}s)"
    sleep 2
done
echo "✅ PostgreSQL is ready"

echo "Creating required databases..."
if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d postgres -tc "SELECT 1 FROM pg_database WHERE datname = '${LIMS_DB_NAME}'" 2>/dev/null | grep -q 1; then
  echo "Creating database ${LIMS_DB_NAME}..."
  if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d postgres -c "CREATE DATABASE ${LIMS_DB_NAME};" 2>/dev/null; then
    echo "❌ Failed to create database ${LIMS_DB_NAME}"
    exit 1
  fi
  docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d postgres -c "GRANT ALL PRIVILEGES ON DATABASE ${LIMS_DB_NAME} TO ${PGUSER};" 2>/dev/null || true
  echo "✅ Database ${LIMS_DB_NAME} created"
else
  echo "✅ Database ${LIMS_DB_NAME} already exists"
fi

if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d postgres -tc "SELECT 1 FROM pg_database WHERE datname = 'crm_lims'" 2>/dev/null | grep -q 1; then
  echo "Creating database crm_lims..."
  if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d postgres -c "CREATE DATABASE crm_lims;" 2>/dev/null; then
    echo "❌ Failed to create database crm_lims"
    exit 1
  fi
  docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d postgres -c "GRANT ALL PRIVILEGES ON DATABASE crm_lims TO ${PGUSER};" 2>/dev/null || true
  echo "✅ Database crm_lims created"
else
  echo "✅ Database crm_lims already exists"
fi

echo "Creating required schemas..."
if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d "${LIMS_DB_NAME}" -tc "SELECT 1 FROM information_schema.schemata WHERE schema_name = 'lims'" 2>/dev/null | grep -q 1; then
  echo "Creating schema lims..."
  if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d "${LIMS_DB_NAME}" -c "CREATE SCHEMA IF NOT EXISTS lims;" 2>/dev/null; then
    echo "❌ Failed to create schema lims"
    exit 1
  fi
  docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d "${LIMS_DB_NAME}" -c "GRANT ALL ON SCHEMA lims TO ${PGUSER};" 2>/dev/null || true
  echo "✅ Schema lims created"
else
  echo "✅ Schema lims already exists"
fi

if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d crm_lims -tc "SELECT 1 FROM information_schema.schemata WHERE schema_name = 'crm'" 2>/dev/null | grep -q 1; then
  echo "Creating schema crm..."
  if ! docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d crm_lims -c "CREATE SCHEMA IF NOT EXISTS crm;" 2>/dev/null; then
    echo "❌ Failed to create schema crm"
    exit 1
  fi
  docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d crm_lims -c "GRANT ALL ON SCHEMA crm TO ${PGUSER};" 2>/dev/null || true
  echo "✅ Schema crm created"
else
  echo "✅ Schema crm already exists"
fi

echo "Creating backup before deployment..."
BACKUP_DIR="${POSTGRES_BACKUP_PATH:-/opt/lims/backups}"
mkdir -p "${BACKUP_DIR}" || true
if [ -z "${GIT_COMMIT_SHORT}" ]; then
  GIT_COMMIT_SHORT="manual"
fi
BACKUP_FILE="${BACKUP_DIR}/pre-deploy-$(date +%Y%m%d-%H%M%S)-${GIT_COMMIT_SHORT}.sql"
if docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" pg_dump -U "${PGUSER}" "${LIMS_DB_NAME}" > "${BACKUP_FILE}" 2>/dev/null; then
  echo "✅ Database backup created: ${BACKUP_FILE}"
  # Keep only last 20 backups for production (more than stage)
  ls -t "${BACKUP_DIR}"/pre-deploy-*.sql 2>/dev/null | tail -n +21 | xargs rm -f 2>/dev/null || true
  echo "✅ Backup cleanup completed (keeping last 20 backups)"
else
  echo "⚠️  Warning: Database backup failed, but continuing deployment"
fi

echo "Bootstrapping CRM database..."
if ! docker-compose -f "${COMPOSE_FILE}" run --rm --no-deps --build crm python scripts/bootstrap_db.py; then
  echo "❌ CRM database bootstrap failed"
  exit 1
fi
echo "✅ CRM database bootstrap completed"

echo "Verifying database migrations..."
TABLE_COUNT=$(docker exec -e PGPASSWORD="${PGPASSWORD}" "${PG_CONTAINER}" psql -U "${PGUSER}" -d crm_lims -tc "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'crm'" 2>/dev/null | tr -d ' ' || echo "0")
if [ "${TABLE_COUNT}" -gt "0" ]; then
  echo "✅ CRM schema verified: ${TABLE_COUNT} tables found"
else
  echo "❌ CRM schema verification failed - no tables found"
  exit 1
fi

echo "Verifying service directories exist..."
REQUIRED_DIRS=(
  "\${DEPLOY_PATH}/services/crm"
  "\${DEPLOY_PATH}/services/test-packages"
  "\${DEPLOY_PATH}/services/notification"
  "\${DEPLOY_PATH}/services/user"
  "\${DEPLOY_PATH}/services/inward"
  "\${DEPLOY_PATH}/services/sample-management"
)
for dir in "\${REQUIRED_DIRS[@]}"; do
  if [ ! -d "\${dir}" ]; then
    echo "❌ ERROR: Required directory does not exist: \${dir}"
    echo "This may indicate a sync issue. Please check rsync logs."
    exit 1
  fi
  echo "✅ Directory exists: \${dir}"
done

echo "⚠️  PRODUCTION DEPLOYMENT - Starting all services..."
if ! docker-compose -f "${COMPOSE_FILE}" up -d --build; then
  echo "❌ Failed to start services"
  exit 1
fi
echo "✅ All services started successfully"

echo "Initializing RabbitMQ vhost..."
RABBITMQ_CONTAINER="lims-rabbitmq-prod"

# Wait for RabbitMQ to be ready
echo "Waiting for RabbitMQ to be ready..."
MAX_WAIT=90
WAIT_COUNT=0
until docker exec "${RABBITMQ_CONTAINER}" rabbitmqctl ping > /dev/null 2>&1; do
  WAIT_COUNT=$((WAIT_COUNT + 2))
  if [ ${WAIT_COUNT} -ge ${MAX_WAIT} ]; then
    echo "❌ RabbitMQ failed to become ready within ${MAX_WAIT} seconds"
    exit 1
  fi
  echo "Waiting for RabbitMQ... (${WAIT_COUNT}s/${MAX_WAIT}s)"
  sleep 2
done
echo "✅ RabbitMQ is ready"

# Create vhost and set permissions
echo "Creating RabbitMQ vhost 'lims'..."
docker exec "${RABBITMQ_CONTAINER}" rabbitmqctl add_vhost lims 2>/dev/null || echo "Vhost 'lims' may already exist"
docker exec "${RABBITMQ_CONTAINER}" rabbitmqctl set_permissions -p lims lims_user ".*" ".*" ".*" 2>/dev/null || echo "Permissions may already be set"
echo "✅ RabbitMQ vhost initialized"
SCRIPT_EOF'''
                        def scriptPart2 = sshCommandLine + bashScriptContent
                        def deployScript = scriptPart1 + scriptPart2
                        sh deployScript
                    }
                }
            }
        }

        /* =========================
           HEALTH CHECKS
        ========================= */
        stage('Health Checks') {
            steps {
                script {
                    def retries = 0
                    while (retries < env.MAX_HEALTH_CHECK_RETRIES.toInteger()) {
                        // Try HTTPS first, fallback to HTTP
                        def status = sh(
                            script: "curl -sfk https://${VM_HOST}:${API_GATEWAY_HTTPS_PORT}/health || curl -sf http://${VM_HOST}:${API_GATEWAY_PORT}/health",
                            returnStatus: true
                        )
                        if (status == 0) {
                            echo "✅ API Gateway healthy"
                            return
                        }
                        retries++
                        echo "Health check attempt ${retries}/${env.MAX_HEALTH_CHECK_RETRIES} failed, retrying..."
                        sleep env.HEALTH_CHECK_INTERVAL.toInteger()
                    }
                    error "❌ Health checks failed after ${env.MAX_HEALTH_CHECK_RETRIES} attempts"
                }
            }
        }

        /* =========================
           CONTAINER STATUS
        ========================= */
        stage('Container Status') {
            steps {
                sshagent(credentials: ['lims-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no euroasia-lims@\${VM_HOST} \\
                          "export DEPLOY_PATH='\${DEPLOY_PATH}' && /bin/bash -s" << 'STATUS_EOF'
cd "\${DEPLOY_PATH}/api-gateway" && docker-compose -f docker-compose.prod.yml ps
STATUS_EOF
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Production deployment completed successfully'
            // TODO: Add notification (email, Slack, etc.) on success
        }
        failure {
            echo '❌ Production deployment failed'
            // TODO: Add notification (email, Slack, etc.) on failure
        }
        always {
            echo 'Pipeline finished'
        }
    }
}

