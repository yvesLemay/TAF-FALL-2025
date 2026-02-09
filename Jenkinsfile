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

          # Télécharge et échoue si HTTP != 200
          curl -fL --retry 5 --retry-delay 2 \
            -o codeql.zip \
            "https://github.com/github/codeql-cli-binaries/releases/download/v${CODEQL_VERSION}/codeql-linux64.zip"

          # Vérifie que c'est bien un zip (debug utile)
          ls -lh codeql.zip
          file codeql.zip
          head -c 200 codeql.zip || true

          rm -rf "${CODEQL_DIR}"
          unzip -q codeql.zip -d "${WORKSPACE}"
          mv "${WORKSPACE}/codeql" "${CODEQL_DIR}"
          "${CODEQL_DIR}/codeql" version
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
          rm -rf codeql-db codeql-results.sarif

          "${CODEQL_DIR}/codeql" database create codeql-db \
            --language=java,javascript \
            --source-root . \
            --command="bash -lc 'true'"

          "${CODEQL_DIR}/codeql" database analyze codeql-db \
            --format=sarifv2.1.0 \
            --output=codeql-results.sarif \
            --threads=0
        '''
      }
    }

    stage('Archive SARIF') {
      steps {
        archiveArtifacts artifacts: 'codeql-results.sarif', fingerprint: true
      }
    }
  }

  post {
    always {
      // utile si le pipeline plante après avoir créé des fichiers
      archiveArtifacts artifacts: 'codeql.zip', allowEmptyArchive: true
    }
  }
}
