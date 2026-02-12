# Jenkins CI/CD - Corrections et Améliorations

## 🔧 Problèmes Corrigés

### 1. Configuration Java 21
**Problème:** Gradle et Maven ne trouvaient pas l'installation JDK 21.

**Solution:**
- Ajout de `JAVA_HOME=/opt/jdk21` dans les variables d'environnement Jenkins
- Ajout de `/opt/jdk21/bin` au PATH
- Configuration explicite de Gradle avec `-Dorg.gradle.java.home=/opt/jdk21`
- Configuration explicite de Maven avec `JAVA_HOME=/opt/jdk21`

### 2. Structure du Jenkinsfile
**Problème:** Le stage "Deploy to TAF" était en dehors du bloc `pipeline`.

**Solution:**
- Déplacé le stage de déploiement à l'intérieur du bloc `stages`
- Ajout d'un titre plus clair: "Deploy to TAF" au lieu de "Deploy to TAF (A - minimal)"

### 3. Build CodeQL
**Problème:** CodeQL échouait car les compilations Java échouaient.

**Solution:**
- Configuration correcte de JAVA_HOME dans le script `codeql-build-java.sh`
- Export des variables d'environnement Java avant chaque build

### 4. Dockerfile amélioré
**Ajouts:**
- Installation de Python3 et pip (utiles pour les scripts d'analyse)
- Vérification de l'installation Java avec `java -version`
- Meilleure documentation

## 📝 Changements Détaillés

### Jenkinsfile

#### Variables d'environnement ajoutées:
```groovy
environment {
  JAVA_HOME = "/opt/jdk21"
  PATH = "/opt/jdk21/bin:${env.PATH}"
  // ... autres variables existantes
}
```

#### Stage "Verify toolchain":
```bash
echo "JAVA_HOME=${JAVA_HOME}"  # Ajout pour vérification
```

#### Stage "Build (best effort)":
```bash
# Configuration Java explicite pour Gradle
(cd "$d" && ./gradlew build -x test -Dorg.gradle.java.home=/opt/jdk21)

# Configuration Java explicite pour Maven
JAVA_HOME=/opt/jdk21 ./mvnw -q -DskipTests package
```

#### Stage "CodeQL database + analyze":
```bash
# Script de build avec configuration Java
cat > codeql-build-java.sh << 'EOF'
#!/bin/bash
set -eux
export JAVA_HOME=/opt/jdk21
export PATH=/opt/jdk21/bin:$PATH
# ... reste du script
EOF
```

#### Stage "Deploy to TAF":
- Déplacé de l'extérieur vers l'intérieur du bloc `stages`
- Titre simplifié

### Dockerfile

#### Améliorations:
```dockerfile
# Installation de Python (pour les scripts d'analyse)
RUN apt-get install -y python3 python3-pip

# Vérification de l'installation Java
RUN java -version && javac -version
```

## 🚀 Déploiement

### Mise à jour du Jenkins

1. **Arrêter Jenkins:**
   ```bash
   docker compose down
   ```

2. **Reconstruire l'image avec le nouveau Dockerfile:**
   ```bash
   docker compose build
   ```

3. **Redémarrer Jenkins:**
   ```bash
   docker compose up -d
   ```

### Mise à jour du Jenkinsfile

1. Remplacer l'ancien `Jenkinsfile` par le nouveau dans votre repository
2. Commit et push:
   ```bash
   git add Jenkinsfile
   git commit -m "Fix: Configure Java 21 for Gradle and Maven builds"
   git push
   ```

3. Jenkins détectera automatiquement le changement et utilisera le nouveau fichier

## ✅ Vérification

### Test de la configuration Java:
Après redémarrage de Jenkins, le stage "Verify toolchain" devrait afficher:
```
JAVA_HOME=/opt/jdk21
openjdk version "21.0.10" 2026-01-20 LTS
javac 21.0.10
```

### Build Gradle réussi:
Les builds Gradle dans `auth`, `gateway`, `registry`, et `user` devraient maintenant compiler sans erreur.

### Build Maven réussi:
Les builds Maven dans les sous-projets devraient également compiler correctement.

### CodeQL:
La création de la base de données Java et l'analyse devraient se terminer avec succès:
```
=== Creating Java database ===
=== Analyzing Java with 4096MB RAM ===
=== Java analysis complete ===
```

## 📊 Workflow CI/CD Complet

```
1. Checkout
   ↓
2. Verify toolchain (✅ vérifie Java 21)
   ↓
3. Install Node.js
   ↓
4. Download CodeQL CLI
   ↓
5. Download CodeQL Query Packs
   ↓
6. Build (best effort) (✅ utilise Java 21)
   ↓
7. CodeQL database + analyze (✅ utilise Java 21)
   ↓
8. Quality Gate CodeQL
   ↓
9. Archive SARIF
   ↓
10. Deploy to TAF (seulement sur branche main)
```

## 🐛 Troubleshooting

### Si Gradle échoue toujours:
Vérifier que `/opt/jdk21` existe dans le conteneur:
```bash
docker exec -it <jenkins-container> ls -la /opt/jdk21
```

### Si Maven échoue toujours:
Vérifier la variable JAVA_HOME dans les logs:
```bash
docker exec -it <jenkins-container> echo $JAVA_HOME
```

### Si CodeQL échoue:
1. Vérifier que les builds précédents ont réussi
2. Vérifier les logs du script `codeql-build-java.sh`
3. Augmenter la RAM si nécessaire: `CODEQL_RAM = "8192"`

## 📚 Ressources

- [Gradle Toolchain Documentation](https://docs.gradle.org/current/userguide/toolchains.html)
- [CodeQL CLI Manual](https://codeql.github.com/docs/codeql-cli/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)

## 🔄 Prochaines Étapes Recommandées

1. **Tests automatisés:** Réactiver les tests (`-x test` pourrait être enlevé après résolution des problèmes)
2. **Cache Gradle/Maven:** Implémenter un cache pour accélérer les builds
3. **Notifications:** Ajouter des notifications Slack/Email en cas d'échec
4. **Optimisation CodeQL:** Ajuster les seuils de quality gate selon vos besoins
5. **Multi-branch:** Tester le pipeline sur différentes branches

## 📞 Support

Pour toute question ou problème, consulter:
- Les logs Jenkins: `http://your-jenkins-url/job/your-job/lastBuild/console`
- Les artefacts SARIF: disponibles dans chaque build
- La documentation CodeQL dans le repository
