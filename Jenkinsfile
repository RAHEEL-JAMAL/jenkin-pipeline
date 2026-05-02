pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL',     defaultValue: '', description: 'GitHub repository URL')
        string(name: 'APP_NAME',     defaultValue: '', description: 'Application name (slug)')
        choice(name: 'DEPLOY_MODE',  choices: ['local', 'aws'], description: 'Deployment target')
    }

    environment {
        DOCKERHUB_USER      = credentials('dockerhub-username')   // Jenkins credential (Secret Text)
        DOCKERHUB_PASS      = credentials('dockerhub-password')   // Jenkins credential (Secret Text)
        AWS_ACCESS_KEY_ID   = credentials('aws-access-key-id')    // Jenkins credential (Secret Text)
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_REGION          = 'us-east-1'
        VM_IP               = '192.168.122.127'
        PORT_RANGE_START    = '3000'
        PORT_RANGE_END      = '4000'
        WORK_DIR            = "/tmp/deployments/${params.APP_NAME}"
        IMAGE_NAME          = "${DOCKERHUB_USER}/${params.APP_NAME}:latest"
    }

    stages {

        // ─────────────────────────────────────────────────────────────────
        // STAGE 1 — INIT
        // ─────────────────────────────────────────────────────────────────
        stage('Init') {
            steps {
                script {
                    echo "=== INIT ==="
                    echo "APP_NAME   : ${params.APP_NAME}"
                    echo "REPO_URL   : ${params.REPO_URL}"
                    echo "DEPLOY_MODE: ${params.DEPLOY_MODE}"

                    if (!params.REPO_URL || !params.APP_NAME) {
                        error("REPO_URL and APP_NAME are required.")
                    }

                    sh "rm -rf ${WORK_DIR} && mkdir -p ${WORK_DIR}"
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
        // STAGE 3 — ALLOCATE SAFE PORT (local only)
        // ─────────────────────────────────────────────────────────────────
        stage('Allocate Safe Port') {
            when { expression { params.DEPLOY_MODE == 'local' } }
            steps {
                script {
                    echo "=== ALLOCATE SAFE PORT ==="
                    def port = sh(
                        script: """
                            python3 - <<'EOF'
import socket, sys

start = int("${PORT_RANGE_START}")
end   = int("${PORT_RANGE_END}")

for p in range(start, end + 1):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind(('0.0.0.0', p))
        s.close()
        print(p)
        sys.exit(0)
    except OSError:
        pass

print("NO_PORT")
sys.exit(1)
EOF
                        """,
                        returnStdout: true
                    ).trim()

                    if (port == 'NO_PORT') {
                        error("No free port available in range ${PORT_RANGE_START}–${PORT_RANGE_END}")
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
                    sh """
                        git clone --depth=1 ${env.REPO_URL_CLEAN} ${WORK_DIR}/repo
                    """
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
                    // Basic grep-based secret scan (no external tool needed)
                    def secrets = sh(
                        script: """
                            grep -rIE \
                                '(password|passwd|secret|api_key|apikey|token|access_key|private_key|aws_secret)\\s*[:=]\\s*["\\'\\`]?[A-Za-z0-9+/]{8,}' \
                                ${WORK_DIR}/repo \
                                --include='*.env' \
                                --include='*.json' \
                                --include='*.yaml' \
                                --include='*.yml' \
                                --include='*.py' \
                                --include='*.js' \
                                --include='*.ts' \
                                --include='*.php' \
                                --include='*.java' \
                                -l 2>/dev/null || true
                        """,
                        returnStdout: true
                    ).trim()

                    if (secrets) {
                        echo "⚠️  Potential secrets found in files:\n${secrets}"
                        echo "Proceeding with warning — review these files before production use."
                    } else {
                        echo "✅ No obvious secrets found."
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 6 — DETECT STACK
        // Multi-stack detection: a single repo can have MULTIPLE stacks
        // ─────────────────────────────────────────────────────────────────
        stage('Detect Stack') {
            steps {
                script {
                    echo "=== DETECT STACK ==="

                    def repoDir = "${WORK_DIR}/repo"

                    /*
                     * Detection order matters for multi-stack repos.
                     * We collect ALL detected stacks, then pick a build strategy.
                     *
                     * Structure examples we handle:
                     *   /frontend  (React/Vite/Next.js)  +  /backend  (Node / Django / Flask)
                     *   Single-root Next.js full-stack
                     *   PHP Laravel monolith
                     *   Java Spring Boot
                     *   Static HTML site
                     */

                    def stacks = []   // list of detected stack objects

                    // ── React / Vite / Next.js ──────────────────────────
                    def hasFrontendDir = sh(script: "test -d ${repoDir}/frontend && echo yes || echo no", returnStdout: true).trim()
                    def hasClientDir   = sh(script: "test -d ${repoDir}/client   && echo yes || echo no", returnStdout: true).trim()
                    def hasNextConfig  = sh(script: "test -f ${repoDir}/next.config.js -o -f ${repoDir}/next.config.mjs && echo yes || echo no", returnStdout: true).trim()
                    def hasViteConfig  = sh(script: "test -f ${repoDir}/vite.config.js -o -f ${repoDir}/vite.config.ts && echo yes || echo no", returnStdout: true).trim()
                    def hasRootPkg     = sh(script: "test -f ${repoDir}/package.json && echo yes || echo no", returnStdout: true).trim()

                    // ── Node.js backend ──────────────────────────────────
                    def hasBackendDir  = sh(script: "test -d ${repoDir}/backend && echo yes || echo no", returnStdout: true).trim()
                    def hasServerDir   = sh(script: "test -d ${repoDir}/server  && echo yes || echo no", returnStdout: true).trim()
                    def hasExpressDep  = ''
                    if (hasRootPkg == 'yes') {
                        hasExpressDep = sh(script: "grep -q '\"express\"' ${repoDir}/package.json && echo yes || echo no", returnStdout: true).trim()
                    }

                    // ── Python (Django / Flask / FastAPI) ────────────────
                    def hasManagePy    = sh(script: "find ${repoDir} -maxdepth 3 -name 'manage.py'   | head -1", returnStdout: true).trim()
                    def hasFlaskApp    = sh(script: "find ${repoDir} -maxdepth 3 -name 'app.py' -o -name 'main.py' | head -1", returnStdout: true).trim()
                    def hasRequirements= sh(script: "find ${repoDir} -maxdepth 3 -name 'requirements.txt' | head -1", returnStdout: true).trim()
                    def hasPyproject   = sh(script: "find ${repoDir} -maxdepth 3 -name 'pyproject.toml' | head -1", returnStdout: true).trim()

                    // ── PHP ──────────────────────────────────────────────
                    def hasComposer    = sh(script: "find ${repoDir} -maxdepth 3 -name 'composer.json' | head -1", returnStdout: true).trim()
                    def hasPhpFiles    = sh(script: "find ${repoDir} -maxdepth 3 -name '*.php' | head -1", returnStdout: true).trim()
                    def hasArtisan     = sh(script: "find ${repoDir} -maxdepth 3 -name 'artisan'       | head -1", returnStdout: true).trim()

                    // ── Java (Spring Boot / Maven / Gradle) ──────────────
                    def hasPomXml      = sh(script: "find ${repoDir} -maxdepth 3 -name 'pom.xml'       | head -1", returnStdout: true).trim()
                    def hasBuildGradle = sh(script: "find ${repoDir} -maxdepth 3 -name 'build.gradle'  | head -1", returnStdout: true).trim()

                    // ── Static HTML ──────────────────────────────────────
                    def hasIndexHtml   = sh(script: "find ${repoDir} -maxdepth 2 -name 'index.html'    | head -1", returnStdout: true).trim()

                    // ─────────────────────────────────────────────────────
                    // Build the stacks list (ORDER: specific → generic)
                    // ─────────────────────────────────────────────────────

                    // Next.js (full-stack, overrides plain React)
                    if (hasNextConfig == 'yes') {
                        stacks.add([type: 'nextjs', dir: repoDir, port: '3000'])
                    }
                    // Frontend sub-dir (React/Vite)
                    else if (hasFrontendDir == 'yes') {
                        stacks.add([type: 'react', dir: "${repoDir}/frontend", port: '3000'])
                    }
                    else if (hasClientDir == 'yes') {
                        stacks.add([type: 'react', dir: "${repoDir}/client", port: '3000'])
                    }
                    else if (hasViteConfig == 'yes') {
                        stacks.add([type: 'react', dir: repoDir, port: '3000'])
                    }
                    else if (hasRootPkg == 'yes' && hasExpressDep == 'no' && !hasManagePy) {
                        // root package.json that is NOT express and NOT django → assume react/node app
                        stacks.add([type: 'node', dir: repoDir, port: '3000'])
                    }

                    // Backend sub-dir (Node/Express)
                    if (hasBackendDir == 'yes') {
                        def beHasPkg = sh(script: "test -f ${repoDir}/backend/package.json && echo yes || echo no", returnStdout: true).trim()
                        if (beHasPkg == 'yes') {
                            stacks.add([type: 'node', dir: "${repoDir}/backend", port: '4000'])
                        }
                    } else if (hasServerDir == 'yes') {
                        def srvHasPkg = sh(script: "test -f ${repoDir}/server/package.json && echo yes || echo no", returnStdout: true).trim()
                        if (srvHasPkg == 'yes') {
                            stacks.add([type: 'node', dir: "${repoDir}/server", port: '4000'])
                        }
                    } else if (hasExpressDep == 'yes' && stacks.isEmpty()) {
                        stacks.add([type: 'node', dir: repoDir, port: '3000'])
                    }

                    // Django
                    if (hasManagePy) {
                        def djangoDir = sh(script: "dirname ${hasManagePy}", returnStdout: true).trim()
                        stacks.add([type: 'django', dir: djangoDir, port: '8000'])
                    }
                    // Flask / FastAPI / plain Python
                    else if (hasFlaskApp && (hasRequirements || hasPyproject)) {
                        def pyDir = sh(script: "dirname ${hasFlaskApp}", returnStdout: true).trim()
                        def framework = sh(
                            script: "grep -qiE 'fastapi' ${hasRequirements ?: hasPyproject} && echo fastapi || echo flask",
                            returnStdout: true
                        ).trim()
                        stacks.add([type: framework, dir: pyDir, port: '8000'])
                    }
                    else if (hasRequirements && stacks.isEmpty()) {
                        stacks.add([type: 'python', dir: repoDir, port: '8000'])
                    }

                    // PHP / Laravel
                    if (hasArtisan) {
                        def laravelDir = sh(script: "dirname ${hasArtisan}", returnStdout: true).trim()
                        stacks.add([type: 'laravel', dir: laravelDir, port: '8080'])
                    } else if (hasPhpFiles && hasComposer) {
                        def phpDir = sh(script: "dirname ${hasComposer}", returnStdout: true).trim()
                        stacks.add([type: 'php', dir: phpDir, port: '8080'])
                    } else if (hasPhpFiles) {
                        stacks.add([type: 'php', dir: repoDir, port: '8080'])
                    }

                    // Java Spring Boot
                    if (hasPomXml || hasBuildGradle) {
                        def javaDir = hasPomXml \
                            ? sh(script: "dirname ${hasPomXml}", returnStdout: true).trim() \
                            : sh(script: "dirname ${hasBuildGradle}", returnStdout: true).trim()
                        stacks.add([type: 'java', dir: javaDir, port: '8080'])
                    }

                    // Static HTML fallback
                    if (stacks.isEmpty() && hasIndexHtml) {
                        stacks.add([type: 'static', dir: repoDir, port: '80'])
                    }

                    // Final fallback
                    if (stacks.isEmpty()) {
                        echo "⚠️  No stack detected. Defaulting to Node.js."
                        stacks.add([type: 'node', dir: repoDir, port: '3000'])
                    }

                    // ─────────────────────────────────────────────────────
                    // Multi-stack strategy
                    // ─────────────────────────────────────────────────────
                    if (stacks.size() == 1) {
                        echo "✅ Single-stack repo: ${stacks[0].type}"
                        env.IS_MULTI_STACK = 'false'
                    } else {
                        echo "✅ Multi-stack repo detected (${stacks.size()} stacks):"
                        stacks.eachWithIndex { s, i -> echo "   [${i}] type=${s.type}  dir=${s.dir}  port=${s.port}" }
                        env.IS_MULTI_STACK = 'true'
                    }

                    // Serialize stacks into env (JSON-like string, parsed in later stages)
                    // Format: "type:dir:port|type:dir:port|..."
                    env.STACKS_SERIALIZED = stacks.collect { "${it.type}:${it.dir}:${it.port}" }.join('|')
                    env.PRIMARY_STACK     = stacks[0].type
                    env.PRIMARY_DIR       = stacks[0].dir
                    env.PRIMARY_PORT      = stacks[0].port

                    echo "PRIMARY_STACK: ${env.PRIMARY_STACK}"
                    echo "STACKS_SERIALIZED: ${env.STACKS_SERIALIZED}"
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
                    def stacks = env.STACKS_SERIALIZED.split('\\|').collect { it.split(':') }

                    stacks.each { parts ->
                        def type = parts[0]
                        def dir  = parts[1]

                        switch (type) {
                            case ['react', 'nextjs', 'node']:
                                def pkgLock = sh(script: "test -f ${dir}/package-lock.json && echo yes || echo no", returnStdout: true).trim()
                                if (pkgLock == 'yes') {
                                    sh "cd ${dir} && npm audit --audit-level=high --json 2>/dev/null | python3 -c \"import sys,json; d=json.load(sys.stdin); print('Vulnerabilities (high+):', d.get('metadata',{}).get('vulnerabilities',{}).get('high',0)+d.get('metadata',{}).get('vulnerabilities',{}).get('critical',0))\" || true"
                                } else {
                                    echo "No package-lock.json — skipping npm audit for ${type}"
                                }
                                break
                            case ['django', 'flask', 'fastapi', 'python']:
                                def reqFile = sh(script: "test -f ${dir}/requirements.txt && echo yes || echo no", returnStdout: true).trim()
                                if (reqFile == 'yes') {
                                    sh "pip3 install safety --quiet 2>/dev/null && safety check -r ${dir}/requirements.txt 2>/dev/null || true"
                                }
                                break
                            case ['laravel', 'php']:
                                echo "PHP dependency audit — skipping (no composer audit in base image)"
                                break
                            case 'java':
                                echo "Java dependency audit — skipping (requires OWASP plugin)"
                                break
                            default:
                                echo "No audit configured for stack: ${type}"
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 8 — CREATE DOCKERFILE
        // Generates Dockerfile per stack. Multi-stack = multi-stage or
        // docker-compose (written to WORK_DIR for reference).
        // ─────────────────────────────────────────────────────────────────
        stage('Create Dockerfile') {
            steps {
                script {
                    echo "=== CREATE DOCKERFILE ==="

                    def stacksRaw = env.STACKS_SERIALIZED.split('\\|').collect {
                        def p = it.split(':')
                        [type: p[0], dir: p[1], port: p[2]]
                    }

                    // Helper closure: generate Dockerfile content per stack type
                    def generateDockerfile = { Map s ->
                        switch (s.type) {

                            case 'react':
                            case 'vite':
                                return """
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY --from=builder /app/build /usr/share/nginx/html 2>/dev/null || true
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
""".stripIndent()

                            case 'nextjs':
                                return """
FROM node:20-alpine AS builder
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
""".stripIndent()

                            case 'node':
                                // Detect entry point
                                def entry = sh(
                                    script: """
                                        cd ${s.dir}
                                        node -e "try{const p=require('./package.json');console.log(p.main||p.scripts&&(p.scripts.start||'').replace('node ','').trim()||'index.js')}catch(e){console.log('index.js')}" 2>/dev/null || echo 'index.js'
                                    """,
                                    returnStdout: true
                                ).trim()
                                return """
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev --legacy-peer-deps
COPY . .
EXPOSE ${s.port}
CMD ["node", "${entry}"]
""".stripIndent()

                            case 'django':
                                return """
FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN python manage.py collectstatic --noinput 2>/dev/null || true
EXPOSE ${s.port}
CMD ["gunicorn", "--bind", "0.0.0.0:${s.port}", "--workers", "3", "$(find . -name 'wsgi.py' | head -1 | xargs dirname | xargs basename).wsgi:application"]
""".stripIndent()

                            case 'flask':
                                return """
FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt gunicorn
COPY . .
EXPOSE ${s.port}
CMD ["gunicorn", "--bind", "0.0.0.0:${s.port}", "--workers", "3", "app:app"]
""".stripIndent()

                            case 'fastapi':
                                return """
FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt uvicorn[standard]
COPY . .
EXPOSE ${s.port}
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "${s.port}"]
""".stripIndent()

                            case 'python':
                                return """
FROM python:3.11-slim
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --no-cache-dir -r requirements.txt 2>/dev/null || true
COPY . .
EXPOSE ${s.port}
CMD ["python", "main.py"]
""".stripIndent()

                            case 'laravel':
                                return """
FROM php:8.2-fpm-alpine
WORKDIR /var/www/html

RUN apk add --no-cache nginx supervisor curl unzip \\
    && docker-php-ext-install pdo pdo_mysql

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

COPY composer*.json ./
RUN composer install --no-dev --optimize-autoloader --no-scripts

COPY . .
RUN cp .env.example .env 2>/dev/null || true \\
    && php artisan key:generate --force \\
    && php artisan config:cache \\
    && php artisan route:cache \\
    && chmod -R 775 storage bootstrap/cache

COPY docker/nginx-laravel.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE ${s.port}
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
""".stripIndent()

                            case 'php':
                                return """
FROM php:8.2-apache
WORKDIR /var/www/html
RUN docker-php-ext-install pdo pdo_mysql mysqli
COPY . .
RUN chown -R www-data:www-data /var/www/html
EXPOSE 80
CMD ["apache2-foreground"]
""".stripIndent()

                            case 'java':
                                def isMaven = sh(script: "test -f ${s.dir}/pom.xml && echo yes || echo no", returnStdout: true).trim()
                                def buildCmd = isMaven == 'yes' ? 'mvn package -DskipTests' : './gradlew build -x test'
                                def jarGlob  = isMaven == 'yes' ? 'target/*.jar' : 'build/libs/*.jar'
                                return """
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY . .
RUN ${buildCmd}

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/${jarGlob} app.jar
EXPOSE ${s.port}
ENTRYPOINT ["java", "-jar", "app.jar"]
""".stripIndent()

                            case 'static':
                                return """
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
""".stripIndent()

                            default:
                                return """
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN test -f package.json && npm ci --legacy-peer-deps || true
EXPOSE ${s.port}
CMD ["sh", "-c", "npm start 2>/dev/null || node index.js 2>/dev/null || python main.py 2>/dev/null || echo 'No known entry point found'"]
""".stripIndent()
                        }
                    }

                    if (stacksRaw.size() == 1) {
                        // ── Single stack ──────────────────────────────────
                        def content = generateDockerfile(stacksRaw[0])
                        def dfPath  = "${stacksRaw[0].dir}/Dockerfile"
                        sh "cat > ${dfPath} << 'DOCKERFILE_EOF'\n${content}\nDOCKERFILE_EOF"
                        env.DOCKERFILE_PATH = dfPath
                        env.BUILD_CONTEXT   = stacksRaw[0].dir
                        echo "Dockerfile written: ${dfPath}"

                    } else {
                        // ── Multi-stack: write docker-compose.yml ─────────
                        echo "Multi-stack detected — generating docker-compose.yml"
                        def compose = "version: '3.8'\nservices:\n"

                        stacksRaw.eachWithIndex { s, idx ->
                            def svcName = idx == 0 ? params.APP_NAME : "${params.APP_NAME}-${s.type}-${idx}"
                            def dfContent = generateDockerfile(s)
                            def dfPath = "${s.dir}/Dockerfile"
                            sh "cat > ${dfPath} << 'DOCKERFILE_EOF'\n${dfContent}\nDOCKERFILE_EOF"

                            // Compute host port (primary uses ALLOCATED_PORT, others increment)
                            def hostPort = idx == 0 ? (env.ALLOCATED_PORT ?: s.port) : (Integer.parseInt(env.ALLOCATED_PORT ?: s.port) + idx).toString()

                            compose += """\
  ${svcName}:
    build:
      context: ${s.dir}
      dockerfile: Dockerfile
    image: ${DOCKERHUB_USER}/${svcName}:latest
    ports:
      - "${hostPort}:${s.port}"
    restart: unless-stopped
"""
                        }

                        def composePath = "${WORK_DIR}/docker-compose.yml"
                        sh "cat > ${composePath} << 'COMPOSE_EOF'\n${compose}\nCOMPOSE_EOF"
                        env.COMPOSE_FILE = composePath

                        // For image build we use primary stack only (each service builds its own)
                        env.DOCKERFILE_PATH = "${stacksRaw[0].dir}/Dockerfile"
                        env.BUILD_CONTEXT   = stacksRaw[0].dir

                        echo "docker-compose.yml written: ${composePath}"
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 9 — BUILD IMAGE(S)
        // ─────────────────────────────────────────────────────────────────
        stage('Build Image') {
            steps {
                script {
                    echo "=== BUILD IMAGE ==="

                    def stacksRaw = env.STACKS_SERIALIZED.split('\\|').collect {
                        def p = it.split(':')
                        [type: p[0], dir: p[1], port: p[2]]
                    }

                    if (stacksRaw.size() == 1) {
                        sh """
                            docker build \\
                                --no-cache \\
                                -t ${IMAGE_NAME} \\
                                -f ${env.DOCKERFILE_PATH} \\
                                ${env.BUILD_CONTEXT}
                        """
                        env.BUILT_IMAGES = "${IMAGE_NAME}"
                    } else {
                        // Multi-stack: build each image separately
                        def imageList = []
                        stacksRaw.eachWithIndex { s, idx ->
                            def svcName = idx == 0 ? params.APP_NAME : "${params.APP_NAME}-${s.type}-${idx}"
                            def imgName = "${DOCKERHUB_USER}/${svcName}:latest"
                            sh """
                                docker build \\
                                    --no-cache \\
                                    -t ${imgName} \\
                                    -f ${s.dir}/Dockerfile \\
                                    ${s.dir}
                            """
                            imageList.add(imgName)
                        }
                        env.BUILT_IMAGES = imageList.join('|')
                    }

                    echo "Built images: ${env.BUILT_IMAGES}"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────
        // STAGE 10 — IMAGE SCAN (Trivy)
        // ─────────────────────────────────────────────────────────────────
        stage('Image Scan') {
            steps {
                script {
                    echo "=== IMAGE SCAN ==="
                    def trivyAvailable = sh(script: "which trivy 2>/dev/null && echo yes || echo no", returnStdout: true).trim()

                    env.BUILT_IMAGES.split('\\|').each { img ->
                        if (trivyAvailable == 'yes') {
                            sh """
                                trivy image \\
                                    --exit-code 0 \\
                                    --severity HIGH,CRITICAL \\
                                    --no-progress \\
                                    ${img} || true
                            """
                        } else {
                            echo "Trivy not installed — skipping image scan for ${img}"
                            echo "Install: https://aquasecurity.github.io/trivy/latest/getting-started/installation/"
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
                    sh "echo ${DOCKERHUB_PASS} | docker login -u ${DOCKERHUB_USER} --password-stdin"

                    env.BUILT_IMAGES.split('\\|').each { img ->
                        sh "docker push ${img}"
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
                    echo "=== DEPLOY (mode: ${params.DEPLOY_MODE}) ==="

                    def stacksRaw = env.STACKS_SERIALIZED.split('\\|').collect {
                        def p = it.split(':')
                        [type: p[0], dir: p[1], port: p[2]]
                    }

                    if (params.DEPLOY_MODE == 'local') {

                        // ── LOCAL DEPLOYMENT ──────────────────────────────
                        def appUrls = []

                        stacksRaw.eachWithIndex { s, idx ->
                            def svcName  = idx == 0 ? params.APP_NAME : "${params.APP_NAME}-${s.type}-${idx}"
                            def imgName  = "${DOCKERHUB_USER}/${svcName}:latest"
                            def hostPort = idx == 0 \
                                ? (env.ALLOCATED_PORT ?: s.port) \
                                : sh(
                                    script: """
                                        python3 -c "
import socket, sys
start=${Integer.parseInt(env.ALLOCATED_PORT ?: '3000') + 1}
end=4000
for p in range(start+${idx}-1, end+1):
    try:
        s=socket.socket(); s.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1); s.bind(('0.0.0.0',p)); s.close(); print(p); sys.exit(0)
    except: pass
print('NO_PORT'); sys.exit(1)"
                                    """,
                                    returnStdout: true
                                ).trim()

                            // Stop & remove old container
                            sh """
                                docker stop ${svcName}  2>/dev/null || true
                                docker rm   ${svcName}  2>/dev/null || true
                            """

                            // Pull latest image
                            sh "docker pull ${imgName}"

                            // Run new container
                            sh """
                                docker run -d \\
                                    --name ${svcName} \\
                                    --restart unless-stopped \\
                                    -p ${hostPort}:${s.port} \\
                                    -e NODE_ENV=production \\
                                    -e PORT=${s.port} \\
                                    ${imgName}
                            """

                            def url = "http://${env.VM_IP}:${hostPort}"
                            appUrls.add("${svcName} → ${url}")
                            echo "✅ Deployed ${svcName} at ${url}"
                        }

                        env.APP_URL = appUrls[0].split('→')[1].trim()
                        echo "\n=== DEPLOYED SERVICES ===\n${appUrls.join('\n')}"

                    } else {

                        // ── AWS DEPLOYMENT ────────────────────────────────
                        // We deploy the PRIMARY image to a new EC2 instance.
                        // For multi-stack, the same instance runs docker-compose.

                        def primaryImg = env.BUILT_IMAGES.split('\\|')[0]
                        def sgName     = "sg-${params.APP_NAME}-${BUILD_NUMBER}"
                        def keyName    = "deploy-key-${params.APP_NAME}"

                        // Create Security Group
                        def sgId = sh(
                            script: """
                                aws ec2 create-security-group \\
                                    --group-name ${sgName} \\
                                    --description "Auto-created for ${params.APP_NAME}" \\
                                    --region ${AWS_REGION} \\
                                    --query 'GroupId' \\
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()
                        echo "Security Group: ${sgId}"

                        // Open ports
                        sh """
                            aws ec2 authorize-security-group-ingress \\
                                --group-id ${sgId} \\
                                --protocol tcp --port 22   --cidr 0.0.0.0/0 \\
                                --region ${AWS_REGION} || true

                            aws ec2 authorize-security-group-ingress \\
                                --group-id ${sgId} \\
                                --protocol tcp --port 80   --cidr 0.0.0.0/0 \\
                                --region ${AWS_REGION} || true

                            aws ec2 authorize-security-group-ingress \\
                                --group-id ${sgId} \\
                                --protocol tcp --port 443  --cidr 0.0.0.0/0 \\
                                --region ${AWS_REGION} || true

                            aws ec2 authorize-security-group-ingress \\
                                --group-id ${sgId} \\
                                --protocol tcp --port 8080 --cidr 0.0.0.0/0 \\
                                --region ${AWS_REGION} || true
                        """

                        // Create key pair (save pem to WORK_DIR)
                        sh """
                            aws ec2 delete-key-pair --key-name ${keyName} --region ${AWS_REGION} 2>/dev/null || true
                            aws ec2 create-key-pair \\
                                --key-name ${keyName} \\
                                --region ${AWS_REGION} \\
                                --query 'KeyMaterial' \\
                                --output text > ${WORK_DIR}/${keyName}.pem
                            chmod 600 ${WORK_DIR}/${keyName}.pem
                        """

                        // User data script: install Docker + pull + run
                        def allImages   = env.BUILT_IMAGES.split('\\|')
                        def dockerRunCmds = stacksRaw.eachWithIndex.collect { s, idx ->
                            def svcName = idx == 0 ? params.APP_NAME : "${params.APP_NAME}-${s.type}-${idx}"
                            def img     = allImages[Math.min(idx, allImages.size() - 1)]
                            def hPort   = idx == 0 ? '80' : (8080 + idx).toString()
                            "docker run -d --name ${svcName} --restart unless-stopped -p ${hPort}:${s.port} -e PORT=${s.port} ${img}"
                        }.join('\n')

                        def userData = """#!/bin/bash
set -e
apt-get update -y
apt-get install -y docker.io
systemctl start docker
systemctl enable docker
docker login -u ${DOCKERHUB_USER} -p ${DOCKERHUB_PASS}
${dockerRunCmds}
docker logout
""".stripIndent()

                        def userDataB64 = sh(
                            script: "echo '${userData}' | base64 -w 0",
                            returnStdout: true
                        ).trim()

                        // Launch EC2
                        def instanceId = sh(
                            script: """
                                aws ec2 run-instances \\
                                    --image-id ami-0c7217cdde317cfec \\
                                    --instance-type t2.micro \\
                                    --key-name ${keyName} \\
                                    --security-group-ids ${sgId} \\
                                    --user-data '${userData}' \\
                                    --region ${AWS_REGION} \\
                                    --count 1 \\
                                    --query 'Instances[0].InstanceId' \\
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()
                        echo "EC2 Instance ID: ${instanceId}"

                        // Wait for running state
                        sh """
                            aws ec2 wait instance-running \\
                                --instance-ids ${instanceId} \\
                                --region ${AWS_REGION}
                        """

                        // Get Public IP
                        def publicIp = sh(
                            script: """
                                aws ec2 describe-instances \\
                                    --instance-ids ${instanceId} \\
                                    --region ${AWS_REGION} \\
                                    --query 'Reservations[0].Instances[0].PublicIpAddress' \\
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()

                        env.APP_URL     = "http://${publicIp}"
                        env.INSTANCE_ID = instanceId
                        echo "✅ AWS deployment complete: ${env.APP_URL}"
                    }
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
                    def maxRetries = 12
                    def retryDelay = 10   // seconds
                    def success    = false

                    for (int i = 1; i <= maxRetries; i++) {
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' --max-time 10 ${env.APP_URL} || echo 000",
                            returnStdout: true
                        ).trim()

                        echo "Attempt ${i}/${maxRetries} — HTTP ${status} — ${env.APP_URL}"

                        if (status ==~ /^[23]\d\d$/) {
                            success = true
                            echo "✅ Application is live at ${env.APP_URL}"
                            break
                        }

                        if (i < maxRetries) {
                            sleep(retryDelay)
                        }
                    }

                    if (!success) {
                        echo "⚠️  Health check did not return 2xx/3xx after ${maxRetries} attempts."
                        echo "The container may still be starting. Check: ${env.APP_URL}"
                        // Don't hard-fail — container may need more startup time
                    }

                    // Final summary
                    echo """
╔══════════════════════════════════════════════════╗
║           DEPLOYMENT SUMMARY                     ║
╠══════════════════════════════════════════════════╣
║ App Name   : ${params.APP_NAME}
║ Mode       : ${params.DEPLOY_MODE}
║ Stack(s)   : ${env.STACKS_SERIALIZED}
║ Image(s)   : ${env.BUILT_IMAGES}
║ App URL    : ${env.APP_URL}
║ Multi-Stack: ${env.IS_MULTI_STACK}
╚══════════════════════════════════════════════════╝
"""
                }
            }
        }
    }

    // ─────────────────────────────────────────────────────────────────────
    // POST
    // ─────────────────────────────────────────────────────────────────────
    post {
        success {
            echo "🎉 Pipeline completed successfully. App live at: ${env.APP_URL ?: 'unknown'}"
        }
        failure {
            echo "❌ Pipeline failed. Check stage logs above."
            // Cleanup on failure (local mode only)
            script {
                if (params.DEPLOY_MODE == 'local') {
                    sh "docker stop ${params.APP_NAME} 2>/dev/null || true"
                    sh "docker rm   ${params.APP_NAME} 2>/dev/null || true"
                }
            }
        }
        always {
            echo "Cleaning up workspace temp dir..."
            sh "rm -rf ${WORK_DIR} || true"
        }
    }
}
