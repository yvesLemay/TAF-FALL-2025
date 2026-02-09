pipeline {
  agent any
  options { timestamps() }

  environment {
    // Version CodeQL CLI (tu peux la changer)
    CODEQL_VERSION = "2.29.2"
    CODEQL_DIR = "${WORKSPACE}/codeql"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install prerequisites') {
      steps {
        sh '''
          set -eux
          apt-get update
          # Java + outils de base (node facultatif pour CodeQL JS)
          apt-get install -y curl unzip git jq openjdk-17-jdk
          java -version
        '''
      }
    }

    stage('Download CodeQL CLI') {
      steps {
        sh '''
          set -eux
          curl -L -o codeql.zip https://github.com/github/codeql-cli-binaries/releases/download/v${CODEQL_VERSION}/codeql-linux64.zip
          rm -rf "${CODEQL_DIR}"
          unzip -q codeql.zip -d "${WORKSPACE}"
          mv "${WORKSPACE}/codeql" "${CODEQL_DIR}"
          "${CODEQL_DIR}/codeql" version
        '''
      }
    }

    stage('Build Java (best effort)') {
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

          # Maven wrappers (on build ce qu'on trouve)
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

          # Crée une DB multi-lang. On fournit une "commande build" légère;
          # les builds au stage précédent ont déjà fait le gros du travail.
          "${CODEQL_DIR}/codeql" database create codeql-db \
            --language=java,javascript \
            --source-root . \
            --command="true"

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

    // Optionnel: upload vers GitHub Code Scanning (si ton PAT a security_events)
    /*
    stage('Upload SARIF to GitHub') {
      steps {
        withCredentials([string(credentialsId: 'github_pat', variable: 'GH_PAT')]) {
          sh '''
            set -eux
            # À adapter: owner/repo + SHA du commit
            # GitHub API SARIF upload: demande un endpoint spécifique et du base64 gzippé.
            # Si tu veux l’activer, dis-moi ton owner/repo et je te mets la commande exacte.
          '''
        }
      }
    }
    */
  }
}
