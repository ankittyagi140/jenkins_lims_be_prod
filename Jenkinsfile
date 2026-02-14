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

                    export \$(grep -v '^#' .env | xargs)

                    echo "♻️ Restarting services (docker compose v2)"
                    docker compose -p euroasia-lims-prod -f "${COMPOSE_FILE}" down --remove-orphans || true
                    docker compose -p euroasia-lims-prod -f "${COMPOSE_FILE}" up -d --build

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
                    while (retries < env.MAX_HEALTH_CHECK_RETRIES.toInteger()) {
                        def status = sh(
                            script: "curl -sf http://localhost:${API_GATEWAY_PORT}/health",
                            returnStatus: true
                        )
                        if (status == 0) {
                            echo "✅ API Gateway healthy"
                            return
                        }
                        retries++
                        sleep env.HEALTH_CHECK_INTERVAL.toInteger()
                    }
                    error "❌ Health checks failed"
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
