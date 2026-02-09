pipeline {
  agent any
  options { timestamps() }
  environment {
    CODEQL_VERSION = "2.24.1"
    CODEQL_DIR = "${WORKSPACE}/codeql"
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
          rm -rf codeql-db-java codeql-db-js codeql-*.sarif
          
          # Créer un script de build pour CodeQL
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
          "${CODEQL_DIR}/codeql" database create codeql-db-java \
            --language=java \
            --source-root . \
            --command=./codeql-build-java.sh
          
          "${CODEQL_DIR}/codeql" database analyze codeql-db-java \
            codeql/java-queries:codeql-suites/java-security-and-quality.qls \
            --format=sarifv2.1.0 \
            --output=codeql-java.sarif \
            --threads=0
          
          # --- JAVASCRIPT (pas besoin de build)
          "${CODEQL_DIR}/codeql" database create codeql-db-js \
            --language=javascript \
            --source-root .
          
          "${CODEQL_DIR}/codeql" database analyze codeql-db-js \
            codeql/javascript-queries:codeql-suites/javascript-security-and-quality.qls \
            --format=sarifv2.1.0 \
            --output=codeql-js.sarif \
            --threads=0
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