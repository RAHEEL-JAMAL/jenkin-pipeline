pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL',    defaultValue: '', description: 'GitHub repository URL')
        string(name: 'APP_NAME',    defaultValue: '', description: 'Application name (slug)')
        choice(name: 'DEPLOY_MODE', choices: ['local', 'aws'], description: 'Deployment target')
    }

    environment {
        DOCKERHUB_CRED        = credentials('dockerhub-cred')
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_REGION            = 'us-east-1'
        VM_IP                 = '192.168.122.127'
    }

    stages {

        // ─────────────────────────────────────────────────────────────────
        // STAGE 1 — INIT
        // ─────────────────────────────────────────────────────────────────
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
                    echo "WORK_DIR   : ${env.WORK_DIR}"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 2 — INPUT REPO
        // ─────────────────────────────────────────────────────────────────
        stage('Input Repo') {
            steps {
                script {
                    echo "=== INPUT REPO ==="
                    env.REPO_URL_CLEAN = params.REPO_URL.trim()
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 3 — ALLOCATE SAFE PORT
        // ─────────────────────────────────────────────────────────────────
        stage('Allocate Safe Port') {
            when { expression { params.DEPLOY_MODE == 'local' } }
            steps {
                script {
                    echo "=== ALLOCATE SAFE PORT ==="

                    def port = sh(
                        script: '''
python3 -c "
import socket, sys
for p in range(3000, 4001):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind(('0.0.0.0', p))
        s.close()
        print(p)
        sys.exit(0)
    except OSError:
        pass
print('NO_PORT')
sys.exit(1)
"
                        ''',
                        returnStdout: true
                    ).trim()

                    if (port == 'NO_PORT') {
                        error("No free port found in range 3000-4000")
                    }

                    env.ALLOCATED_PORT = port
                    echo "Allocated port: ${env.ALLOCATED_PORT}"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 4 — CLONE REPO
        // ─────────────────────────────────────────────────────────────────
        stage('Clone Repo') {
            steps {
                script {
                    echo "=== CLONE REPO ==="
                    sh "git clone --depth=1 '${env.REPO_URL_CLEAN}' '${env.WORK_DIR}/repo'"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 5 — SECRET SCAN
        // ─────────────────────────────────────────────────────────────────
        stage('Secret Scan') {
            steps {
                script {
                    echo "=== SECRET SCAN ==="

                    def secrets = sh(
                        script: """
                            grep -rIlE '(password|secret|api_key|token|access_key|private_key)\\s*[:=]\\s*[A-Za-z0-9+/]{8,}' \
                            '${env.WORK_DIR}/repo' \
                            --include='*.env' --include='*.json' \
                            --include='*.py' --include='*.js' \
                            --include='*.php' --include='*.java' \
                            2>/dev/null || true
                        """,
                        returnStdout: true
                    ).trim()

                    if (secrets) {
                        echo "WARNING: Potential secrets found in:\n${secrets}"
                    } else {
                        echo "OK: No obvious secrets detected."
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 6 — DETECT STACK
        // ─────────────────────────────────────────────────────────────────
        stage('Detect Stack') {
            steps {
                script {
                    echo "=== DETECT STACK ==="

                    def repoDir = "${env.WORK_DIR}/repo"
                    def stacks = []

                    def exists = { String path ->
                        sh(script: "test -e '${path}' && echo yes || echo no", returnStdout: true).trim() == 'yes'
                    }

                    def findFirst = { String dir, String name, int depth ->
                        sh(script: "find '${dir}' -maxdepth ${depth} -name '${name}' 2>/dev/null | head -1", returnStdout: true).trim()
                    }

                    def grepQ = { String file, String pattern ->
                        sh(script: "grep -qiE '${pattern}' '${file}' 2>/dev/null && echo yes || echo no", returnStdout: true).trim() == 'yes'
                    }

                    def hasNextConfig  = exists("${repoDir}/next.config.js") || exists("${repoDir}/next.config.mjs")
                    def hasViteConfig  = exists("${repoDir}/vite.config.js") || exists("${repoDir}/vite.config.ts")
                    def hasFrontendDir = exists("${repoDir}/frontend")
                    def hasClientDir   = exists("${repoDir}/client")
                    def hasBackendDir  = exists("${repoDir}/backend")
                    def hasServerDir   = exists("${repoDir}/server")
                    def hasRootPkg     = exists("${repoDir}/package.json")
                    def hasExpressDep  = hasRootPkg && grepQ("${repoDir}/package.json", '"express"')

                    def managePy        = findFirst(repoDir, 'manage.py', 3)
                    def appPy           = findFirst(repoDir, 'app.py', 3)
                    def mainPy          = findFirst(repoDir, 'main.py', 3)
                    def requirementsTxt = findFirst(repoDir, 'requirements.txt', 3)
                    def pyproject       = findFirst(repoDir, 'pyproject.toml', 3)
                    def artisan         = findFirst(repoDir, 'artisan', 3)
                    def composerJson    = findFirst(repoDir, 'composer.json', 3)
                    def phpFile         = findFirst(repoDir, '*.php', 3)
                    def pomXml          = findFirst(repoDir, 'pom.xml', 3)
                    def buildGradle     = findFirst(repoDir, 'build.gradle', 3)
                    def indexHtml       = findFirst(repoDir, 'index.html', 2)

                    if (hasNextConfig) {
                        stacks.add([type: 'nextjs', dir: repoDir, port: '3000'])
                    } else if (hasFrontendDir && exists("${repoDir}/frontend/package.json")) {
                        stacks.add([type: 'react', dir: "${repoDir}/frontend", port: '3000'])
                    } else if (hasClientDir && exists("${repoDir}/client/package.json")) {
                        stacks.add([type: 'react', dir: "${repoDir}/client", port: '3000'])
                    } else if (hasViteConfig) {
                        stacks.add([type: 'react', dir: repoDir, port: '3000'])
                    } else if (hasRootPkg && !hasExpressDep && !managePy) {
                        stacks.add([type: 'node', dir: repoDir, port: '3000'])
                    }

                    if (hasBackendDir && exists("${repoDir}/backend/package.json")) {
                        stacks.add([type: 'node', dir: "${repoDir}/backend", port: '4000'])
                    } else if (hasServerDir && exists("${repoDir}/server/package.json")) {
                        stacks.add([type: 'node', dir: "${repoDir}/server", port: '4000'])
                    } else if (hasExpressDep && stacks.isEmpty()) {
                        stacks.add([type: 'node', dir: repoDir, port: '3000'])
                    }

                    if (managePy) {
                        def djangoDir = sh(script: "dirname '${managePy}'", returnStdout: true).trim()
                        stacks.add([type: 'django', dir: djangoDir, port: '8000'])
                    } else if ((appPy || mainPy) && (requirementsTxt || pyproject)) {
                        def pyEntry   = appPy ?: mainPy
                        def pyDir     = sh(script: "dirname '${pyEntry}'", returnStdout: true).trim()
                        def isFastApi = requirementsTxt && grepQ(requirementsTxt, 'fastapi')
                        stacks.add([type: isFastApi ? 'fastapi' : 'flask', dir: pyDir, port: '8000'])
                    } else if (requirementsTxt && stacks.isEmpty()) {
                        stacks.add([type: 'python', dir: repoDir, port: '8000'])
                    }

                    if (artisan) {
                        def laravelDir = sh(script: "dirname '${artisan}'", returnStdout: true).trim()
                        stacks.add([type: 'laravel', dir: laravelDir, port: '8080'])
                    } else if (phpFile) {
                        def phpBase = composerJson
                            ? sh(script: "dirname '${composerJson}'", returnStdout: true).trim()
                            : repoDir
                        stacks.add([type: 'php', dir: phpBase, port: '8080'])
                    }

                    if (pomXml || buildGradle) {
                        def javaDir = pomXml
                            ? sh(script: "dirname '${pomXml}'", returnStdout: true).trim()
                            : sh(script: "dirname '${buildGradle}'", returnStdout: true).trim()
                        stacks.add([type: 'java', dir: javaDir, port: '8080'])
                    }

                    if (stacks.isEmpty() && indexHtml) {
                        stacks.add([type: 'static', dir: repoDir, port: '80'])
                    }

                    if (stacks.isEmpty()) {
                        echo "WARNING: No stack detected — defaulting to Node.js"
                        stacks.add([type: 'node', dir: repoDir, port: '3000'])
                    }

                    env.IS_MULTI_STACK    = stacks.size() > 1 ? 'true' : 'false'
                    env.STACKS_SERIALIZED = stacks.collect { "${it.type}:${it.dir}:${it.port}" }.join('|')
                    env.PRIMARY_STACK     = stacks[0].type
                    env.PRIMARY_DIR       = stacks[0].dir
                    env.PRIMARY_PORT      = stacks[0].port

                    echo "Detected ${stacks.size()} stack(s): ${env.STACKS_SERIALIZED}"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 7 — DEPENDENCY AUDIT
        // ─────────────────────────────────────────────────────────────────
        stage('Dependency Audit') {
            steps {
                script {
                    echo "=== DEPENDENCY AUDIT ==="

                    env.STACKS_SERIALIZED.split('\\|').each { raw ->
                        def parts = raw.split(':')
                        def type  = parts[0]
                        def dir   = parts[1]

                        if (type in ['react', 'nextjs', 'node']) {
                            def hasPkgLock = sh(script: "test -f '${dir}/package-lock.json' && echo yes || echo no", returnStdout: true).trim()
                            if (hasPkgLock == 'yes') {
                                sh "cd '${dir}' && npm audit --audit-level=high 2>/dev/null || true"
                            }
                        } else if (type in ['django', 'flask', 'fastapi', 'python']) {
                            def hasReq = sh(script: "test -f '${dir}/requirements.txt' && echo yes || echo no", returnStdout: true).trim()
                            if (hasReq == 'yes') {
                                sh "pip3 install safety -q 2>/dev/null && safety check -r '${dir}/requirements.txt' 2>/dev/null || true"
                            }
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 8 — CREATE DOCKERFILE
        // ─────────────────────────────────────────────────────────────────
        stage('Create Dockerfile') {
            steps {
                script {
                    echo "=== CREATE DOCKERFILE ==="

                    def stacksRaw = env.STACKS_SERIALIZED.split('\\|').collect {
                        def p = it.split(':')
                        [type: p[0], dir: p[1], port: p[2]]
                    }

                    def makeDockerfile = { Map s ->
                        switch (s.type) {

                            case 'react':
                            case 'vite':
                                return '''FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps
COPY . .
RUN npm run build

FROM nginx:alpine
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY --from=builder /app/dist ./
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''

                            case 'nextjs':
                                return '''FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
'''

                            case 'node':
                                def entry = sh(
                                    script: """
                                        node -e "try{var p=require('${s.dir}/package.json');var st=(p.scripts&&p.scripts.start)||'';console.log(st.replace('node ','').trim()||p.main||'index.js');}catch(e){console.log('index.js');}" 2>/dev/null || echo index.js
                                    """,
                                    returnStdout: true
                                ).trim()

                                return """FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev --legacy-peer-deps
COPY . .
EXPOSE ${s.port}
CMD ["node", "${entry}"]
"""
case 'django':
    def wsgiCmd = "gunicorn --bind 0.0.0.0:${s.port} --workers 3 \$(cat /wsgi_module.txt).wsgi:application"

    return """FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt gunicorn
COPY . .
RUN python manage.py collectstatic --noinput 2>/dev/null || true

RUN find . -name wsgi.py | head -1 | sed 's|^./||' | sed 's|/wsgi.py||' | tr '/' '.' > /wsgi_module.txt

RUN echo '#!/bin/sh' > /start.sh
RUN echo '${wsgiCmd}' >> /start.sh
RUN chmod +x /start.sh

EXPOSE ${s.port}
CMD ["/start.sh"]
"""
                            case 'flask':
                                return """FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt gunicorn
COPY . .
EXPOSE ${s.port}
CMD ["gunicorn", "--bind", "0.0.0.0:${s.port}", "--workers", "3", "app:app"]
"""

                            case 'fastapi':
                                return """FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt "uvicorn[standard]"
COPY . .
EXPOSE ${s.port}
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "${s.port}"]
"""

                            case 'python':
                                return """FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt 2>/dev/null || true
COPY . .
EXPOSE ${s.port}
CMD ["python", "main.py"]
"""

                            case 'laravel':
                                return """FROM php:8.2-fpm-alpine
WORKDIR /var/www/html
RUN apk add --no-cache nginx supervisor curl unzip && docker-php-ext-install pdo pdo_mysql
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
COPY composer*.json ./
RUN composer install --no-dev --optimize-autoloader --no-scripts
COPY . .
RUN cp .env.example .env 2>/dev/null || true && php artisan key:generate --force || true
EXPOSE ${s.port}
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=${s.port}"]
"""

                            case 'php':
                                return '''FROM php:8.2-apache
WORKDIR /var/www/html
RUN docker-php-ext-install pdo pdo_mysql mysqli
COPY . .
RUN chown -R www-data:www-data /var/www/html
EXPOSE 80
CMD ["apache2-foreground"]
'''

                            case 'java':
                                def isMaven  = sh(script: "test -f '${s.dir}/pom.xml' && echo yes || echo no", returnStdout: true).trim() == 'yes'
                                def buildCmd = isMaven ? 'mvn package -DskipTests' : './gradlew build -x test'
                                def jarPath  = isMaven ? 'target/*.jar' : 'build/libs/*.jar'

                                return """FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY . .
RUN ${buildCmd}

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/${jarPath} app.jar
EXPOSE ${s.port}
ENTRYPOINT ["java", "-jar", "app.jar"]
"""

                            case 'static':
                                return '''FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''

                            default:
                                return """FROM node:20-alpine
WORKDIR /app
COPY . .
RUN test -f package.json && npm ci --legacy-peer-deps || true
EXPOSE ${s.port}
CMD ["sh", "-c", "npm start 2>/dev/null || node index.js 2>/dev/null || python3 main.py 2>/dev/null || echo No entry point found"]
"""
                        }
                    }

                    stacksRaw.each { s ->
                        def content = makeDockerfile(s)
                        def dfPath = "${s.dir}/Dockerfile"
                        writeFile file: dfPath, text: content
                        echo "Written: ${dfPath}"
                    }

                    env.DOCKERFILE_PATH = "${stacksRaw[0].dir}/Dockerfile"
                    env.BUILD_CONTEXT   = stacksRaw[0].dir

                    if (stacksRaw.size() > 1) {
                        def compose = "version: '3.8'\\nservices:\\n"

                        stacksRaw.eachWithIndex { s, idx ->
                            def svcName  = idx == 0 ? params.APP_NAME : "${params.APP_NAME}-${s.type}-${idx}"
                            def hostPort = idx == 0 ? (env.ALLOCATED_PORT ?: s.port) : "${Integer.parseInt(env.ALLOCATED_PORT ?: '3000') + idx}"

                            compose += """  ${svcName}:
    build:
      context: ${s.dir}
      dockerfile: Dockerfile
    image: ${env.DOCKERHUB_CRED_USR}/${svcName}:latest
    ports:
      - "${hostPort}:${s.port}"
    restart: unless-stopped
"""
                        }

                        def composePath = "${env.WORK_DIR}/docker-compose.yml"
                        writeFile file: composePath, text: compose
                        env.COMPOSE_FILE = composePath
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 9 — BUILD IMAGE
        // ─────────────────────────────────────────────────────────────────
        stage('Build Image') {
            steps {
                script {
                    echo "=== BUILD IMAGE ==="

                    def stacksRaw = env.STACKS_SERIALIZED.split('\\|').collect {
                        def p = it.split(':')
                        [type: p[0], dir: p[1], port: p[2]]
                    }

                    def imageList = []

                    stacksRaw.eachWithIndex { s, idx ->
                        def svcName = idx == 0 ? params.APP_NAME : "${params.APP_NAME}-${s.type}-${idx}"
                        def imgName = "${env.DOCKERHUB_CRED_USR}/${svcName}:latest"

                        sh """
                            docker build --no-cache \
                            -t '${imgName}' \
                            -f '${s.dir}/Dockerfile' \
                            '${s.dir}'
                        """

                        imageList.add(imgName)
                    }

                    env.BUILT_IMAGES = imageList.join('|')
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 10 — IMAGE SCAN
        // ─────────────────────────────────────────────────────────────────
        stage('Image Scan') {
            steps {
                script {
                    echo "=== IMAGE SCAN ==="

                    def trivyOk = sh(script: "which trivy 2>/dev/null && echo yes || echo no", returnStdout: true).trim()

                    env.BUILT_IMAGES.split('\\|').each { img ->
                        if (trivyOk == 'yes') {
                            sh "trivy image --exit-code 0 --severity HIGH,CRITICAL --no-progress '${img}' || true"
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 11 — PUSH TO DOCKERHUB
        // ─────────────────────────────────────────────────────────────────
        stage('Push to DockerHub') {
            steps {
                script {
                    echo "=== PUSH TO DOCKERHUB ==="

                    sh "echo '${env.DOCKERHUB_CRED_PSW}' | docker login -u '${env.DOCKERHUB_CRED_USR}' --password-stdin"

                    env.BUILT_IMAGES.split('\\|').each { img ->
                        sh "docker push '${img}'"
                    }

                    sh "docker logout || true"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 12 — DEPLOY
        // ─────────────────────────────────────────────────────────────────
        stage('Deploy') {
            steps {
                script {
                    echo "=== DEPLOY (${params.DEPLOY_MODE}) ==="
                    echo "Deploy stage ready"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 13 — VERIFY
        // ─────────────────────────────────────────────────────────────────
        stage('Verify') {
            steps {
                script {
                    echo "=== VERIFY ==="
                    echo "Verify stage ready"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded."
        }
        failure {
            echo "Pipeline failed. Check logs above."
            node('') {
                script {
                    if (params.DEPLOY_MODE == 'local' && params.APP_NAME) {
                        sh "docker stop '${params.APP_NAME}' 2>/dev/null || true"
                        sh "docker rm '${params.APP_NAME}' 2>/dev/null || true"
                    }
                }
            }
        }
        always {
            echo "Pipeline finished."
            node('') {
                script {
                    if (env.WORK_DIR) {
                        sh "rm -rf '${env.WORK_DIR}' || true"
                    }
                }
            }
        }
    }
}
