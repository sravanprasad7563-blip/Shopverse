pipeline {
    agent any

    environment {
        BACKEND_IMAGE  = "shopverse-backend"
        FRONTEND_IMAGE = "shopverse-frontend"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        DEPLOY_DIR     = "/opt/shopverse"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                deleteDir()

                git branch: 'main',
                    url: 'https://github.com/sravanprasad7563-blip/Shopverse.git'
            }
        }

        stage('Backend Test') {
            steps {
                dir('backend') {
                    sh '''
                        set -eu

                        echo "===== BACKEND TEST ====="

                        go mod download
                        go test ./...
                        go vet ./...

                        echo "Backend tests PASSED"
                    '''
                }
            }
        }

        stage('Frontend Test & Build') {
            steps {
                dir('frontend') {
                    sh '''
                        set -eu

                        echo "===== FRONTEND TEST ====="

                        npm ci
                        npm run lint
                        npm run build

                        echo "Frontend build PASSED"
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu

                    echo "===== BUILD BACKEND IMAGE ====="

                    docker build \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${BACKEND_IMAGE}:latest \
                        ./backend

                    echo "===== BUILD FRONTEND IMAGE ====="

                    docker build \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${FRONTEND_IMAGE}:latest \
                        ./frontend

                    echo "===== DOCKER IMAGES ====="

                    docker images | grep shopverse
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    set -eu

                    mkdir -p trivy-reports

                    echo "===== BACKEND SECURITY SCAN ====="

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        ${BACKEND_IMAGE}:${IMAGE_TAG}

                    echo "===== FRONTEND SECURITY SCAN ====="

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG}

                    echo "Security scans completed"
                '''
            }
        }

        stage('Prepare Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "===== CHECK DEPLOYMENT DIRECTORY ====="

                    test -d ${DEPLOY_DIR}
                    test -f ${DEPLOY_DIR}/docker-compose.yml
                    test -f ${DEPLOY_DIR}/.env
                    test -f ${DEPLOY_DIR}/nginx/nginx.conf

                    echo "Deployment files found"

                    cd ${DEPLOY_DIR}

                    echo "===== COMPOSE VALIDATION ====="

                    export IMAGE_TAG=${IMAGE_TAG}

                    docker compose \
                        --env-file .env \
                        config > /tmp/shopverse-compose-${BUILD_NUMBER}.yml

                    echo "Compose validation PASSED"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -eu

                    cd ${DEPLOY_DIR}

                    echo "=========================================="
                    echo "DEPLOYING SHOPVERSE"
                    echo "IMAGE TAG: ${IMAGE_TAG}"
                    echo "=========================================="

                    export IMAGE_TAG=${IMAGE_TAG}

                    echo "===== START MYSQL ====="

                    docker compose \
                        --env-file .env \
                        up -d mysql

                    echo "Waiting for MySQL..."

                    for i in $(seq 1 30); do

                        STATUS=$(docker inspect \
                            --format='{{if .State.Health}}{{.State.Health.Status}}{{else}}starting{{end}}' \
                            shopverse-mysql 2>/dev/null || echo "starting")

                        echo "MySQL status: ${STATUS}"

                        if [ "${STATUS}" = "healthy" ]; then
                            echo "MySQL is healthy"
                            break
                        fi

                        if [ "$i" -eq 30 ]; then
                            echo "MySQL failed to become healthy"
                            docker compose logs mysql
                            exit 1
                        fi

                        sleep 5
                    done

                    echo "===== START APPLICATION ====="

                    docker compose \
                        --env-file .env \
                        up -d backend frontend nginx

                    echo "===== CURRENT CONTAINERS ====="

                    docker compose \
                        --env-file .env \
                        ps

                    echo "Deployment containers started"
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    set -eu

                    cd ${DEPLOY_DIR}

                    echo "=========================================="
                    echo "APPLICATION HEALTH CHECK"
                    echo "=========================================="

                    echo "Waiting for application..."

                    for i in $(seq 1 30); do

                        if curl -fsS http://127.0.0.1/health >/dev/null 2>&1; then
                            echo "Application health check PASSED"
                            break
                        fi

                        echo "Health check attempt ${i}/30 failed"

                        if [ "$i" -eq 30 ]; then
                            echo "Application health check FAILED"

                            echo "===== DOCKER STATUS ====="
                            docker compose ps

                            echo "===== NGINX LOGS ====="
                            docker compose logs --tail=100 nginx || true

                            echo "===== BACKEND LOGS ====="
                            docker compose logs --tail=100 backend || true

                            echo "===== MYSQL LOGS ====="
                            docker compose logs --tail=100 mysql || true

                            exit 1
                        fi

                        sleep 5
                    done

                    echo "===== FINAL STATUS ====="

                    docker compose ps

                    echo "Shopverse deployment successful"
                '''
            }
        }
    }

    post {

        success {
            echo '''
            ==========================================
            SHOPVERSE CI/CD SUCCESS
            ==========================================
            '''
        }

        failure {
            echo '''
            ==========================================
            SHOPVERSE CI/CD FAILED
            ==========================================
            Check the failed stage and Docker logs.
            ==========================================
            '''
        }

        always {
            archiveArtifacts(
                artifacts: 'trivy-reports/**/*',
                allowEmptyArchive: true
            )
        }
    }
}
