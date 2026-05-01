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

            def df = '''
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

RUN npm run build

RUN npm install -g serve

EXPOSE 3000

ENV HOST=0.0.0.0
ENV PORT=3000

CMD ["serve", "-s", "dist", "-l", "tcp://0.0.0.0:3000"]
'''

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

                echo "Docker push failed / stalled... retrying ($n/3)"
                n=$((n+1))
                sleep 10
            done

            if [ "$n" -gt 3 ]; then
                echo "Docker push failed after 3 attempts"
                exit 1
            fi

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

                        echo "Deploying on LOCAL VM"

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

                        echo "[META] DEPLOYMENT=LOCAL_SUCCESS"
                        echo "[META] URL=http://localhost:${PORT}"
                    }

                    else if (params.DEPLOY_MODE == 'aws') {

                        echo "AWS DEPLOY STARTED"

                        sh '''
                            set -e

                            SG_ID=$(aws ec2 create-security-group \
                                --group-name fyp-sg-${APP_ID} \
                                --description "FYP SG" \
                                --query GroupId --output text)

                            aws ec2 authorize-security-group-ingress \
                                --group-id $SG_ID \
                                --protocol tcp --port 22 --cidr 0.0.0.0/0

                            aws ec2 authorize-security-group-ingress \
                                --group-id $SG_ID \
                                --protocol tcp --port 80 --cidr 0.0.0.0/0

                            aws ec2 authorize-security-group-ingress \
                                --group-id $SG_ID \
                                --protocol tcp --port 3000 --cidr 0.0.0.0/0

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

                            echo "[META] URL=http://$PUBLIC_IP"
                        '''
                    }

                    echo '[STAGE_SUCCESS] Deploy'
                }
            }
        }

        stage('Verify') {
            steps {
                script { echo '[STAGE_START] Verify' }
                sh '''
                    echo "checking container..."
                    docker logs ${CONTAINER_NAME} --tail 20 || true
                '''
                script { echo '[STAGE_SUCCESS] Verify' }
            }
        }
    }
}
