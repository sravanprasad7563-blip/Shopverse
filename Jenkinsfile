pipeline {
    agent any

    environment {
        REPO_URL       = 'https://github.com/sravanprasad7563-blip/Shopverse.git'
        REPO_BRANCH    = 'main'

        BACKEND_IMAGE  = 'shopverse-backend'
        FRONTEND_IMAGE = 'shopverse-frontend'
        IMAGE_TAG      = "${BUILD_NUMBER}"

        DEPLOY_DIR     = '/opt/shopverse'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                git branch: "${REPO_BRANCH}",
                    url: "${REPO_URL}"
            }
        }

        stage('Validate Tools') {
            steps {
                sh '''
                    set -eu

                    echo "============================================"
                    echo "VALIDATING JENKINS TOOLS"
                    echo "============================================"

                    echo "Go:"
                    go version

                    echo "Node:"
                    node -v

                    echo "NPM:"
                    npm -v

                    echo "Git:"
                    git --version

                    echo "Docker:"
                    docker --version

                    echo "Docker Compose:"
                    docker compose version

                    echo "Trivy:"
                    trivy --version

                    echo "============================================"
                    echo "TOOL VALIDATION PASSED"
                    echo "============================================"
                '''
            }
        }

        stage('Validate Project') {
            steps {
                sh '''
                    set -eu

                    echo "============================================"
                    echo "PROJECT VALIDATION"
                    echo "============================================"

                    test -f backend/go.mod
                    test -f backend/Dockerfile

                    test -f frontend/package.json
                    test -f frontend/package-lock.json
                    test -f frontend/Dockerfile

                    test -f docker-compose.yml

                    echo "Required project files found."

                    echo
                    echo "Git commit:"
                    git rev-parse --short HEAD

                    echo
                    echo "Project structure:"
                    find . -maxdepth 2 -type f | sort

                    echo
                    echo "PROJECT VALIDATION PASSED"
                '''
            }
        }

        stage('Backend Test') {
            steps {
                dir('backend') {
                    sh '''
                        set -eu

                        echo "============================================"
                        echo "BACKEND TEST"
                        echo "============================================"

                        go mod download
                        go test ./...
                        go vet ./...

                        echo
                        echo "BACKEND TEST PASSED"
                    '''
                }
            }
        }

        stage('Frontend Test and Build') {
            steps {
                dir('frontend') {
                    sh '''
                        set -eu

                        echo "============================================"
                        echo "FRONTEND TEST AND BUILD"
                        echo "============================================"

                        npm ci

                        npm run lint

                        npm run build

                        test -d dist

                        echo
                        echo "FRONTEND BUILD PASSED"
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu

                    echo "============================================"
                    echo "DOCKER IMAGE BUILD"
                    echo "============================================"

                    echo "Building backend..."

                    docker build \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${BACKEND_IMAGE}:latest \
                        ./backend

                    echo "Building frontend..."

                    docker build \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${FRONTEND_IMAGE}:latest \
                        ./frontend

                    echo
                    echo "Built images:"

                    docker images --format \
                        '{{.Repository}}:{{.Tag}} {{.Size}}' \
                        | grep -E '^shopverse-(backend|frontend):'

                    echo
                    echo "DOCKER BUILD PASSED"
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    set -eu

                    echo "============================================"
                    echo "TRIVY SECURITY SCAN"
                    echo "============================================"

                    mkdir -p trivy-reports

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        | tee trivy-reports/backend-${IMAGE_TAG}.txt

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        | tee trivy-reports/frontend-${IMAGE_TAG}.txt

                    echo
                    echo "TRIVY SCAN COMPLETED"
                '''
            }
        }

        stage('Prepare Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "============================================"
                    echo "PREPARING DEPLOYMENT"
                    echo "============================================"

                    test -d "${DEPLOY_DIR}"

                    test -f "${DEPLOY_DIR}/docker-compose.yml"
                    test -f "${DEPLOY_DIR}/.env"
                    test -f "${DEPLOY_DIR}/nginx/nginx.conf"

                    echo "Deployment files found."

                    cd "${DEPLOY_DIR}"

                    docker compose --env-file .env config >/dev/null

                    echo "Docker Compose configuration valid."
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -eu

                    echo "============================================"
                    echo "DEPLOYING SHOPVERSE"
                    echo "============================================"

                    cd "${DEPLOY_DIR}"

                    echo "Copying newly built images into deployment tags..."

                    docker tag \
                        ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        ${BACKEND_IMAGE}:jenkins-${BUILD_NUMBER}

                    docker tag \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        ${FRONTEND_IMAGE}:jenkins-${BUILD_NUMBER}

                    echo
                    echo "Starting MySQL..."

                    docker compose --env-file .env up -d mysql

                    echo "Waiting for MySQL..."

                    for i in $(seq 1 30); do

                        STATUS=$(docker inspect \
                            --format='{{if .State.Health}}{{.State.Health.Status}}{{else}}starting{{end}}' \
                            shopverse-mysql 2>/dev/null || true)

                        echo "MySQL status: ${STATUS}"

                        if [ "${STATUS}" = "healthy" ]; then
                            break
                        fi

                        sleep 2

                    done

                    STATUS=$(docker inspect \
                        --format='{{.State.Health.Status}}' \
                        shopverse-mysql)

                    if [ "${STATUS}" != "healthy" ]; then
                        echo "ERROR: MySQL did not become healthy."
                        exit 1
                    fi

                    echo "MySQL is healthy."

                    echo
                    echo "Starting backend..."

                    docker compose --env-file .env up -d backend

                    echo
                    echo "Starting frontend..."

                    docker compose --env-file .env up -d frontend

                    echo
                    echo "Starting Nginx..."

                    docker compose --env-file .env up -d nginx

                    echo
                    echo "Deployment containers:"
                    docker compose --env-file .env ps

                    echo
                    echo "DEPLOYMENT COMPLETED"
                '''
            }
        }

        stage('Application Verification') {
            steps {
                sh '''
                    set -eu

                    echo "============================================"
                    echo "APPLICATION VERIFICATION"
                    echo "============================================"

                    cd "${DEPLOY_DIR}"

                    echo
                    echo "Container status:"
                    docker compose --env-file .env ps

                    echo
                    echo "Testing backend through Docker network..."

                    BACKEND_RESPONSE=$(docker exec shopverse-nginx \
                        wget -qO- http://backend:8080/health)

                    echo "Backend response:"
                    echo "${BACKEND_RESPONSE}"

                    echo "${BACKEND_RESPONSE}" | grep -q '"status":"healthy"'

                    echo
                    echo "Testing frontend through Docker network..."

                    docker exec shopverse-nginx \
                        wget -qO- http://frontend:80/ \
                        >/tmp/shopverse-frontend.html

                    grep -q "ShopVerse" /tmp/shopverse-frontend.html

                    echo "Frontend response is valid."

                    echo
                    echo "Testing Nginx from host..."

                    curl -fsS http://127.0.0.1/ \
                        >/tmp/shopverse-nginx.html

                    grep -q "ShopVerse" /tmp/shopverse-nginx.html

                    echo "Nginx frontend routing works."

                    echo
                    echo "Testing Nginx health endpoint..."

                    curl -fsS http://127.0.0.1/health

                    echo
                    echo
                    echo "============================================"
                    echo "APPLICATION VERIFICATION PASSED"
                    echo "============================================"
                '''
            }
        }
    }

    post {

        always {
            archiveArtifacts(
                artifacts: 'trivy-reports/**/*',
                allowEmptyArchive: true
            )

            sh '''
                set +e

                echo
                echo "============================================"
                echo "FINAL CONTAINER STATUS"
                echo "============================================"

                cd "${DEPLOY_DIR}"

                docker compose --env-file .env ps

                echo
                echo "Docker containers:"
                docker ps \
                    --format 'table {{.Names}}\\t{{.Image}}\\t{{.Status}}\\t{{.Ports}}' \
                    | grep -E 'shopverse|NAMES'

                echo
            '''
        }

        success {
            echo '''
============================================
SHOPVERSE CI/CD PIPELINE SUCCESS
============================================
Build completed successfully.
Application deployed successfully.
'''
        }

        failure {
            echo '''
============================================
SHOPVERSE CI/CD PIPELINE FAILED
============================================
Check the failed stage above.
Existing running containers were not intentionally removed.
'''
        }
    }
}
