pipeline {

    agent any

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(
            numToKeepStr: '10',
            artifactNumToKeepStr: '10'
        ))
    }

    environment {

        REPOSITORY_URL = 'https://github.com/sravanprasad7563-blip/Shopverse.git'
        BRANCH_NAME = 'main'

        DEPLOY_DIR = '/opt/shopverse'

        BACKEND_IMAGE = 'shopverse-backend'
        FRONTEND_IMAGE = 'shopverse-frontend'

        IMAGE_TAG = "${BUILD_NUMBER}"

        TRIVY_REPORT_DIR = 'trivy-reports'

        PATH = '/usr/local/go/bin:/usr/local/bin:/usr/bin:/bin'
    }

    stages {

        /*
         * ============================================================
         * 1. CLEAN WORKSPACE
         * ============================================================
         */

        stage('Clean Workspace') {
            steps {

                deleteDir()

                sh '''
                    set -eu

                    echo "============================================================"
                    echo "WORKSPACE"
                    echo "============================================================"

                    pwd
                    ls -la
                '''
            }
        }


        /*
         * ============================================================
         * 2. CHECKOUT
         * ============================================================
         */

        stage('Checkout') {
            steps {

                git(
                    branch: "${BRANCH_NAME}",
                    url: "${REPOSITORY_URL}"
                )

                sh '''
                    set -eu

                    echo "============================================================"
                    echo "GIT REMOTE"
                    echo "============================================================"

                    git remote -v

                    echo
                    echo "============================================================"
                    echo "COMMIT"
                    echo "============================================================"

                    git rev-parse HEAD

                    echo
                    echo "SHORT COMMIT:"
                    git rev-parse --short HEAD

                    echo
                    echo "BRANCH:"
                    git branch --show-current

                    echo
                    echo "STATUS:"
                    git status --short

                    echo
                    echo "LAST COMMIT:"
                    git log -1 --oneline
                '''
            }
        }


        /*
         * ============================================================
         * 3. VALIDATE TOOLS AND PROJECT
         * ============================================================
         */

        stage('Validate Project') {
            steps {

                sh '''
                    set -eu

                    echo "============================================================"
                    echo "PROJECT STRUCTURE"
                    echo "============================================================"

                    find . -maxdepth 2 -type f -not -path './.git/*' | sort

                    echo
                    echo "============================================================"
                    echo "GO"
                    echo "============================================================"

                    command -v go
                    go version

                    echo
                    echo "============================================================"
                    echo "NODE"
                    echo "============================================================"

                    command -v node
                    node -v

                    echo
                    echo "============================================================"
                    echo "NPM"
                    echo "============================================================"

                    command -v npm
                    npm -v

                    echo
                    echo "============================================================"
                    echo "DOCKER"
                    echo "============================================================"

                    command -v docker
                    docker --version

                    echo
                    echo "============================================================"
                    echo "DOCKER COMPOSE"
                    echo "============================================================"

                    docker compose version

                    echo
                    echo "============================================================"
                    echo "TRIVY"
                    echo "============================================================"

                    command -v trivy
                    trivy --version

                    echo
                    echo "============================================================"
                    echo "GIT"
                    echo "============================================================"

                    git --version

                    echo
                    echo "============================================================"
                    echo "DOCKER ACCESS"
                    echo "============================================================"

                    docker info >/dev/null

                    echo "Docker access successful."
                '''
            }
        }


        /*
         * ============================================================
         * 4. BACKEND TEST
         * ============================================================
         */

        stage('Backend Dependency Installation & Testing') {
            steps {

                dir('backend') {

                    sh '''
                        set -eu

                        echo "============================================================"
                        echo "GO MOD DOWNLOAD"
                        echo "============================================================"

                        go mod download

                        echo
                        echo "============================================================"
                        echo "GO MOD VERIFY"
                        echo "============================================================"

                        go mod verify

                        echo
                        echo "============================================================"
                        echo "GO TEST"
                        echo "============================================================"

                        go test ./...

                        echo
                        echo "============================================================"
                        echo "GO VET"
                        echo "============================================================"

                        go vet ./...

                        echo
                        echo "Backend validation successful."
                    '''
                }
            }
        }


        /*
         * ============================================================
         * 5. FRONTEND TEST
         * ============================================================
         */

        stage('Frontend Dependency Installation & Build') {
            steps {

                dir('frontend') {

                    sh '''
                        set -eu

                        echo "============================================================"
                        echo "NPM CI"
                        echo "============================================================"

                        npm ci

                        echo
                        echo "============================================================"
                        echo "NPM AUDIT"
                        echo "============================================================"

                        npm audit --audit-level=high --json > ../npm-audit.json 2>&1 || true

                        echo "NPM audit completed."
                        echo "Audit report saved as npm-audit.json"

                        echo
                        echo "============================================================"
                        echo "LINT"
                        echo "============================================================"

                        npm run lint

                        echo
                        echo "============================================================"
                        echo "BUILD"
                        echo "============================================================"

                        npm run build

                        echo
                        echo "Frontend validation successful."
                    '''
                }
            }
        }


        /*
         * ============================================================
         * 6. BUILD DOCKER IMAGES
         * ============================================================
         */

        stage('Docker Image Build') {
            steps {

                sh '''
                    set -eu

                    echo "============================================================"
                    echo "BACKEND DOCKER BUILD"
                    echo "============================================================"

                    docker build \
                        --pull \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${BACKEND_IMAGE}:latest \
                        ./backend

                    echo
                    echo "============================================================"
                    echo "FRONTEND DOCKER BUILD"
                    echo "============================================================"

                    docker build \
                        --pull \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${FRONTEND_IMAGE}:latest \
                        ./frontend

                    echo
                    echo "============================================================"
                    echo "IMAGE VERIFICATION"
                    echo "============================================================"

                    docker image inspect ${BACKEND_IMAGE}:${IMAGE_TAG} >/dev/null
                    docker image inspect ${FRONTEND_IMAGE}:${IMAGE_TAG} >/dev/null

                    docker images --filter reference=${BACKEND_IMAGE} \
                                  --filter reference=${FRONTEND_IMAGE}

                    echo
                    echo "Docker image build successful."
                '''
            }
        }


        /*
         * ============================================================
         * 7. TRIVY SECURITY SCAN
         * ============================================================
         */

        stage('Docker Image Security Scan') {
            steps {

                sh '''
                    set -eu

                    mkdir -p ${TRIVY_REPORT_DIR}

                    echo "============================================================"
                    echo "TRIVY BACKEND HIGH/CRITICAL SCAN"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --output ${TRIVY_REPORT_DIR}/backend-${IMAGE_TAG}.txt \
                        ${BACKEND_IMAGE}:${IMAGE_TAG} || true

                    echo
                    echo "============================================================"
                    echo "TRIVY FRONTEND HIGH/CRITICAL SCAN"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --output ${TRIVY_REPORT_DIR}/frontend-${IMAGE_TAG}.txt \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG} || true

                    echo
                    echo "============================================================"
                    echo "TRIVY BACKEND JSON"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --format json \
                        --output ${TRIVY_REPORT_DIR}/backend-${IMAGE_TAG}.json \
                        ${BACKEND_IMAGE}:${IMAGE_TAG} || true

                    echo
                    echo "============================================================"
                    echo "TRIVY FRONTEND JSON"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --format json \
                        --output ${TRIVY_REPORT_DIR}/frontend-${IMAGE_TAG}.json \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG} || true

                    echo
                    echo "Security scan completed."
                '''
            }
        }


        /*
         * ============================================================
         * 8. PREPARE DEPLOYMENT
         * ============================================================
         */

        stage('Prepare Deployment Environment') {
            steps {

                sh '''
                    set -eu

                    echo "============================================================"
                    echo "DEPLOYMENT DIRECTORY"
                    echo "============================================================"

                    test -d "${DEPLOY_DIR}"

                    cd "${DEPLOY_DIR}"

                    echo
                    echo "============================================================"
                    echo "REQUIRED FILES"
                    echo "============================================================"

                    test -f docker-compose.yml
                    test -f .env
                    test -f nginx/nginx.conf

                    echo "Required deployment files exist."

                    echo
                    echo "============================================================"
                    echo "ENVIRONMENT ACCESS"
                    echo "============================================================"

                    test -r .env

                    echo ".env is readable by Jenkins."

                    echo
                    echo "============================================================"
                    echo "DOCKER COMPOSE VALIDATION"
                    echo "============================================================"

                    export IMAGE_TAG="${IMAGE_TAG}"

                    docker compose --env-file .env config >/tmp/shopverse-compose-${BUILD_NUMBER}.yml

                    echo "Docker Compose validation successful."

                    echo
                    echo "============================================================"
                    echo "NGINX CONFIGURATION VALIDATION"
                    echo "============================================================"

                    docker run --rm \
                        -v "${DEPLOY_DIR}/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro" \
                        nginx:1.27-alpine \
                        nginx -t

                    echo "Nginx configuration validation successful."
                '''
            }
        }


        /*
         * ============================================================
         * 9. SAVE CURRENT DEPLOYMENT STATE
         * ============================================================
         */

        stage('Save Deployment State') {
            steps {

                script {
                    env.DEPLOYMENT_STATE_SAVED = 'false'
                }

                sh '''
                    set -eu

                    cd "${DEPLOY_DIR}"

                    echo "============================================================"
                    echo "CURRENT DEPLOYMENT STATE"
                    echo "============================================================"

                    rm -f .previous-backend-image
                    rm -f .previous-frontend-image

                    BACKEND_CURRENT=$(docker inspect \
                        -f '{{.Config.Image}}' \
                        shopverse-backend 2>/dev/null || true)

                    FRONTEND_CURRENT=$(docker inspect \
                        -f '{{.Config.Image}}' \
                        shopverse-frontend 2>/dev/null || true)

                    echo "Current backend image: ${BACKEND_CURRENT:-NONE}"
                    echo "Current frontend image: ${FRONTEND_CURRENT:-NONE}"

                    if [ -n "${BACKEND_CURRENT}" ] && [ -n "${FRONTEND_CURRENT}" ]; then

                        printf '%s\\n' "${BACKEND_CURRENT}" > .previous-backend-image
                        printf '%s\\n' "${FRONTEND_CURRENT}" > .previous-frontend-image

                        echo "Previous deployment state saved."

                    else

                        echo "No complete previous application deployment found."
                        echo "This can be treated as the initial deployment."

                    fi
                '''

                script {
                    env.DEPLOYMENT_STATE_SAVED = 'true'
                }
            }
        }


        /*
         * ============================================================
         * 10. DEPLOY
         * ============================================================
         */

        stage('Docker Compose Deployment') {
            steps {

                script {
                    env.DEPLOYMENT_STARTED = 'true'
                }

                sh '''
                    set -eu

                    cd "${DEPLOY_DIR}"

                    echo "============================================================"
                    echo "SHOPVERSE DEPLOYMENT"
                    echo "============================================================"

                    echo "Deployment image tag: ${IMAGE_TAG}"

                    export IMAGE_TAG="${IMAGE_TAG}"

                    echo
                    echo "============================================================"
                    echo "MYSQL"
                    echo "============================================================"

                    docker compose --env-file .env up -d mysql

                    echo
                    echo "Waiting for MySQL..."

                    MYSQL_READY=false

                    for i in $(seq 1 30); do

                        STATUS=$(docker inspect \
                            -f '{{if .State.Health}}{{.State.Health.Status}}{{else}}unknown{{end}}' \
                            shopverse-mysql 2>/dev/null || echo "unknown")

                        echo "MySQL health: ${STATUS}"

                        if [ "${STATUS}" = "healthy" ]; then
                            MYSQL_READY=true
                            break
                        fi

                        sleep 5

                    done

                    if [ "${MYSQL_READY}" != "true" ]; then
                        echo "ERROR: MySQL did not become healthy."
                        docker logs --tail 100 shopverse-mysql || true
                        exit 1
                    fi

                    echo
                    echo "============================================================"
                    echo "BACKEND"
                    echo "============================================================"

                    docker compose --env-file .env up -d --force-recreate --no-deps backend

                    echo
                    echo "============================================================"
                    echo "FRONTEND"
                    echo "============================================================"

                    docker compose --env-file .env up -d --force-recreate --no-deps frontend

                    echo
                    echo "============================================================"
                    echo "NGINX"
                    echo "============================================================"

                    docker compose --env-file .env up -d --force-recreate --no-deps nginx

                    echo
                    echo "============================================================"
                    echo "COMPOSE STATUS"
                    echo "============================================================"

                    docker compose --env-file .env ps

                    echo
                    echo "Deployment containers started successfully."
                '''
            }
        }


        /*
         * ============================================================
         * 11. HEALTH CHECKS
         * ============================================================
         */

        stage('Service Health Checks') {
            steps {

                sh '''
                    set -eu

                    cd "${DEPLOY_DIR}"

                    echo "============================================================"
                    echo "SERVICE HEALTH CHECKS"
                    echo "============================================================"

                    docker compose --env-file .env ps

                    echo
                    echo "============================================================"
                    echo "MYSQL HEALTH"
                    echo "============================================================"

                    MYSQL_STATUS=$(docker inspect \
                        -f '{{if .State.Health}}{{.State.Health.Status}}{{else}}unknown{{end}}' \
                        shopverse-mysql)

                    echo "MySQL: ${MYSQL_STATUS}"

                    if [ "${MYSQL_STATUS}" != "healthy" ]; then
                        echo "MySQL health check failed."
                        exit 1
                    fi

                    echo
                    echo "============================================================"
                    echo "NGINX HEALTH"
                    echo "============================================================"

                    NGINX_STATUS=$(docker inspect \
                        -f '{{if .State.Health}}{{.State.Health.Status}}{{else}}unknown{{end}}' \
                        shopverse-nginx)

                    echo "Nginx: ${NGINX_STATUS}"

                    if [ "${NGINX_STATUS}" != "healthy" ]; then
                        echo "Nginx health check failed."

                        docker logs --tail 100 shopverse-nginx || true

                        exit 1
                    fi

                    echo
                    echo "============================================================"
                    echo "BACKEND CONTAINER"
                    echo "============================================================"

                    BACKEND_RUNNING=$(docker inspect \
                        -f '{{.State.Running}}' \
                        shopverse-backend)

                    echo "Backend running: ${BACKEND_RUNNING}"

                    if [ "${BACKEND_RUNNING}" != "true" ]; then
                        echo "Backend container is not running."
                        docker logs --tail 100 shopverse-backend || true
                        exit 1
                    fi

                    echo
                    echo "============================================================"
                    echo "FRONTEND CONTAINER"
                    echo "============================================================"

                    FRONTEND_RUNNING=$(docker inspect \
                        -f '{{.State.Running}}' \
                        shopverse-frontend)

                    echo "Frontend running: ${FRONTEND_RUNNING}"

                    if [ "${FRONTEND_RUNNING}" != "true" ]; then
                        echo "Frontend container is not running."
                        docker logs --tail 100 shopverse-frontend || true
                        exit 1
                    fi

                    echo
                    echo "All service health checks passed."
                '''
            }
        }


        /*
         * ============================================================
         * 12. APPLICATION SMOKE TEST
         * ============================================================
         */

        stage('Application Smoke Test') {
            steps {

                sh '''
                    set -eu

                    echo "============================================================"
                    echo "APPLICATION SMOKE TEST"
                    echo "============================================================"

                    echo
                    echo "Testing /health..."

                    curl --fail --silent --show-error \
                        --max-time 15 \
                        http://127.0.0.1/health

                    echo

                    echo
                    echo "Testing frontend..."

                    curl --fail --silent --show-error \
                        --max-time 15 \
                        http://127.0.0.1/ \
                        >/tmp/shopverse-homepage.html

                    test -s /tmp/shopverse-homepage.html

                    echo "Frontend response received."

                    echo
                    echo "Application smoke tests passed."
                '''
            }
        }


        /*
         * ============================================================
         * 13. RECORD SUCCESSFUL DEPLOYMENT
         * ============================================================
         */

        stage('Record Successful Deployment') {
            steps {

                sh '''
                    set -eu

                    cd "${DEPLOY_DIR}"

                    echo "============================================================"
                    echo "RECORD SUCCESSFUL DEPLOYMENT"
                    echo "============================================================"

                    printf '%s\\n' "${IMAGE_TAG}" > .current-image-tag

                    docker inspect \
                        -f '{{.Config.Image}}' \
                        shopverse-backend \
                        > .current-backend-image

                    docker inspect \
                        -f '{{.Config.Image}}' \
                        shopverse-frontend \
                        > .current-frontend-image

                    echo "Current deployment recorded."

                    echo
                    echo "Backend:"
                    cat .current-backend-image

                    echo
                    echo "Frontend:"
                    cat .current-frontend-image
                '''

                script {
                    env.DEPLOYMENT_SUCCESS = 'true'
                }
            }
        }


        /*
         * ============================================================
         * 14. CLEANUP
         * ============================================================
         */

        stage('Docker Image Cleanup') {
            steps {

                sh '''
                    set +e

                    echo "============================================================"
                    echo "DOCKER CLEANUP"
                    echo "============================================================"

                    docker image prune -f

                    echo
                    echo "Cleanup completed."
                '''
            }
        }
    }


    /*
     * ================================================================
     * POST ACTIONS
     * ================================================================
     */

    post {

        always {

            echo '''
============================================================
PIPELINE COMPLETE
============================================================
'''

            archiveArtifacts(
                artifacts: 'trivy-reports/**/*,npm-audit.json',
                allowEmptyArchive: true,
                fingerprint: true
            )

            sh '''
                set +e

                echo
                echo "============================================================"
                echo "FINAL DOCKER STATUS"
                echo "============================================================"

                docker ps \
                    --format 'table {{.Names}}\\t{{.Image}}\\t{{.Status}}\\t{{.Ports}}' \
                    | grep -E 'shopverse|NAMES' || true

                echo
                echo "============================================================"
                echo "FINAL SHOPVERSE IMAGES"
                echo "============================================================"

                docker images \
                    --filter reference=shopverse-backend \
                    --filter reference=shopverse-frontend || true
            '''
        }


        success {

            echo '''
============================================================
SHOPVERSE CI/CD SUCCESS
============================================================

Pipeline completed successfully.

Application:
    ShopVerse

Deployment:
    Docker Compose

Backend:
    Go

Frontend:
    Node / Vite

Security:
    Trivy

Reverse Proxy:
    Nginx

Health Checks:
    MySQL
    Backend
    Frontend
    Nginx

Smoke Tests:
    /health
    /

============================================================
'''
        }


        failure {

            echo '''
============================================================
SHOPVERSE PIPELINE FAILED
============================================================

A pipeline stage failed.

The rollback procedure will now be evaluated.

============================================================
'''

            sh '''
                set +e

                DEPLOY_DIR="/opt/shopverse"

                if [ ! -d "${DEPLOY_DIR}" ]; then
                    echo "Deployment directory does not exist."
                    exit 0
                fi

                cd "${DEPLOY_DIR}"

                echo "============================================================"
                echo "ROLLBACK CHECK"
                echo "============================================================"

                if [ "${DEPLOYMENT_STARTED:-false}" != "true" ]; then

                    echo "Deployment was never started."
                    echo "Rollback is not required."

                    exit 0

                fi

                if [ "${DEPLOYMENT_SUCCESS:-false}" = "true" ]; then

                    echo "Deployment completed successfully."
                    echo "Rollback is not required."

                    exit 0

                fi

                if [ ! -f .previous-backend-image ] || \
                   [ ! -f .previous-frontend-image ]; then

                    echo "No complete previous deployment state found."
                    echo "Rollback cannot be performed automatically."

                    exit 0
                fi

                PREVIOUS_BACKEND_IMAGE=$(cat .previous-backend-image)
                PREVIOUS_FRONTEND_IMAGE=$(cat .previous-frontend-image)

                echo "Previous backend image:"
                echo "${PREVIOUS_BACKEND_IMAGE}"

                echo
                echo "Previous frontend image:"
                echo "${PREVIOUS_FRONTEND_IMAGE}"

                PREVIOUS_BACKEND_TAG="${PREVIOUS_BACKEND_IMAGE#shopverse-backend:}"
                PREVIOUS_FRONTEND_TAG="${PREVIOUS_FRONTEND_IMAGE#shopverse-frontend:}"

                if [ "${PREVIOUS_BACKEND_TAG}" != "${PREVIOUS_FRONTEND_TAG}" ]; then
                    echo "Backend and frontend previous tags do not match."
                    echo "Rollback aborted."
                    exit 0
                fi

                ROLLBACK_TAG="${PREVIOUS_BACKEND_TAG}"

                echo
                echo "Rollback image tag: ${ROLLBACK_TAG}"

                export IMAGE_TAG="${ROLLBACK_TAG}"

                echo
                echo "============================================================"
                echo "ROLLING BACK BACKEND"
                echo "============================================================"

                docker compose --env-file .env \
                    up -d \
                    --force-recreate \
                    --no-deps \
                    backend

                echo
                echo "============================================================"
                echo "ROLLING BACK FRONTEND"
                echo "============================================================"

                docker compose --env-file .env \
                    up -d \
                    --force-recreate \
                    --no-deps \
                    frontend

                echo
                echo "============================================================"
                echo "ROLLING BACK NGINX"
                echo "============================================================"

                docker compose --env-file .env \
                    up -d \
                    --force-recreate \
                    --no-deps \
                    nginx

                echo
                echo "Waiting for rollback services..."

                sleep 10

                echo
                echo "============================================================"
                echo "ROLLBACK HEALTH CHECK"
                echo "============================================================"

                docker compose --env-file .env ps

                if ! curl \
                    --fail \
                    --silent \
                    --show-error \
                    --max-time 15 \
                    http://127.0.0.1/health \
                    >/dev/null; then

                    echo "ROLLBACK FAILED: /health check failed."

                    docker logs --tail 100 shopverse-backend || true
                    docker logs --tail 100 shopverse-nginx || true

                    exit 1
                fi

                echo
                echo "Rollback completed successfully."

                printf '%s\\n' "${ROLLBACK_TAG}" > .current-image-tag

                docker inspect \
                    -f '{{.Config.Image}}' \
                    shopverse-backend \
                    > .current-backend-image

                docker inspect \
                    -f '{{.Config.Image}}' \
                    shopverse-frontend \
                    > .current-frontend-image

            '''
        }
    }
}
