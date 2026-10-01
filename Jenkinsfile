pipeline {
    agent any

    environment {
        APP_NAME       = "shopverse"
        BACKEND_IMAGE  = "shopverse-backend"
        FRONTEND_IMAGE = "shopverse-frontend"

        IMAGE_TAG      = "${BUILD_NUMBER}"

        COMPOSE_PROJECT_NAME = "shopverse"

        TRIVY_REPORT_DIR = "trivy-reports"

        DEPLOY_DIR = "/opt/shopverse"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)

        buildDiscarder(
            logRotator(
                numToKeepStr: '10',
                artifactNumToKeepStr: '10'
            )
        )
    }

    stages {

        // =========================================================
        // 1. CLEAN WORKSPACE
        // =========================================================

        stage('Clean Workspace') {
            steps {
                deleteDir()

                sh '''
                    set -e

                    echo "===== WORKSPACE ====="
                    pwd
                    ls -la
                '''
            }
        }


        // =========================================================
        // 2. CHECKOUT LATEST CODE
        // =========================================================

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/vijaygiduthuri/shopverse.git'

                sh '''
                    set -e

                    echo "===== GIT REMOTE ====="
                    git remote -v

                    echo
                    echo "===== COMMIT ====="
                    git rev-parse HEAD

                    echo
                    echo "===== SHORT COMMIT ====="
                    git rev-parse --short HEAD

                    echo
                    echo "===== BRANCH ====="
                    git branch --show-current

                    echo
                    echo "===== STATUS ====="
                    git status
                '''
            }
        }


        // =========================================================
        // 3. VALIDATE PROJECT
        // =========================================================

        stage('Validate Project') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "PROJECT STRUCTURE"
                    echo "======================================"

                    find . -maxdepth 2 -type f | sort

                    echo
                    echo "======================================"
                    echo "GO VERSION"
                    echo "======================================"

                    go version

                    echo
                    echo "======================================"
                    echo "NODE VERSION"
                    echo "======================================"

                    node -v

                    echo
                    echo "======================================"
                    echo "NPM VERSION"
                    echo "======================================"

                    npm -v

                    echo
                    echo "======================================"
                    echo "DOCKER VERSION"
                    echo "======================================"

                    docker --version

                    echo
                    echo "======================================"
                    echo "DOCKER COMPOSE"
                    echo "======================================"

                    docker compose version

                    echo
                    echo "======================================"
                    echo "TRIVY VERSION"
                    echo "======================================"

                    trivy --version

                    echo
                    echo "======================================"
                    echo "REQUIRED FILES"
                    echo "======================================"

                    test -f backend/Dockerfile
                    test -f backend/go.mod
                    test -f backend/go.sum

                    test -f frontend/Dockerfile
                    test -f frontend/package.json
                    test -f frontend/package-lock.json

                    test -f docker-compose.yml

                    echo "Required files verified successfully."
                '''
            }
        }


        // =========================================================
        // 4. BACKEND DEPENDENCY INSTALLATION + TEST
        // =========================================================

        stage('Backend Dependency Installation & Testing') {
            steps {
                dir('backend') {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "GO MOD DOWNLOAD"
                        echo "======================================"

                        go mod download

                        echo
                        echo "======================================"
                        echo "GO MOD VERIFY"
                        echo "======================================"

                        go mod verify

                        echo
                        echo "======================================"
                        echo "GO TEST"
                        echo "======================================"

                        go test ./...

                        echo
                        echo "======================================"
                        echo "GO VET"
                        echo "======================================"

                        go vet ./...

                        echo
                        echo "Backend validation completed successfully."
                    '''
                }
            }
        }


        // =========================================================
        // 5. FRONTEND DEPENDENCY INSTALLATION + BUILD
        // =========================================================

        stage('Frontend Dependency Installation & Build') {
            steps {
                dir('frontend') {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "NPM CI"
                        echo "======================================"

                        npm ci

                        echo
                        echo "======================================"
                        echo "NPM LINT"
                        echo "======================================"

                        npm run lint

                        echo
                        echo "======================================"
                        echo "NPM BUILD"
                        echo "======================================"

                        npm run build

                        echo
                        echo "======================================"
                        echo "FRONTEND BUILD OUTPUT"
                        echo "======================================"

                        ls -lah dist

                        echo
                        echo "Frontend build completed successfully."
                    '''
                }
            }
        }


        // =========================================================
        // 6. DOCKER IMAGE BUILD
        // =========================================================

        stage('Docker Image Build') {
            steps {

                sh '''
                    set -e

                    echo "======================================"
                    echo "BUILD BACKEND IMAGE"
                    echo "======================================"

                    docker build \
                        --pull \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        ./backend

                    echo
                    echo "======================================"
                    echo "BUILD FRONTEND IMAGE"
                    echo "======================================"

                    docker build \
                        --pull \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        ./frontend

                    echo
                    echo "======================================"
                    echo "VERIFY IMAGES"
                    echo "======================================"

                    docker image inspect ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker image inspect ${FRONTEND_IMAGE}:${IMAGE_TAG}

                    echo
                    echo "======================================"
                    echo "SHOPVERSE IMAGES"
                    echo "======================================"

                    docker images --format \
                        'table {{.Repository}}\\t{{.Tag}}\\t{{.ID}}\\t{{.Size}}' \
                        | grep shopverse || true
                '''
            }
        }


        // =========================================================
        // 7. TRIVY SECURITY SCAN
        // =========================================================

        stage('Docker Image Security Scan') {
            steps {

                sh '''
                    set -e

                    echo "======================================"
                    echo "PREPARING TRIVY REPORT DIRECTORY"
                    echo "======================================"

                    mkdir -p ${TRIVY_REPORT_DIR}

                    rm -f ${TRIVY_REPORT_DIR}/* || true


                    echo
                    echo "======================================"
                    echo "BACKEND TRIVY SCAN"
                    echo "======================================"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format table \
                        ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        | tee ${TRIVY_REPORT_DIR}/backend-${IMAGE_TAG}.txt


                    echo
                    echo "======================================"
                    echo "FRONTEND TRIVY SCAN"
                    echo "======================================"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format table \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        | tee ${TRIVY_REPORT_DIR}/frontend-${IMAGE_TAG}.txt


                    echo
                    echo "======================================"
                    echo "TRIVY JSON - BACKEND"
                    echo "======================================"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format json \
                        -o ${TRIVY_REPORT_DIR}/backend-${IMAGE_TAG}.json \
                        ${BACKEND_IMAGE}:${IMAGE_TAG}


                    echo
                    echo "======================================"
                    echo "TRIVY JSON - FRONTEND"
                    echo "======================================"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format json \
                        -o ${TRIVY_REPORT_DIR}/frontend-${IMAGE_TAG}.json \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG}


                    echo
                    echo "======================================"
                    echo "TRIVY REPORTS GENERATED"
                    echo "======================================"

                    ls -lah ${TRIVY_REPORT_DIR}
                '''
            }
        }


        // =========================================================
        // 8. PREPARE DEPLOYMENT ENVIRONMENT
        // =========================================================

        stage('Prepare Deployment Environment') {
            steps {

                sh '''
                    set -e

                    echo "======================================"
                    echo "DEPLOYMENT DIRECTORY"
                    echo "======================================"

                    test -d ${DEPLOY_DIR}

                    cd ${DEPLOY_DIR}

                    echo
                    echo "======================================"
                    echo "CHECK ENV FILE"
                    echo "======================================"

                    if [ ! -f .env ]; then
                        echo "ERROR: ${DEPLOY_DIR}/.env does not exist."
                        echo "Create the production .env file before deployment."
                        exit 1
                    fi

                    echo ".env exists."


                    echo
                    echo "======================================"
                    echo "VALIDATE REQUIRED ENVIRONMENT VARIABLES"
                    echo "======================================"

                    required_vars="
                    MYSQL_ROOT_PASSWORD
                    MYSQL_DATABASE
                    MYSQL_USER
                    MYSQL_PASSWORD
                    JWT_SECRET
                    DB_HOST
                    DB_PORT
                    DB_NAME
                    DB_USER
                    DB_PASSWORD
                    FRONTEND_ORIGIN
                    PORT
                    "

                    for VAR in $required_vars
                    do
                        VALUE=$(grep "^${VAR}=" .env | cut -d '=' -f2-)

                        if [ -z "$VALUE" ]; then
                            echo "ERROR: Missing or empty variable: ${VAR}"
                            exit 1
                        fi
                    done

                    echo "All required environment variables are present."


                    echo
                    echo "======================================"
                    echo "COMPOSE CONFIG VALIDATION"
                    echo "======================================"

                    cd ${DEPLOY_DIR}

                    export IMAGE_TAG=${IMAGE_TAG}

                    docker compose config > /tmp/shopverse-compose-${BUILD_NUMBER}.yml

                    echo "Docker Compose configuration is valid."
                '''
            }
        }


        // =========================================================
        // 9. SAVE CURRENT DEPLOYMENT INFORMATION
        // =========================================================

        stage('Save Deployment State') {
            steps {

                sh '''
                    set -e

                    cd ${DEPLOY_DIR}

                    echo "======================================"
                    echo "CURRENT DEPLOYMENT STATE"
                    echo "======================================"

                    if docker inspect shopverse-backend >/dev/null 2>&1; then

                        CURRENT_BACKEND_IMAGE=$(docker inspect \
                            --format='{{.Config.Image}}' \
                            shopverse-backend)

                        echo "Current backend image:"
                        echo "${CURRENT_BACKEND_IMAGE}"

                        echo "${CURRENT_BACKEND_IMAGE}" \
                            > ${DEPLOY_DIR}/.previous-backend-image

                    else

                        echo "No previous backend deployment found."

                        rm -f ${DEPLOY_DIR}/.previous-backend-image
                    fi


                    if docker inspect shopverse-frontend >/dev/null 2>&1; then

                        CURRENT_FRONTEND_IMAGE=$(docker inspect \
                            --format='{{.Config.Image}}' \
                            shopverse-frontend)

                        echo "Current frontend image:"
                        echo "${CURRENT_FRONTEND_IMAGE}"

                        echo "${CURRENT_FRONTEND_IMAGE}" \
                            > ${DEPLOY_DIR}/.previous-frontend-image

                    else

                        echo "No previous frontend deployment found."

                        rm -f ${DEPLOY_DIR}/.previous-frontend-image
                    fi
                '''
            }
        }


        // =========================================================
        // 10. DEPLOY
        // =========================================================

        stage('Docker Compose Deployment') {
            steps {

                sh '''
                    set -e

                    cd ${DEPLOY_DIR}

                    echo "======================================"
                    echo "DEPLOYMENT IMAGE TAG"
                    echo "======================================"

                    echo "IMAGE_TAG=${IMAGE_TAG}"

                    export IMAGE_TAG=${IMAGE_TAG}


                    echo
                    echo "======================================"
                    echo "PULL REQUIRED BASE IMAGES"
                    echo "======================================"

                    docker compose pull mysql nginx


                    echo
                    echo "======================================"
                    echo "STOP PREVIOUS APPLICATION CONTAINERS"
                    echo "======================================"

                    docker compose stop backend frontend nginx || true


                    echo
                    echo "======================================"
                    echo "REMOVE OLD APPLICATION CONTAINERS"
                    echo "======================================"

                    docker compose rm -f backend frontend nginx || true


                    echo
                    echo "======================================"
                    echo "START MYSQL"
                    echo "======================================"

                    docker compose up -d mysql


                    echo
                    echo "======================================"
                    echo "WAIT FOR MYSQL"
                    echo "======================================"

                    MYSQL_READY=0

                    for i in $(seq 1 30)
                    do
                        STATUS=$(docker inspect \
                            --format='{{.State.Health.Status}}' \
                            shopverse-mysql 2>/dev/null || echo "missing")

                        echo "MySQL health: ${STATUS}"

                        if [ "$STATUS" = "healthy" ]; then
                            MYSQL_READY=1
                            break
                        fi

                        sleep 5
                    done

                    if [ "$MYSQL_READY" -ne 1 ]; then
                        echo "ERROR: MySQL did not become healthy."
                        exit 1
                    fi


                    echo
                    echo "======================================"
                    echo "START BACKEND"
                    echo "======================================"

                    docker compose up -d backend


                    echo
                    echo "======================================"
                    echo "START FRONTEND"
                    echo "======================================"

                    docker compose up -d frontend


                    echo
                    echo "======================================"
                    echo "START NGINX"
                    echo "======================================"

                    docker compose up -d nginx


                    echo
                    echo "======================================"
                    echo "DEPLOYMENT CONTAINERS"
                    echo "======================================"

                    docker compose ps
                '''
            }
        }


        // =========================================================
        // 11. SERVICE HEALTH CHECKS
        // =========================================================

        stage('Service Health Checks') {
            steps {

                sh '''
                    set -e

                    cd ${DEPLOY_DIR}

                    echo "======================================"
                    echo "DOCKER COMPOSE STATUS"
                    echo "======================================"

                    docker compose ps


                    echo
                    echo "======================================"
                    echo "MYSQL HEALTH CHECK"
                    echo "======================================"

                    MYSQL_READY=0

                    for i in $(seq 1 30)
                    do
                        STATUS=$(docker inspect \
                            --format='{{.State.Health.Status}}' \
                            shopverse-mysql 2>/dev/null || echo "missing")

                        echo "MySQL health: ${STATUS}"

                        if [ "$STATUS" = "healthy" ]; then
                            MYSQL_READY=1
                            break
                        fi

                        sleep 5
                    done

                    if [ "$MYSQL_READY" -ne 1 ]; then
                        echo "ERROR: MySQL health check failed."
                        exit 1
                    fi


                    echo
                    echo "======================================"
                    echo "NGINX HEALTH CHECK"
                    echo "======================================"

                    NGINX_READY=0

                    for i in $(seq 1 20)
                    do
                        if curl -fsS http://127.0.0.1/health >/dev/null; then
                            NGINX_READY=1
                            echo "Nginx health check successful."
                            break
                        fi

                        echo "Waiting for Nginx..."
                        sleep 5
                    done

                    if [ "$NGINX_READY" -ne 1 ]; then
                        echo "ERROR: Nginx health check failed."
                        exit 1
                    fi


                    echo
                    echo "======================================"
                    echo "CONTAINER RUNNING CHECK"
                    echo "======================================"

                    for CONTAINER in \
                        shopverse-mysql \
                        shopverse-backend \
                        shopverse-frontend \
                        shopverse-nginx
                    do

                        RUNNING=$(docker inspect \
                            --format='{{.State.Running}}' \
                            "$CONTAINER" 2>/dev/null || echo "false")

                        echo "${CONTAINER}: ${RUNNING}"

                        if [ "$RUNNING" != "true" ]; then
                            echo "ERROR: ${CONTAINER} is not running."
                            exit 1
                        fi

                    done


                    echo
                    echo "All required containers are running."
                '''
            }
        }


        // =========================================================
        // 12. APPLICATION SMOKE TEST
        // =========================================================

        stage('Application Smoke Test') {
            steps {

                sh '''
                    set -e

                    echo "======================================"
                    echo "NGINX ROOT TEST"
                    echo "======================================"

                    HTTP_CODE=$(curl \
                        -s \
                        -o /tmp/shopverse-home.html \
                        -w "%{http_code}" \
                        http://127.0.0.1/)

                    echo "HTTP status: ${HTTP_CODE}"

                    if [ "$HTTP_CODE" -lt 200 ] || [ "$HTTP_CODE" -ge 400 ]; then
                        echo "ERROR: Frontend is not accessible."
                        exit 1
                    fi


                    echo
                    echo "======================================"
                    echo "NGINX HEALTH ENDPOINT"
                    echo "======================================"

                    curl -fsS http://127.0.0.1/health

                    echo


                    echo
                    echo "======================================"
                    echo "BACKEND CONTAINER LOG CHECK"
                    echo "======================================"

                    docker logs \
                        --tail 50 \
                        shopverse-backend || true


                    echo
                    echo "======================================"
                    echo "FRONTEND CONTAINER LOG CHECK"
                    echo "======================================"

                    docker logs \
                        --tail 50 \
                        shopverse-frontend || true


                    echo
                    echo "======================================"
                    echo "NGINX CONTAINER LOG CHECK"
                    echo "======================================"

                    docker logs \
                        --tail 50 \
                        shopverse-nginx || true


                    echo
                    echo "Application smoke test completed successfully."
                '''
            }
        }


        // =========================================================
        // 13. RECORD SUCCESSFUL VERSION
        // =========================================================

        stage('Record Successful Deployment') {
            steps {

                sh '''
                    set -e

                    cd ${DEPLOY_DIR}

                    echo "${IMAGE_TAG}" \
                        > ${DEPLOY_DIR}/.last-successful-build

                    echo "Successful deployment recorded:"
                    cat ${DEPLOY_DIR}/.last-successful-build
                '''
            }
        }


        // =========================================================
        // 14. CLEANUP OLD APPLICATION IMAGES
        // =========================================================

        stage('Docker Image Cleanup') {
            steps {

                sh '''
                    set -e

                    echo "======================================"
                    echo "CURRENT SHOPVERSE IMAGES"
                    echo "======================================"

                    docker images \
                        --format '{{.Repository}}:{{.Tag}} {{.ID}}' \
                        | grep -E '^shopverse-(backend|frontend):' \
                        || true


                    echo
                    echo "======================================"
                    echo "REMOVE DANGLING IMAGES"
                    echo "======================================"

                    docker image prune -f


                    echo
                    echo "======================================"
                    echo "KEEP RECENT SHOPVERSE IMAGES"
                    echo "======================================"

                    KEEP_COUNT=3

                    for IMAGE in \
                        ${BACKEND_IMAGE} \
                        ${FRONTEND_IMAGE}
                    do

                        echo
                        echo "Processing ${IMAGE}"

                        docker images \
                            "${IMAGE}" \
                            --format '{{.Tag}} {{.ID}}' \
                            | grep -E '^[0-9]+ ' \
                            | sort -rn \
                            | tail -n +$((KEEP_COUNT + 1)) \
                            | awk '{print $2}' \
                            | while read IMAGE_ID
                            do
                                if [ -n "$IMAGE_ID" ]; then
                                    echo "Removing old image: ${IMAGE_ID}"
                                    docker rmi "$IMAGE_ID" || true
                                fi
                            done

                    done


                    echo
                    echo "======================================"
                    echo "FINAL IMAGE LIST"
                    echo "======================================"

                    docker images \
                        --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}' \
                        | grep -E 'shopverse|REPOSITORY' \
                        || true
                '''
            }
        }
    }


    // =============================================================
    // POST ACTIONS
    // =============================================================

    post {

        success {

            echo '''
============================================================
SHOPVERSE PIPELINE SUCCESS
============================================================

Git Checkout
     ↓
Backend Test
     ↓
Frontend Build
     ↓
Docker Build
     ↓
Trivy Security Scan
     ↓
Docker Compose Deployment
     ↓
Health Checks
     ↓
Application Smoke Test
     ↓
Successful Deployment Recorded
     ↓
Docker Image Cleanup

Deployment completed successfully.
============================================================
'''
        }


        failure {

            echo '''
============================================================
SHOPVERSE PIPELINE FAILED
============================================================

The pipeline failed.

The deployment/health-check failure must be investigated.
Rollback will be attempted using the previously recorded
application images where available.

============================================================
'''


            sh '''
                set +e

                DEPLOY_DIR="/opt/shopverse"

                echo "======================================"
                echo "ROLLBACK CHECK"
                echo "======================================"

                cd ${DEPLOY_DIR} || exit 0


                if [ ! -f .previous-backend-image ]; then
                    echo "No previous backend image recorded."
                    exit 0
                fi

                if [ ! -f .previous-frontend-image ]; then
                    echo "No previous frontend image recorded."
                    exit 0
                fi


                PREVIOUS_BACKEND=$(cat .previous-backend-image)
                PREVIOUS_FRONTEND=$(cat .previous-frontend-image)


                echo "Previous backend image:"
                echo "${PREVIOUS_BACKEND}"

                echo
                echo "Previous frontend image:"
                echo "${PREVIOUS_FRONTEND}"


                echo
                echo "======================================"
                echo "VERIFY PREVIOUS IMAGES"
                echo "======================================"

                docker image inspect "${PREVIOUS_BACKEND}" >/dev/null 2>&1

                if [ $? -ne 0 ]; then
                    echo "Previous backend image is not available."
                    exit 0
                fi

                docker image inspect "${PREVIOUS_FRONTEND}" >/dev/null 2>&1

                if [ $? -ne 0 ]; then
                    echo "Previous frontend image is not available."
                    exit 0
                fi


                echo
                echo "======================================"
                echo "ROLLING BACK APPLICATION"
                echo "======================================"

                BACKEND_TAG=$(echo "${PREVIOUS_BACKEND}" | cut -d ':' -f2)
                FRONTEND_TAG=$(echo "${PREVIOUS_FRONTEND}" | cut -d ':' -f2)

                if [ -z "${BACKEND_TAG}" ] || [ -z "${FRONTEND_TAG}" ]; then
                    echo "Unable to determine previous image tags."
                    exit 0
                fi


                export IMAGE_TAG="${BACKEND_TAG}"


                echo "Rollback backend image: ${PREVIOUS_BACKEND}"
                echo "Rollback frontend image: ${PREVIOUS_FRONTEND}"


                echo
                echo "Stopping failed deployment..."

                docker compose stop backend frontend nginx || true

                docker compose rm -f backend frontend nginx || true


                echo
                echo "Starting previous backend..."

                docker compose up -d backend


                echo
                echo "Starting previous frontend..."

                docker compose up -d frontend


                echo
                echo "Starting Nginx..."

                docker compose up -d nginx


                echo
                echo "Waiting for rollback..."

                sleep 15


                echo
                echo "======================================"
                echo "ROLLBACK CONTAINER STATUS"
                echo "======================================"

                docker compose ps


                echo
                echo "======================================"
                echo "ROLLBACK HEALTH CHECK"
                echo "======================================"

                MYSQL_STATUS=$(docker inspect \
                    --format='{{.State.Health.Status}}' \
                    shopverse-mysql 2>/dev/null || echo "unknown")

                echo "MySQL: ${MYSQL_STATUS}"


                if ! curl -fsS http://127.0.0.1/health >/dev/null; then
                    echo "ROLLBACK FAILED: Nginx health check failed."
                    exit 0
                fi


                if ! curl -fsS http://127.0.0.1/ >/dev/null; then
                    echo "ROLLBACK FAILED: Frontend is not accessible."
                    exit 0
                fi


                echo
                echo "============================================================"
                echo "ROLLBACK COMPLETED"
                echo "============================================================"

                echo "Previous application version has been restored."
                echo "MySQL volume was preserved."
                echo "No 'docker compose down -v' was executed."

            '''
        }


        always {

            echo "============================================================"
            echo "BUILD ${BUILD_NUMBER} FINISHED"
            echo "============================================================"

            archiveArtifacts(
                artifacts: 'trivy-reports/**/*',
                allowEmptyArchive: true,
                fingerprint: true
            )

            sh '''
                set +e

                echo "===== FINAL DOCKER STATUS ====="

                docker ps --format \
                    'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}' \
                    | grep -E 'shopverse|NAMES' || true
            '''
        }
    }
}
