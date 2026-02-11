pipeline {
  agent any
  options { timestamps() }
  environment {
    CODEQL_VERSION = "2.24.1"
    CODEQL_DIR = "${WORKSPACE}/codeql"
    CODEQL_RAM = "4096"
    NODE_VERSION = "20.18.1"
    NODE_DIR = "${WORKSPACE}/node"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Verify toolchain') {
      steps {
        sh '''
          set -eux
          whoami
          id
          java -version
          javac -version
          git --version
          curl --version
          unzip -v | head -n 1
          node --version || true
          npm --version || true
        '''
      }
    }
    stage('Install Node.js') {
      steps {
        sh '''
          set -eux
          # Vérifier si Node.js est déjà installé dans le workspace
          if [ -x "${NODE_DIR}/bin/node" ]; then
            echo "Node.js is already installed in workspace: $(${NODE_DIR}/bin/node --version)"
          else
            echo "Installing Node.js ${NODE_VERSION}..."
            
            # Télécharger Node.js (binaire Linux x64) en format tar.gz au lieu de tar.xz
            rm -f node.tar.gz
            curl -fL --retry 5 --retry-delay 2 \
              -o node.tar.gz \
              "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.gz"
            
            # Vérifier le téléchargement
            ls -lh node.tar.gz
            
            # Extraire (tar.gz au lieu de tar.xz)
            rm -rf "${NODE_DIR}"
            mkdir -p "${NODE_DIR}"
            tar -xzf node.tar.gz -C "${NODE_DIR}" --strip-components=1
            
            # Nettoyer
            rm -f node.tar.gz
          fi
          
          # Ajouter Node.js au PATH et vérifier
          export PATH="${NODE_DIR}/bin:$PATH"
          echo "Node.js version: $(node --version)"
          echo "npm version: $(npm --version)"
        '''
      }
    }
    stage('Download CodeQL CLI') {
      steps {
        sh '''
          set -eux
          rm -f codeql.zip
          curl -fL --retry 5 --retry-delay 2 \
            -o codeql.zip \
            "https://github.com/github/codeql-cli-binaries/releases/download/v${CODEQL_VERSION}/codeql-linux64.zip"
          ls -lh codeql.zip
          head -c 200 codeql.zip || true
          rm -rf "${CODEQL_DIR}"
          unzip -q codeql.zip -d "${WORKSPACE}"
          "${CODEQL_DIR}/codeql" version
        '''
      }
    }
    stage('Download CodeQL Query Packs') {
      steps {
        sh '''
          set -eux
          # Télécharger les query packs depuis GitHub
          "${CODEQL_DIR}/codeql" pack download codeql/java-queries
          "${CODEQL_DIR}/codeql" pack download codeql/javascript-queries
          
          # Vérifier les packs disponibles
          "${CODEQL_DIR}/codeql" resolve packs
          "${CODEQL_DIR}/codeql" resolve languages
        '''
      }
    }
    stage('Build (best effort)') {
      steps {
        sh '''
          set -eux
          # Gradle services (wrappers)
          for d in auth gateway registry user; do
            if [ -x "$d/gradlew" ]; then
              echo "== Gradle build in $d =="
              (cd "$d" && ./gradlew build -x test) || echo "WARN: Gradle build failed in $d (continuing)"
            fi
          done
          # Maven wrappers
          for w in $(find . -maxdepth 4 -name mvnw -type f); do
            d=$(dirname "$w")
            echo "== Maven build in $d =="
            (cd "$d" && chmod +x mvnw && ./mvnw -q -DskipTests package) || echo "WARN: Maven build failed in $d (continuing)"
          done
        '''
      }
    }
    stage('CodeQL database + analyze (Java + JS)') {
      steps {
        sh '''
          set -eux
          
          # Ajouter Node.js au PATH pour CodeQL
          export PATH="${NODE_DIR}/bin:$PATH"
          
          rm -rf codeql-db-java codeql-db-js codeql-*.sarif
          
          # Créer un script de build pour CodeQL Java
          cat > codeql-build-java.sh << 'EOF'
#!/bin/bash
set -eux
for dir in auth gateway registry user; do
  if [ -x "${dir}/gradlew" ]; then
    echo "== CodeQL trace Gradle in ${dir} =="
    (cd "${dir}" && ./gradlew --no-daemon clean compileJava)
  fi
done
EOF
          chmod +x codeql-build-java.sh
          
          # --- JAVA (tracer la compilation)
          echo "=== Creating Java database ==="
          "${CODEQL_DIR}/codeql" database create codeql-db-java \
            --language=java \
            --source-root . \
            --command=./codeql-build-java.sh
          
          echo "=== Analyzing Java with ${CODEQL_RAM}MB RAM ==="
          "${CODEQL_DIR}/codeql" database analyze codeql-db-java \
            codeql/java-queries:codeql-suites/java-security-and-quality.qls \
            --format=sarifv2.1.0 \
            --output=codeql-java.sarif \
            --ram=${CODEQL_RAM} \
            --threads=0
          
          echo "=== Java analysis complete ==="
          ls -lh codeql-java.sarif
          
          # --- JAVASCRIPT/TYPESCRIPT (analyse statique)
          echo "=== Creating JavaScript/TypeScript database ==="
          "${CODEQL_DIR}/codeql" database create codeql-db-js \
            --language=javascript \
            --source-root .
          
          echo "=== Analyzing JavaScript/TypeScript with ${CODEQL_RAM}MB RAM ==="
          "${CODEQL_DIR}/codeql" database analyze codeql-db-js \
            codeql/javascript-queries:codeql-suites/javascript-security-and-quality.qls \
            --format=sarifv2.1.0 \
            --output=codeql-js.sarif \
            --ram=${CODEQL_RAM} \
            --threads=0
          
          echo "=== JavaScript/TypeScript analysis complete ==="
          ls -lh codeql-js.sarif
          
          echo "=== All analyses complete ==="
          ls -lh codeql-*.sarif
        '''
      }
    }
    stage('Archive SARIF') {
      steps {
        archiveArtifacts artifacts: 'codeql-*.sarif', fingerprint: true
      }
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'codeql.zip, codeql-*.sarif', allowEmptyArchive: true
    }
  }
}