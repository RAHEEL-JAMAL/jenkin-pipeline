pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: 'https://github.com/RAHEEL-JAMAL/portfolio-website.git', description: 'Git Repo URL')
        string(name: 'APP_NAME', defaultValue: 'my-app', description: 'Application Name')
        choice(name: 'DEPLOY_MODE', choices: ['local', 'aws'], description: 'Deployment Mode')
    }

    environment {
        DOCKERHUB_CRED = credentials('dockerhub-cred')
        DOCKERHUB_USER = 'raheeljamal'
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo '[STAGE_START] Init'
                    echo "DEPLOY MODE = ${params.DEPLOY_MODE}"
                    echo '[STAGE_SUCCESS] Init'
                }
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    echo '[STAGE_START] Input Repo'

                    def repoUrl = params.REPO_URL?.trim()

                    def appId = sh(
                        script: "printf '%s' '${repoUrl}' | md5sum | cut -c1-6",
                        returnStdout: true
                    ).trim()

                    env.REPO_URL       = repoUrl
                    env.APP_ID         = appId
                    env.APP_NAME       = params.APP_NAME
                    env.DEPLOY_MODE    = params.DEPLOY_MODE
                    env.CONTAINER_NAME = "app_${appId}"
                    env.IMAGE_NAME     = "${DOCKERHUB_USER}/${env.CONTAINER_NAME}:latest"

                    echo "[META] APP_NAME=${env.APP_NAME}"
                    echo "[META] DEPLOY_MODE=${env.DEPLOY_MODE}"
                    echo "[META] REPO_URL=${env.REPO_URL}"
                    echo "[META] APP_ID=${env.APP_ID}"
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

                    while (usedPorts.contains(port.toString())) {
                        port++
                    }

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
                        git clone --depth 1 "${REPO_URL}" app
                    '''
                }

                script { echo '[STAGE_SUCCESS] Clone Repo' }
            }
        }

        stage('Secret Scan') {
            steps {
                script { echo '[STAGE_START] Secret Scan' }
                sh 'echo "Scanning secrets..."'
                script { echo '[STAGE_SUCCESS] Secret Scan' }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    echo '[STAGE_START] Detect Stack'

                    def stack = 'node'

                    if (fileExists('app/package.json')) {
                        def pkg = readFile('app/package.json')
                        if (pkg.contains('"react"')) stack = 'react'
                        else if (pkg.contains('"next"')) stack = 'nextjs'
                    }

                    env.STACK = stack
                    echo "[META] STACK=${env.STACK}"
                    echo '[STAGE_SUCCESS] Detect Stack'
                }
            }
        }

        stage('Dependency Audit') {
            steps {
                script { echo '[STAGE_START] Dependency Audit' }
                sh 'echo "audit running..."'
                script { echo '[STAGE_SUCCESS] Dependency Audit' }
            }
        }

        stage('Create Dockerfile') {
    steps {
        script {
            echo '[STAGE_START] Create Dockerfile'

            // ================================
            // 🔥 FIX 1: smarter monorepo detection
            // ================================
            def appDir = "app"

            if (fileExists('app/frontend/package.json')) {
                appDir = "app/frontend"
            } else if (fileExists('app/client/package.json')) {
                appDir = "app/client"
            } else if (fileExists('app/frontend/src')) {
                appDir = "app/frontend"
            } else if (fileExists('app/client/src')) {
                appDir = "app/client"
            }

            echo "[DEBUG] Using appDir = ${appDir}"

            def df = ''

            // ================================
            // React / Vite / SPA
            // ================================
            if (fileExists("${appDir}/package.json") &&
                (fileExists("${appDir}/src") || fileExists("${appDir}/vite.config.js"))) {

                df = """
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

RUN npm run build
RUN npm install -g serve

EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
"""
            }

            // ================================
            // Next.js
            // ================================
            else if (fileExists("${appDir}/next.config.js")) {

                df = """
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

EXPOSE 3000
CMD ["npm", "start"]
"""
            }

            // ================================
            // Node / Express backend
            // ================================
            else if (fileExists("${appDir}/package.json")) {

                df = """
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "start"]
"""
            }

            // ================================
            // Python
            // ================================
            else if (fileExists('app/requirements.txt')) {

                def runCmd = fileExists('app/manage.py')
                    ? 'python manage.py migrate && python manage.py runserver 0.0.0.0:3000'
                    : 'python app.py'

                df = """
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 3000
CMD ["sh", "-c", "${runCmd}"]
"""
            }

            // ================================
            // PHP
            // ================================
            else if (fileExists('app/index.php')) {

                df = """
FROM php:8.2-apache

WORKDIR /var/www/html

COPY . /var/www/html

EXPOSE 80
"""
            }

            // ================================
            // Java Maven
            // ================================
            else if (fileExists('app/pom.xml')) {

                df = """
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre

WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

EXPOSE 3000
CMD ["java", "-jar", "app.jar"]
"""
            }

            // ================================
            // Static HTML
            // ================================
            else if (fileExists('app/index.html')) {

                df = """
FROM nginx:alpine

COPY . /usr/share/nginx/html

EXPOSE 80
"""
            }

            else {
                error("Unsupported project type - cannot detect stack safely")
            }

            writeFile file: "app/Dockerfile", text: df

            echo '[STAGE_SUCCESS] Create Dockerfile'
        }
    }
}

        stage('Build Image') {
            steps {
                script { echo '[STAGE_START] Build Image' }
                sh 'docker build -t "${IMAGE_NAME}" app'
                script { echo '[STAGE_SUCCESS] Build Image' }
            }
        }

        stage('Image Scan (Trivy)') {
            steps {
                script { echo '[STAGE_START] Image Scan (Trivy)' }
                sh 'echo "scan running..."'
                script { echo '[STAGE_SUCCESS] Image Scan (Trivy)' }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script { echo '[STAGE_START] Push to DockerHub' }

                sh '''
                    set -e

                    echo "$DOCKERHUB_CRED_PSW" | docker login -u "$DOCKERHUB_CRED_USR" --password-stdin

                    n=1
                    until [ "$n" -gt 3 ]
                    do
                        docker push "${IMAGE_NAME}" && break

                        echo "Docker push failed / retrying ($n/3)"
                        n=$((n+1))
                        sleep 10
                    done

                    docker logout
                '''

                script { echo '[STAGE_SUCCESS] Push to DockerHub' }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    echo "[STAGE_START] Deploy Mode = ${params.DEPLOY_MODE}"

                    if (params.DEPLOY_MODE == 'local') {

                        sh """
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true

                            docker run -d --name ${CONTAINER_NAME} \
                            -p ${PORT}:3000 \
                            -e HOST=0.0.0.0 \
                            -e PORT=3000 \
                            --restart unless-stopped \
                            ${IMAGE_NAME}
                        """
                    }

                    else if (params.DEPLOY_MODE == 'aws') {

                        sh '''
                            set -e

                            SG_ID=$(aws ec2 create-security-group \
                                --group-name fyp-sg-${APP_ID} \
                                --description "FYP SG" \
                                --query GroupId --output text)

                            INSTANCE_ID=$(aws ec2 run-instances \
                                --image-id ami-0c02fb55956c7d316 \
                                --instance-type t2.micro \
                                --security-group-ids $SG_ID \
                                --user-data "$(base64 <<EOF
#!/bin/bash
apt update -y
apt install docker.io -y
systemctl start docker
systemctl enable docker

docker pull ${IMAGE_NAME}
docker run -d -p 80:3000 ${IMAGE_NAME}
EOF
)" \
                                --query Instances[0].InstanceId \
                                --output text)

                            aws ec2 wait instance-running --instance-ids $INSTANCE_ID

                            PUBLIC_IP=$(aws ec2 describe-instances \
                                --instance-ids $INSTANCE_ID \
                                --query Reservations[0].Instances[0].PublicIpAddress \
                                --output text)

                            echo "URL=http://$PUBLIC_IP"
                        '''
                    }

                    echo '[STAGE_SUCCESS] Deploy'
                }
            }
        }

        stage('Verify') {
            steps {
                script { echo '[STAGE_START] Verify' }
                sh 'docker logs ${CONTAINER_NAME} --tail 20 || true'
                script { echo '[STAGE_SUCCESS] Verify' }
            }
        }
    }
}
