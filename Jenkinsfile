pipeline {
    agent any

    environment {
        // ===== PRODUCTION CONFIG =====
        DEPLOY_PATH = '/opt/lims'
        REPO_URL    = 'https://github.com/ankittyagi140/euroasia_lims_be.git'

        ENV_TEMPLATE = 'env.prod.template'
        COMPOSE_FILE = 'docker-compose.prod.yml'

        API_GATEWAY_PORT = '8080'
        MAX_HEALTH_CHECK_RETRIES = '30'
        HEALTH_CHECK_INTERVAL = '5'
    }

    stages {

        /* =========================
           CHECKOUT
        ========================= */
        stage('Checkout') {
            steps {
                git branch: 'release',
                    credentialsId: 'github-pat',
                    url: env.REPO_URL
            }
        }

        /* =========================
           CODE QUALITY CHECKS
        ========================= */
        stage('Code Quality Checks') {
            steps {
                echo 'Skipping Black checks in Jenkins (handled in Docker / dev workflow)'
            }
        }

        /* =========================
           DEPLOY (LOCAL, DOCKER COMPOSE V2)
        ========================= */
        stage('Deploy') {
            steps {
                sh """
                    set -e

                    echo "📁 Preparing deploy directory"

                    if [ -z "${DEPLOY_PATH}" ] || [ "${DEPLOY_PATH}" = "/" ]; then
                      echo "❌ Unsafe DEPLOY_PATH"
                      exit 1
                    fi

                    mkdir -p "${DEPLOY_PATH}"
                    rm -rf "${DEPLOY_PATH}/"*

                    echo "📦 Copying source code (tar)"
                    tar --exclude='.git' \
                        --exclude='node_modules' \
                        --exclude='__pycache__' \
                        --exclude='*.log' \
                        -czf - . | tar -xzf - -C "${DEPLOY_PATH}"

                    cd "${DEPLOY_PATH}/api-gateway" || {
                      echo "❌ api-gateway directory not found"
                      exit 1
                    }

                    if [ ! -f ".env" ]; then
                      echo "Creating .env from ${ENV_TEMPLATE}"
                      cp "${ENV_TEMPLATE}" .env
                    fi

                    # Ensure pgAdmin port doesn't conflict with existing services on the host.
                    # `.env` may persist across deployments; force PGADMIN_PORT to 5052.
                    if grep -q '^PGADMIN_PORT=' .env; then
                      sed -i 's/^PGADMIN_PORT=.*/PGADMIN_PORT=5052/' .env
                    else
                      echo "PGADMIN_PORT=5052" >> .env
                    fi

                    export \$(grep -v '^#' .env | xargs)

                    echo "♻️ Restarting services (docker compose v2)"
                    # Force remove old containers with same names if they exist
                    docker compose -p euroasia-lims-prod -f "${COMPOSE_FILE}" down --remove-orphans || true
                    # Remove any containers with conflicting names (from old deployments)
                    docker rm -f lims-auth-service-prod lims-crm-prod lims-inward-prod lims-notification-prod lims-user-prod lims-sample-management-prod lims-test-management-prod lims-test-packages-prod lims-efiling-prod lims-rabbitmq-prod lims-postgres lims-pgadmin-prod LIMS-API-Gateway-PROD lims-minio-prod 2>/dev/null || true

                    # Host is missing Docker buildx; parallel classic builds can fail with
                    # "No such image: sha256:..." on intermediate layers (seen on api-gateway).
                    export DOCKER_BUILDKIT=0
                    export COMPOSE_DOCKER_CLI_BUILD=0
                    echo "🔨 Building images (serial, classic builder)"
                    if ! docker compose -p euroasia-lims-prod -f "${COMPOSE_FILE}" build --parallel 1; then
                      echo "⚠️ Image build failed once; retrying serial build"
                      docker compose -p euroasia-lims-prod -f "${COMPOSE_FILE}" build --parallel 1
                    fi
                    docker compose -p euroasia-lims-prod -f "${COMPOSE_FILE}" up -d --no-build

                    echo "✅ Services deployed successfully"
                """
            }
        }

        /* =========================
           HEALTH CHECKS
        ========================= */
        stage('Health Checks') {
            steps {
                script {
                    int retries = 0
                    def gatewayHealthy = false
                    def servicesHealthy = false
                    
                    while (retries < env.MAX_HEALTH_CHECK_RETRIES.toInteger()) {
                        // Check API Gateway health
                        def gatewayStatus = sh(
                            script: "curl -sf http://localhost:${API_GATEWAY_PORT}/health",
                            returnStatus: true
                        )
                        
                        if (gatewayStatus == 0) {
                            gatewayHealthy = true
                            echo "✅ API Gateway healthy"
                            
                            // Check actual API service endpoints
                            def crmStatus = sh(
                                script: "curl -sf http://localhost:${API_GATEWAY_PORT}/crm/health",
                                returnStatus: true
                            )
                            def authStatus = sh(
                                script: "curl -sf http://localhost:${API_GATEWAY_PORT}/auth/health",
                                returnStatus: true
                            )
                            def notificationStatus = sh(
                                script: "curl -sf http://localhost:${API_GATEWAY_PORT}/notification/health",
                                returnStatus: true
                            )
                            def efilingStatus = sh(
                                script: "curl -sf http://localhost:${API_GATEWAY_PORT}/efiling/health",
                                returnStatus: true
                            )
                            
                            if (crmStatus == 0 && authStatus == 0 && notificationStatus == 0 && efilingStatus == 0) {
                                servicesHealthy = true
                                echo "✅ All API services healthy (CRM, Auth, Notification, eFiling)"
                                return
                            } else {
                                echo "⚠️ Gateway healthy but some services not ready yet (retry ${retries + 1}/${env.MAX_HEALTH_CHECK_RETRIES})"
                            }
                        } else {
                            echo "⚠️ API Gateway not ready yet (retry ${retries + 1}/${env.MAX_HEALTH_CHECK_RETRIES})"
                        }
                        
                        retries++
                        sleep env.HEALTH_CHECK_INTERVAL.toInteger()
                    }
                    
                    if (!gatewayHealthy) {
                        error "❌ API Gateway health check failed after ${env.MAX_HEALTH_CHECK_RETRIES} retries"
                    } else if (!servicesHealthy) {
                        error "❌ API services health check failed after ${env.MAX_HEALTH_CHECK_RETRIES} retries"
                    } else {
                        error "❌ Health checks failed"
                    }
                }
            }
        }

        /* =========================
           CONTAINER STATUS
        ========================= */
        stage('Container Status') {
            steps {
                sh """
                    cd "${DEPLOY_PATH}/api-gateway"
                    docker compose -p euroasia-lims-prod -f "${COMPOSE_FILE}" ps
                """
            }
        }
    }

    post {
        success {
            echo '✅ Production deployment completed successfully'
        }
        failure {
            echo '❌ Production deployment failed'
        }
        always {
            echo 'Pipeline finished'
        }
    }
}
