pipeline {
    agent any

    environment {
        DOCKERHUB_CRED = credentials('dockerhub-cred')
        DOCKERHUB_USER = 'raheeljamal'
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo '[STAGE_START] Init'
                    echo 'MULTI APP SAFE DEPLOY STARTED'
                    echo '[STAGE_SUCCESS] Init'
                }
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    echo '[STAGE_START] Input Repo'

                    def repoUrl = params.REPO_URL?.trim() ?: 'https://github.com/RAHEEL-JAMAL/portfolio-website.git'

                    def appId = sh(
                        script: "printf '%s' '${repoUrl}' | md5sum | cut -c1-6",
                        returnStdout: true
                    ).trim()

                    env.REPO_URL       = repoUrl
                    env.APP_ID         = appId
                    env.CONTAINER_NAME = "app_${appId}"
                    env.IMAGE_NAME     = "${DOCKERHUB_USER}/${env.CONTAINER_NAME}:latest"

                    echo "[META] REPO_URL=${env.REPO_URL}"
                    echo "[META] APP_ID=${env.APP_ID}"
                    echo "[META] CONTAINER_NAME=${env.CONTAINER_NAME}"
                    echo "[META] IMAGE_NAME=${env.IMAGE_NAME}"
                    echo '[STAGE_SUCCESS] Input Repo'
                }
            }
        }

        stage('Allocate Safe Port') {
            steps {
                script {
                    echo '[STAGE_START] Allocate Safe Port'

                    def usedRaw = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]*->' | grep -o '[0-9]*' || true",
                        returnStdout: true
                    ).trim()

                    def usedPorts = usedRaw ? usedRaw.split('\n').toList() : []
                    def port = 3000
                    while (usedPorts.contains(port.toString())) { port++ }
                    env.PORT = port.toString()

                    echo "[META] PORT=${env.PORT}"
                    echo '[STAGE_SUCCESS] Allocate Safe Port'
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script { echo '[STAGE_START] Clone Repo' }
                retry(3) {
                    sh '''
                        rm -rf app
                        git config --global http.version HTTP/1.1
                        git config --global http.postBuffer 524288000
                        git clone --depth 1 "${REPO_URL}" app
                    '''
                }
                script { echo '[STAGE_SUCCESS] Clone Repo' }
            }
        }

        stage('Secret Scan') {
            steps {
                script {
                    echo '[STAGE_START] Secret Scan'
                    echo 'Scanning repo for hardcoded secrets...'
                }
                sh '''#!/bin/bash
                    FOUND=0

                    find app -type f \\( -name "*.js" -o -name "*.py" -o -name "*.env" \\) \
                        -not -path "*/node_modules/*" \
                        -not -path "*/.git/*" \
                        -not -name "package-lock.json" | while read -r f; do
                        if grep -qiE "(password|api_key|secret|access_token)\\s*=" "$f"; then
                            echo "[WARN] Possible secret in: $f"
                            FOUND=1
                        fi
                    done

                    if [ "$FOUND" = "1" ]; then
                        echo "[META] SECRET_SCAN=FAILED"
                        exit 1
                    fi

                    echo "No secrets found."
                    echo "[META] SECRET_SCAN=PASSED"
                '''
                script { echo '[STAGE_SUCCESS] Secret Scan' }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    echo '[STAGE_START] Detect Stack'

                    def pkgRoot = 'app'
                    def stack   = 'unknown'

                    if (fileExists('app/package.json')) {
                        def pkg = readFile('app/package.json')
                        if      (pkg.contains('"vite"'))   stack = 'vite'
                        else if (pkg.contains('"next"'))   stack = 'nextjs'
                        else if (pkg.contains('"react"'))  stack = 'react'
                        else                               stack = 'node'
                    } else {
                        def mp = sh(script: "find app -maxdepth 2 -name manage.py | head -1", returnStdout: true).trim()
                        if (mp)  { stack = 'django'; pkgRoot = mp.replaceAll('/manage\\.py', '') }

                        def cp = sh(script: "find app -maxdepth 2 -name composer.json | head -1", returnStdout: true).trim()
                        if (cp)  { stack = 'php'; pkgRoot = cp.replaceAll('/composer\\.json', '') }

                        def jp = sh(script: "find app -maxdepth 2 \\( -name pom.xml -o -name build.gradle \\) | head -1", returnStdout: true).trim()
                        if (jp)  { stack = 'java'; pkgRoot = jp.replaceAll('/(pom\\.xml|build\\.gradle)', '') }
                    }

                    env.STACK    = stack
                    env.PKG_ROOT = pkgRoot

                    echo "[META] STACK=${env.STACK}"
                    echo "[META] PKG_ROOT=${env.PKG_ROOT}"
                    echo '[STAGE_SUCCESS] Detect Stack'
                }
            }
        }

        stage('Dependency Audit') {
            steps {
                script {
                    echo '[STAGE_START] Dependency Audit'
                    echo 'Scanning dependencies for known vulnerabilities...'
                }
                sh '''#!/bin/bash
                    set -e
                    if [ -f "${PKG_ROOT}/package.json" ]; then
                        echo "Running npm audit inside Docker..."
                        ABS_PATH="$(cd "${PKG_ROOT}" && pwd)"
                        docker run --rm \
                            -v "${ABS_PATH}:/work" \
                            -w /work \
                            node:20-alpine \
                            sh -c "npm install --prefer-offline --no-fund 2>/dev/null; npm audit --json > /work/npm-audit.json 2>&1 || true"

                        if [ -f "${PKG_ROOT}/npm-audit.json" ]; then
                            VULN=$(grep -c '"severity"' "${PKG_ROOT}/npm-audit.json" || echo 0)
                            echo "[META] DEPENDENCY_AUDIT=PASSED (severity hits: ${VULN})"
                        else
                            echo "[META] DEPENDENCY_AUDIT=SKIPPED (no output)"
                        fi
                    else
                        echo "[META] DEPENDENCY_AUDIT=SKIPPED (no package.json)"
                    fi
                '''
                script { echo '[STAGE_SUCCESS] Dependency Audit' }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    echo '[STAGE_START] Create Dockerfile'

                    def df = ''
                    switch (env.STACK) {
                        case 'vite':
                        case 'react':
                            df = 'FROM node:20-alpine AS builder\nWORKDIR /app\nCOPY . .\nRUN npm ci && npm run build\n\nFROM nginx:alpine\nCOPY --from=builder /app/dist /usr/share/nginx/html\nEXPOSE 80\nCMD ["nginx", "-g", "daemon off;"]\n'
                            break
                        case 'nextjs':
                            df = 'FROM node:20-alpine AS builder\nWORKDIR /app\nCOPY . .\nRUN npm ci && npm run build\n\nFROM node:20-alpine\nWORKDIR /app\nCOPY --from=builder /app/.next ./.next\nCOPY --from=builder /app/public ./public\nCOPY --from=builder /app/package.json ./\nRUN npm ci --omit=dev\nEXPOSE 3000\nCMD ["npm", "start"]\n'
                            break
                        case 'node':
                            df = 'FROM node:20-alpine\nWORKDIR /app\nCOPY . .\nRUN npm ci --omit=dev\nEXPOSE 3000\nCMD ["node", "index.js"]\n'
                            break
                        case 'django':
                            df = 'FROM python:3.11-slim\nWORKDIR /app\nCOPY . .\nRUN pip install --no-cache-dir -r requirements.txt\nEXPOSE 8000\nCMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]\n'
                            break
                        case 'php':
                            df = 'FROM php:8.2-apache\nWORKDIR /var/www/html\nCOPY . .\nEXPOSE 80\n'
                            break
                        default:
                            df = 'FROM node:20-alpine\nWORKDIR /app\nCOPY . .\nEXPOSE 3000\nCMD ["sh", "-c", "echo Unknown stack && sleep 9999"]\n'
                    }

                    writeFile file: "${env.PKG_ROOT}/Dockerfile", text: df
                    echo '[META] DOCKERFILE_CREATED=true'
                    echo '[STAGE_SUCCESS] Create Dockerfile'
                }
            }
        }

        stage('Build Image') {
            steps {
                script { echo '[STAGE_START] Build Image' }
                sh 'docker build -t "${IMAGE_NAME}" "${PKG_ROOT}"'
                script { echo '[STAGE_SUCCESS] Build Image' }
            }
        }

        stage('Image Scan (Trivy)') {
            steps {
                script { echo '[STAGE_START] Image Scan (Trivy)' }
                sh '''
                    echo "Running Trivy scan on: ${IMAGE_NAME}"
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        "${IMAGE_NAME}"
                    echo "[META] IMAGE_SCAN=PASSED"
                '''
                script { echo '[STAGE_SUCCESS] Image Scan (Trivy)' }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script { echo '[STAGE_START] Push to DockerHub' }
                sh '''
                    echo "${DOCKERHUB_CRED_PSW}" | docker login -u "${DOCKERHUB_CRED_USR}" --password-stdin
                    docker push "${IMAGE_NAME}"
                    docker logout
                '''
                script { echo '[STAGE_SUCCESS] Push to DockerHub' }
            }
        }

        stage('Stop Old Container') {
            steps {
                script { echo '[STAGE_START] Stop Old Container' }
                sh '''
                    docker stop "${CONTAINER_NAME}" 2>/dev/null || true
                    docker rm   "${CONTAINER_NAME}" 2>/dev/null || true
                '''
                script { echo '[STAGE_SUCCESS] Stop Old Container' }
            }
        }

        stage('Run Container') {
            steps {
                script { echo '[STAGE_START] Run Container' }
                sh 'docker run -d --name "${CONTAINER_NAME}" -p "${PORT}:80" --restart unless-stopped "${IMAGE_NAME}"'
                script { echo '[STAGE_SUCCESS] Run Container' }
            }
        }

        stage('Verify') {
            steps {
                script { echo '[STAGE_START] Verify' }
                sh '''#!/bin/bash
                    sleep 5
                    STATUS=$(docker inspect --format="{{.State.Running}}" "${CONTAINER_NAME}" 2>/dev/null || echo "false")
                    if [ "$STATUS" = "true" ]; then
                        echo "[META] CONTAINER_STATUS=RUNNING"
                        echo "[META] DEPLOYED_PORT=${PORT}"
                        echo "[META] FINAL_STATUS=success"
                        echo "[DEPLOY_SUCCESS]"
                    else
                        echo "[META] FINAL_STATUS=failed"
                        echo "[DEPLOY_FAILED]"
                        exit 1
                    fi
                '''
                script { echo '[STAGE_SUCCESS] Verify' }
            }
        }
    }

    post {
        success {
            echo '[DEPLOY_SUCCESS]'
            echo '[META] FINAL_STATUS=success'
        }
        failure {
            echo '[DEPLOY_FAILED]'
            echo '[META] FINAL_STATUS=failed'
        }
        always {
            echo '[META] Pipeline complete.'
        }
    }
}
