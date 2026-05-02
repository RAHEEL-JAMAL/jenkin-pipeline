pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'GitHub repository URL')
        string(name: 'APP_NAME', defaultValue: '', description: 'Application name (slug)')
        choice(name: 'DEPLOY_MODE', choices: ['local', 'aws'], description: 'Deployment target')
    }

    environment {
        DOCKERHUB_CRED = credentials('dockerhub-cred')
        VM_IP = '192.168.122.127'
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo "=== INIT ==="

                    if (!params.REPO_URL?.trim() || !params.APP_NAME?.trim()) {
                        error("REPO_URL and APP_NAME parameters are required.")
                    }

                    env.WORK_DIR   = "/tmp/deployments/${params.APP_NAME}"
                    env.IMAGE_NAME = "${env.DOCKERHUB_CRED_USR}/${params.APP_NAME}:latest"

                    sh "rm -rf '${env.WORK_DIR}' && mkdir -p '${env.WORK_DIR}'"

                    echo "APP_NAME   : ${params.APP_NAME}"
                    echo "REPO_URL   : ${params.REPO_URL}"
                    echo "DEPLOY_MODE: ${params.DEPLOY_MODE}"
                }
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    env.REPO_URL_CLEAN = params.REPO_URL.trim()
                }
            }
        }

        stage('Allocate Safe Port') {
            when { expression { params.DEPLOY_MODE == 'local' } }
            steps {
                script {
                    def port = sh(
                        script: '''
python3 -c "
import socket, sys
for p in range(3000, 4001):
    try:
        s = socket.socket()
        s.bind(('0.0.0.0', p))
        s.close()
        print(p)
        sys.exit(0)
    except:
        pass
print('NO_PORT')
sys.exit(1)
"
                        ''',
                        returnStdout: true
                    ).trim()

                    if (port == 'NO_PORT') error("No free port found")

                    env.ALLOCATED_PORT = port
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh "git clone --depth=1 '${env.REPO_URL_CLEAN}' '${env.WORK_DIR}/repo'"
            }
        }

        stage('Secret Scan') {
            steps {
                script {
                    def secrets = sh(script: """
                        grep -rIlE '(password|secret|api_key|token|access_key|private_key)' \
                        '${env.WORK_DIR}/repo' 2>/dev/null || true
                    """, returnStdout: true).trim()

                    if (secrets) {
                        echo "WARNING secrets found:\n${secrets}"
                    }
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    def repoDir = "${env.WORK_DIR}/repo"
                    def stacks = []

                    def exists = { sh(script: "test -e '$it' && echo yes || echo no", returnStdout: true).trim() == 'yes' }

                    def managePy = sh(script: "find ${repoDir} -name manage.py | head -1", returnStdout: true).trim()
                    def appPy = sh(script: "find ${repoDir} -name app.py | head -1", returnStdout: true).trim()
                    def mainPy = sh(script: "find ${repoDir} -name main.py | head -1", returnStdout: true).trim()

                    if (managePy) {
                        stacks.add([type: 'django', dir: sh(script: "dirname ${managePy}", returnStdout: true).trim(), port: '8000'])
                    } else if (appPy || mainPy) {
                        stacks.add([type: 'flask', dir: sh(script: "dirname ${appPy ?: mainPy}", returnStdout: true).trim(), port: '8000'])
                    } else {
                        stacks.add([type: 'node', dir: repoDir, port: '3000'])
                    }

                    env.STACKS_SERIALIZED = stacks.collect { "${it.type}:${it.dir}:${it.port}" }.join('|')
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    def stacksRaw = env.STACKS_SERIALIZED.split('\\|').collect {
                        def p = it.split(':')
                        [type: p[0], dir: p[1], port: p[2]]
                    }

                    stacksRaw.each { s ->

                        def dockerfile = ""

                        if (s.type == 'django') {
                            dockerfile = """
FROM python:3.11-slim
WORKDIR /app

COPY . .
RUN pip install django gunicorn

# FIXED: escaped \$ to avoid Jenkins Groovy error
RUN echo '#!/bin/sh' > start.sh
RUN echo "gunicorn --bind 0.0.0.0:${s.port} \$(cat /wsgi.txt).wsgi:application" >> start.sh
RUN chmod +x start.sh

EXPOSE ${s.port}
CMD ["sh", "start.sh"]
"""
                        } else if (s.type == 'flask') {
                            dockerfile = """
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install flask gunicorn
EXPOSE ${s.port}
CMD ["gunicorn", "--bind", "0.0.0.0:${s.port}", "app:app"]
"""
                        } else {
                            dockerfile = """
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE ${s.port}
CMD ["node", "index.js"]
"""
                        }

                        writeFile file: "${s.dir}/Dockerfile", text: dockerfile
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    def stacksRaw = env.STACKS_SERIALIZED.split('\\|').collect {
                        def p = it.split(':')
                        [type: p[0], dir: p[1], port: p[2]]
                    }

                    stacksRaw.each { s ->
                        sh """
                            docker build -t ${DOCKERHUB_CRED_USR}/${params.APP_NAME}:${s.type} -f ${s.dir}/Dockerfile ${s.dir}
                        """
                    }
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    sh "echo '${DOCKERHUB_CRED_PSW}' | docker login -u '${DOCKERHUB_CRED_USR}' --password-stdin"

                    sh """
                        docker images | grep ${params.APP_NAME} | awk '{print \$1\":\"\$2}' | while read img; do
                            docker push \$img
                        done
                    """

                    sh "docker logout"
                }
            }
        }

        // ───────── AWS ONLY (FIXED) ─────────
        stage('AWS Deploy') {
            when { expression { params.DEPLOY_MODE == 'aws' } }
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        echo "AWS deploy running safely now"
                        // your AWS logic here
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploy stage (${params.DEPLOY_MODE})"
            }
        }
    }

    post {
        always {
            echo "Pipeline finished"
            sh "rm -rf ${env.WORK_DIR} || true"
        }
    }
}
