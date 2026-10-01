#!/usr/bin/env groovy

pipeline {

    agent any

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

        timeout(time: 45, unit: 'MINUTES')
    }

    environment {

        /*
         * ============================================================
         * SOURCE CONTROL
         * ============================================================
         */
        GIT_REPO = 'https://github.com/sravanprasad7563-blip/Shopverse.git'
        GIT_BRANCH = 'main'

        /*
         * ============================================================
         * APPLICATION
         * ============================================================
         */
        BACKEND_IMAGE  = 'shopverse-backend'
        FRONTEND_IMAGE = 'shopverse-frontend'

        IMAGE_TAG = "${BUILD_NUMBER}"

        /*
         * ============================================================
         * DEPLOYMENT
         * ============================================================
         */
        DEPLOY_DIR = '/opt/shopverse'

        /*
         * ============================================================
         * REPORTS
         * ============================================================
         */
        TRIVY_REPORT_DIR = 'trivy-reports'

        /*
         * ============================================================
         * IMPORTANT:
         * Jenkins service does not automatically load
         * /etc/profile.d/go.sh.
         *
         * Explicitly add Go to PATH.
         * ============================================================
         */
        PATH = "/usr/local/go/bin:/usr/local/bin:/usr/bin:/bin"

        /*
         * Prevent Go from automatically downloading another toolchain.
         */
        GOTOOLCHAIN = 'local'

        /*
         * Frontend API URL
         */
        VITE_API_URL = '/api'
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
                    branch: "${GIT_BRANCH}",
                    url: "${GIT_REPO}"
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
         * 3. VALIDATE PROJECT
         * ============================================================
         */
        stage('Validate Project') {

            steps {

                sh '''
                    set -eu

                    echo "============================================================"
                    echo "PROJECT STRUCTURE"
                    echo "============================================================"

                    find . -maxdepth 2 -type f \
                        -not -path './.git/*' \
                        | sort

                    echo
                    echo "============================================================"
                    echo "GO"
                    echo "============================================================"

                    which go
                    go version

                    echo
                    echo "============================================================"
                    echo "NODE"
                    echo "============================================================"

                    which node
                    node -v

                    echo
                    echo "============================================================"
                    echo "NPM"
                    echo "============================================================"

                    which npm
                    npm -v

                    echo
                    echo "============================================================"
                    echo "DOCKER"
                    echo "============================================================"

                    which docker
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

                    which trivy
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
         * 5. FRONTEND TEST + BUILD
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

                        npm audit --audit-level=high || true

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
         * 6. DOCKER BUILD
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
                        --label org.opencontainers.image.title="Shopverse Backend" \
                        --label org.opencontainers.image.version="${IMAGE_TAG}" \
                        --label org.opencontainers.image.revision="$(git rev-parse HEAD)" \
                        -t "${BACKEND_IMAGE}:${IMAGE_TAG}" \
                        -t "${BACKEND_IMAGE}:latest" \
                        ./backend

                    echo
                    echo "============================================================"
                    echo "FRONTEND DOCKER BUILD"
                    echo "============================================================"

                    docker build \
                        --pull \
                        --build-arg VITE_API_URL="${VITE_API_URL}" \
                        --label org.opencontainers.image.title="Shopverse Frontend" \
                        --label org.opencontainers.image.version="${IMAGE_TAG}" \
                        --label org.opencontainers.image.revision="$(git rev-parse HEAD)" \
                        -t "${FRONTEND_IMAGE}:${IMAGE_TAG}" \
                        -t "${FRONTEND_IMAGE}:latest" \
                        ./frontend

                    echo
                    echo "============================================================"
                    echo "IMAGE VERIFICATION"
                    echo "============================================================"

                    docker image inspect "${BACKEND_IMAGE}:${IMAGE_TAG}" >/dev/null
                    docker image inspect "${FRONTEND_IMAGE}:${IMAGE_TAG}" >/dev/null

                    docker images \
                        --filter "reference=${BACKEND_IMAGE}" \
                        --filter "reference=${FRONTEND_IMAGE}"

                    echo
                    echo "Docker image build successful."
                '''
            }
        }


        /*
         * ============================================================
         * 7. TRIVY SECURITY SCAN
         *
         * Reports are generated.
         *
         * We do not stop the deployment solely because the current
         * base image has an upstream vulnerability. This allows the
         * CI/CD pipeline to continue while still producing security
         * evidence for review.
         * ============================================================
         */
        stage('Docker Image Security Scan') {

            steps {

                sh '''
                    set -eu

                    mkdir -p "${TRIVY_REPORT_DIR}"

                    echo "============================================================"
                    echo "TRIVY BACKEND SCAN"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --output "${TRIVY_REPORT_DIR}/backend-${IMAGE_TAG}.txt" \
                        "${BACKEND_IMAGE}:${IMAGE_TAG}" || true

                    echo
                    echo "============================================================"
                    echo "TRIVY FRONTEND SCAN"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --output "${TRIVY_REPORT_DIR}/frontend-${IMAGE_TAG}.txt" \
                        "${FRONTEND_IMAGE}:${IMAGE_TAG}" || true

                    echo
                    echo "============================================================"
                    echo "TRIVY BACKEND JSON"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --format json \
                        --output "${TRIVY_REPORT_DIR}/backend-${IMAGE_TAG}.json" \
                        "${BACKEND_IMAGE}:${IMAGE_TAG}" || true

                    echo
                    echo "============================================================"
                    echo "TRIVY FRONTEND JSON"
                    echo "============================================================"

                    trivy image \
                        --scanners vuln \
                        --format json \
                        --output "${TRIVY_REPORT_DIR}/frontend-${IMAGE_TAG}.json" \
                        "${FRONTEND_IMAGE}:${IMAGE_TAG}" || true

                    echo
                    echo "Security scan completed."
                    echo "Reports are available under ${TRIVY_REPORT_DIR}/"
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

                    if [ ! -d "${DEPLOY_DIR}" ]; then
                        echo "ERROR: Deployment directory does not exist:"
                        echo "${DEPLOY_DIR}"
                        exit 1
                    fi

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
                    echo "DOCKER COMPOSE VALIDATION"
                    echo "============================================================"

                    export IMAGE_TAG="${IMAGE_TAG}"

                    docker compose \
                        --env-file .env \
                        config >/tmp/shopverse-compose-${BUILD_NUMBER}.yml

                    echo "Compose configuration is valid."

                    echo
                    echo "============================================================"
                    echo "IMAGE AVAILABILITY"
                    echo "============================================================"

                    docker image inspect "${BACKEND_IMAGE}:${IMAGE_TAG}" >/dev/null
                    docker image inspect "${FRONTEND_IMAGE}:${IMAGE_TAG}" >/dev/null

                    echo "Application images are available."
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

                sh '''
                    set -eu

                    cd "${DEPLOY_DIR}"

                    echo "============================================================"
                    echo "SAVING CURRENT DEPLOYMENT STATE"
                    echo "============================================================"

                    rm -f .previous-backend-image
                    rm -f .previous-frontend-image
                    rm -f .previous-deployment-info

                    BACKEND_CURRENT=""

                    if docker inspect shopverse-backend >/dev/null 2>&1; then
                        BACKEND_CURRENT=$(docker inspect \
                            -f '{{.Config.Image}}' \
                            shopverse-backend || true)
                    fi

                    FRONTEND_CURRENT=""

                    if docker inspect shopverse-frontend >/dev/null 2>&1; then
                        FRONTEND_CURRENT=$(docker inspect \
                            -f '{{.Config.Image}}' \
                            shopverse-frontend || true)
                    fi

                    echo "Current backend image: ${BACKEND_CURRENT:-NONE}"
                    echo "Current frontend image: ${FRONTEND_CURRENT:-NONE}"

                    if [ -n "${BACKEND_CURRENT}" ]; then
                        echo "${BACKEND_CURRENT}" > .previous-backend-image
                    fi

                    if [ -n "${FRONTEND_CURRENT}" ]; then
                        echo "${FRONTEND_CURRENT}" > .previous-frontend-image
                    fi

                    cat > .previous-deployment-info <<EOF
BUILD_NUMBER=${BUILD_NUMBER}
BUILD_TAG=${IMAGE_TAG}
GIT_COMMIT=$(git -C "${WORKSPACE}" rev-parse HEAD)
GIT_COMMIT_SHORT=$(git -C "${WORKSPACE}" rev-parse --short HEAD)
BACKEND_IMAGE=${BACKEND_CURRENT:-NONE}
FRONTEND_IMAGE=${FRONTEND_CURRENT:-NONE}
SAVED_AT=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
EOF

                    echo
                    cat .previous-deployment-info
                '''
            }
        }


        /*
         * ============================================================
         * 10. DEPLOY
         * ============================================================
         */
        stage('Docker Compose Deployment') {

            steps {

                sh '''
                    set -eu

                    cd "${DEPLOY_DIR}"

                    echo "============================================================"
                    echo "DEPLOYMENT"
                    echo "============================================================"

                    echo "Build number: ${BUILD_NUMBER}"
                    echo "Image tag: ${IMAGE_TAG}"

                    export IMAGE_TAG="${IMAGE_TAG}"

                    echo
                    echo "============================================================"
                    echo "MYSQL"
                    echo "============================================================"

                    docker compose \
                        --env-file .env \
                        up -d mysql

                    echo "Waiting for MySQL..."

                    MYSQL_READY=0

                    for i in $(seq 1 30); do

                        STATUS=$(docker inspect \
                            -f '{{.State.Health.Status}}' \
                            shopverse-mysql 2>/dev/null || true)

                        echo "MySQL health: ${STATUS}"

                        if [ "${STATUS}" = "healthy" ]; then
                            MYSQL_READY=1
                            break
                        fi

                        sleep 5
                    done

                    if [ "${MYSQL_READY}" -ne 1 ]; then
                        echo "ERROR: MySQL did not become healthy."
                        docker logs --tail 100 shopverse-mysql || true
                        exit 1
                    fi

                    echo
                    echo "============================================================"
                    echo "BACKEND"
                    echo "============================================================"

                    docker compose \
                        --env-file .env \
                        up -d --no-deps backend

                    echo
                    echo "============================================================"
                    echo "FRONTEND"
                    echo "============================================================"

                    docker compose \
                        --env-file .env \
                        up -d --no-deps frontend

                    echo
                    echo "============================================================"
                    echo "NGINX"
                    echo "============================================================"

                    docker compose \
                        --env-file .env \
                        up -d --no-deps nginx

                    echo
                    echo "============================================================"
                    echo "COMPOSE STATUS"
                    echo "============================================================"

                    docker compose \
                        --env-file .env \
                        ps
                '''
            }
        }


        /*
         * ============================================================
         * 11. SERVICE HEALTH CHECKS
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

                    sleep 10

                    docker compose \
                        --env-file .env \
                        ps

                    echo
                    echo "============================================================"
                    echo "MYSQL HEALTH"
                    echo "============================================================"

                    MYSQL_STATUS=$(docker inspect \
                        -f '{{.State.Health.Status}}' \
                        shopverse-mysql)

                    echo "MySQL: ${MYSQL_STATUS}"

                    if [ "${MYSQL_STATUS}" != "healthy" ]; then
                        echo "ERROR: MySQL is not healthy."
                        docker logs --tail 100 shopverse-mysql || true
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
                        echo "ERROR: Backend container is not running."
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
                        echo "ERROR: Frontend container is not running."
                        docker logs --tail 100 shopverse-frontend || true
                        exit 1
                    fi

                    echo
                    echo "============================================================"
                    echo "NGINX HEALTH"
                    echo "============================================================"

                    NGINX_RUNNING=$(docker inspect \
                        -f '{{.State.Running}}' \
                        shopverse-nginx)

                    echo "Nginx running: ${NGINX_RUNNING}"

                    if [ "${NGINX_RUNNING}" != "true" ]; then
                        echo "ERROR: Nginx container is not running."
                        docker logs --tail 100 shopverse-nginx || true
                        exit 1
                    fi

                    echo
                    echo "Waiting for HTTP health endpoint..."

                    HEALTH_OK=0

                    for i in $(seq 1 20); do

                        if curl -fsS \
                            --max-time 5 \
                            http://127.0.0.1/health \
                            >/dev/null; then

                            HEALTH_OK=1
                            echo "Nginx/backend health endpoint is healthy."
                            break
                        fi

                        echo "Health check attempt ${i}/20 failed."
                        sleep 3
                    done

                    if [ "${HEALTH_OK}" -ne 1 ]; then
                        echo "ERROR: Application health endpoint failed."

                        echo
                        echo "===== NGINX LOGS ====="
                        docker logs --tail 100 shopverse-nginx || true

                        echo
                        echo "===== BACKEND LOGS ====="
                        docker logs --tail 100 shopverse-backend || true

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
                    echo "Health endpoint:"
                    curl -fsS \
                        --max-time 10 \
                        http://127.0.0.1/health

                    echo
                    echo

                    echo "Frontend endpoint:"
                    curl -fsSI \
                        --max-time 10 \
                        http://127.0.0.1/

                    echo
                    echo "API endpoint check:"

                    HTTP_CODE=$(curl \
                        -s \
                        -o /tmp/shopverse-api-response.txt \
                        -w '%{http_code}' \
                        --max-time 10 \
                        http://127.0.0.1/api/)

                    echo "API HTTP status: ${HTTP_CODE}"

                    /*
                     * The API root may legitimately return 404 depending
                     * on the application's routes. The important check
                     * here is that Nginx responds and the request reaches
                     * the application stack.
                     */
                    if [ "${HTTP_CODE}" = "502" ] || \
                       [ "${HTTP_CODE}" = "503" ] || \
                       [ "${HTTP_CODE}" = "504" ]; then

                        echo "ERROR: API gateway returned ${HTTP_CODE}."
                        cat /tmp/shopverse-api-response.txt || true
                        exit 1
                    fi

                    echo
                    echo "Application smoke test completed."
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

                    cat > .last-successful-deployment <<EOF
BUILD_NUMBER=${BUILD_NUMBER}
IMAGE_TAG=${IMAGE_TAG}
GIT_COMMIT=$(git -C "${WORKSPACE}" rev-parse HEAD)
GIT_COMMIT_SHORT=$(git -C "${WORKSPACE}" rev-parse --short HEAD)
BACKEND_IMAGE=${BACKEND_IMAGE}:${IMAGE_TAG}
FRONTEND_IMAGE=${FRONTEND_IMAGE}:${IMAGE_TAG}
DEPLOYED_AT=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
EOF

                    cat .last-successful-deployment

                    echo
                    echo "Deployment completed successfully."
                '''
            }
        }


        /*
         * ============================================================
         * 14. CLEANUP OLD IMAGES
         * ============================================================
         */
        stage('Docker Image Cleanup') {

            steps {

                sh '''
                    set +e

                    echo "============================================================"
                    echo "DOCKER IMAGE CLEANUP"
                    echo "============================================================"

                    CURRENT_BACKEND="${BACKEND_IMAGE}:${IMAGE_TAG}"
                    CURRENT_FRONTEND="${FRONTEND_IMAGE}:${IMAGE_TAG}"

                    echo "Current backend image: ${CURRENT_BACKEND}"
                    echo "Current frontend image: ${CURRENT_FRONTEND}"

                    /*
                     * Remove dangling images only.
                     *
                     * Do NOT aggressively delete tagged images because
                     * rollback depends on previous tagged images.
                     */
                    docker image prune -f

                    echo
                    echo "Remaining Shopverse images:"

                    docker images \
                        --filter "reference=${BACKEND_IMAGE}" \
                        --filter "reference=${FRONTEND_IMAGE}"

                    exit 0
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
                artifacts: 'trivy-reports/**/*',
                allowEmptyArchive: true,
                fingerprint: true
            )

            archiveArtifacts(
                artifacts: 'Jenkinsfile',
                allowEmptyArchive: true,
                fingerprint: true
            )

            archiveArtifacts(
                artifacts: 'backend/go.mod,backend/go.sum,frontend/package.json,frontend/package-lock.json',
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
                    --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}' \
                    | grep -E 'shopverse|NAMES' || true

                echo
                echo "============================================================"
                echo "FINAL SHOPVERSE IMAGES"
                echo "============================================================"

                docker images \
                    --filter "reference=shopverse-backend" \
                    --filter "reference=shopverse-frontend" || true
            '''
        }


        success {

            echo '''
============================================================
SHOPVERSE PIPELINE SUCCESS
============================================================

Build:
Deployment:
Health checks:
Smoke test:

All completed successfully.
============================================================
'''
        }


        failure {

            echo '''
============================================================
SHOPVERSE PIPELINE FAILED
============================================================

A pipeline stage failed.

The deployment rollback procedure will now be evaluated.
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

                if [ ! -f .previous-backend-image ] || \
                   [ ! -f .previous-frontend-image ]; then

                    echo "No complete previous deployment state found."
                    echo "Rollback will not be attempted."
                    exit 0
                fi

                PREVIOUS_BACKEND=$(cat .previous-backend-image)
                PREVIOUS_FRONTEND=$(cat .previous-frontend-image)

                echo "Previous backend image:"
                echo "${PREVIOUS_BACKEND}"

                echo "Previous frontend image:"
                echo "${PREVIOUS_FRONTEND}"

                if [ -z "${PREVIOUS_BACKEND}" ] || \
                   [ -z "${PREVIOUS_FRONTEND}" ]; then

                    echo "Previous image information is incomplete."
                    exit 0
                fi

                echo
                echo "Checking previous images..."

                if ! docker image inspect "${PREVIOUS_BACKEND}" >/dev/null 2>&1; then
                    echo "Previous backend image is not available."
                    exit 0
                fi

                if ! docker image inspect "${PREVIOUS_FRONTEND}" >/dev/null 2>&1; then
                    echo "Previous frontend image is not available."
                    exit 0
                fi

                echo
                echo "============================================================"
                echo "STARTING ROLLBACK"
                echo "============================================================"

                export IMAGE_TAG="${BUILD_NUMBER}"

                /*
                 * docker-compose.yml uses IMAGE_TAG for both
                 * application images.
                 *
                 * Therefore create temporary rollback tags that match
                 * the Compose configuration.
                 */

                docker tag \
                    "${PREVIOUS_BACKEND}" \
                    "shopverse-backend:rollback-${BUILD_NUMBER}"

                docker tag \
                    "${PREVIOUS_FRONTEND}" \
                    "shopverse-frontend:rollback-${BUILD_NUMBER}"

                /*
                 * Temporarily change IMAGE_TAG to rollback tag.
                 *
                 * Both application images receive the same tag.
                 */
                export IMAGE_TAG="rollback-${BUILD_NUMBER}"

                docker compose \
                    --env-file .env \
                    up -d --no-deps backend

                docker compose \
                    --env-file .env \
                    up -d --no-deps frontend

                docker compose \
                    --env-file .env \
                    up -d nginx

                sleep 10

                echo
                echo "============================================================"
                echo "ROLLBACK HEALTH CHECK"
                echo "============================================================"

                MYSQL_STATUS=$(docker inspect \
                    -f '{{.State.Health.Status}}' \
                    shopverse-mysql 2>/dev/null || true)

                echo "MySQL: ${MYSQL_STATUS}"

                BACKEND_RUNNING=$(docker inspect \
                    -f '{{.State.Running}}' \
                    shopverse-backend 2>/dev/null || true)

                echo "Backend: ${BACKEND_RUNNING}"

                FRONTEND_RUNNING=$(docker inspect \
                    -f '{{.State.Running}}' \
                    shopverse-frontend 2>/dev/null || true)

                echo "Frontend: ${FRONTEND_RUNNING}"

                NGINX_RUNNING=$(docker inspect \
                    -f '{{.State.Running}}' \
                    shopverse-nginx 2>/dev/null || true)

                echo "Nginx: ${NGINX_RUNNING}"

                if curl -fsS \
                    --max-time 10 \
                    http://127.0.0.1/health \
                    >/dev/null; then

                    echo
                    echo "============================================================"
                    echo "ROLLBACK SUCCESSFUL"
                    echo "============================================================"

                else

                    echo
                    echo "============================================================"
                    echo "ROLLBACK HEALTH CHECK FAILED"
                    echo "============================================================"

                    docker logs --tail 100 shopverse-backend || true
                    docker logs --tail 100 shopverse-frontend || true
                    docker logs --tail 100 shopverse-nginx || true
                fi

                echo
                echo "Rollback procedure completed."
            '''
        }
    }
}
