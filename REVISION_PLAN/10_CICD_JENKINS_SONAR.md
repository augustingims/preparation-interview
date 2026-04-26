# 10 — CI/CD (Jenkins) & SonarQube

## 1. Pipeline Jenkins (Declarative)

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'eclipse-temurin:21-jdk-alpine'
            args '-v $HOME/.m2:/root/.m2' // cache Maven partagé
        }
    }

    environment {
        SONAR_TOKEN      = credentials('sonarqube-token')
        DOCKER_REGISTRY  = 'registry.example.com'
        IMAGE_NAME       = 'myapp'
        IMAGE_TAG        = "${env.GIT_COMMIT[0..7]}" // 8 premiers caractères du SHA
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_BRANCH = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
                }
            }
        }

        stage('Build & Test') {
            steps {
                sh './mvnw clean verify -B -Dmaven.test.failure.ignore=false'
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                    jacoco execPattern: 'target/jacoco.exec', minimumLineCoverage: '80'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ./mvnw sonar:sonar \
                            -Dsonar.projectKey=myapp \
                            -Dsonar.branch.name=${GIT_BRANCH} \
                            -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch pattern: 'release/*'
                }
            }
            steps {
                sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
            }
        }

        stage('Push Image') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-registry', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "docker login ${DOCKER_REGISTRY} -u ${USER} -p ${PASS}"
                    sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                sh "helm upgrade --install myapp ./helm/myapp \
                    --set image.tag=${IMAGE_TAG} \
                    --namespace staging"
            }
        }

        stage('Integration Tests') {
            when { branch 'main' }
            steps {
                sh './mvnw verify -Pintegration-tests -Dapp.url=http://myapp-staging'
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' }
            input {
                message 'Deploy to production?'
                ok 'Deploy'
                parameters {
                    string(name: 'REASON', description: 'Deployment reason')
                }
            }
            steps {
                sh "helm upgrade --install myapp ./helm/myapp \
                    --set image.tag=${IMAGE_TAG} \
                    --namespace production"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            slackSend(channel: '#alerts', color: 'danger',
                message: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
        }
        success {
            slackSend(channel: '#deployments', color: 'good',
                message: "Deployed ${IMAGE_TAG} to production: ${env.JOB_NAME}")
        }
    }
}
```

---

## 2. SonarQube

### Configuration Maven

```xml
<!-- pom.xml -->
<properties>
    <sonar.projectKey>com.example:myapp</sonar.projectKey>
    <sonar.coverage.jacoco.xmlReportPaths>target/site/jacoco/jacoco.xml</sonar.coverage.jacoco.xmlReportPaths>
    <sonar.exclusions>
        **/generated/**,
        **/dto/**,
        **/*Application.java,
        **/config/**
    </sonar.exclusions>
</properties>

<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

### Quality Gate Personnalisée

| Métrique | Seuil Recommandé |
|----------|-----------------|
| Coverage (nouveau code) | ≥ 80% |
| Duplications (nouveau code) | ≤ 3% |
| Maintainability Rating | A |
| Reliability Rating | A (0 bugs) |
| Security Rating | A (0 vulnérabilités) |
| Security Hotspots Reviewed | 100% |

### Types de Problèmes Sonar

```java
// Bug : comportement incorrect
String name = user.getName().toUpperCase();  // NullPointerException si name null
// Fix :
String name = Optional.ofNullable(user.getName()).map(String::toUpperCase).orElse("");

// Code Smell : maintenabilité
// "Cognitive Complexity" trop élevée
if (a) { if (b) { if (c) { if (d) { /* 4 niveaux */ } } } }
// Fix : extraire des méthodes, early return

// Vulnerability
// SQL Injection
String query = "SELECT * FROM users WHERE name = '" + userInput + "'";
// Fix : PreparedStatement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, userInput);

// Security Hotspot (à review manuellement)
MessageDigest.getInstance("MD5")  // algorithme faible
// Fix : utiliser SHA-256 ou BCrypt pour les mots de passe
```

---

## Questions d'Interview — CI/CD & SonarQube

---

### Q1 : Expliquez la différence entre CI et CD.

**Réponse :**

| | **CI** (Continuous Integration) | **CD** (Continuous Delivery) | **CD** (Continuous Deployment) |
|-|--------------------------------|------------------------------|-------------------------------|
| **Objectif** | Intégrer fréquemment | Livrer à tout moment | Déployer automatiquement |
| **Déclencheur** | Push/PR | CI réussi | CI réussi |
| **Déploiement** | Non | Manuel (prêt à déployer) | Automatique en prod |

**En pratique :** CI = build + tests + qualité à chaque commit. CD = pipeline qui peut déployer en prod à la demande (avec approbation) ou automatiquement.

---

### Q2 : Comment gérer les secrets dans un pipeline Jenkins ?

**Réponse :**

```groovy
// JAMAIS dans le Jenkinsfile ou le code source
// BONNE PRATIQUE : Jenkins Credentials Store

// 1. Username/Password
withCredentials([usernamePassword(
    credentialsId: 'db-credentials',
    usernameVariable: 'DB_USER',
    passwordVariable: 'DB_PASS'
)]) {
    sh "flyway migrate -url=${DB_URL} -user=${DB_USER} -password=${DB_PASS}"
}

// 2. Secret text (token API)
withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
    sh "./mvnw sonar:sonar -Dsonar.token=${SONAR_TOKEN}"
}

// 3. Fichier secret (kubeconfig, certificat)
withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    sh "kubectl --kubeconfig=${KUBECONFIG} apply -f k8s/"
}
```

---

### Q3 : Qu'est-ce que le Quality Gate SonarQube et comment le bloquer le pipeline ?

**Réponse :**

Le Quality Gate est un ensemble de conditions qui définissent si le code est "prêt à livrer". Si une condition échoue, le pipeline doit être bloqué.

```groovy
// Dans le Jenkinsfile
stage('Quality Gate') {
    steps {
        // Attendre que l'analyse soit complète
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
            // abortPipeline: true → Jenkins status = FAILURE si QG échoue
        }
    }
}
```

**Configuration SonarQube :**
1. Activer le webhook SonarQube → Jenkins : `Project Settings > Webhooks > http://jenkins/sonarqube-webhook/`
2. Dans Jenkins : configurer l'URL SonarQube et le token dans `Manage Jenkins > System`

---

## Pièges Classiques

- Mettre des secrets en clair dans le Jenkinsfile (même dans un repo privé)
- Pipeline sans Quality Gate → dette technique non contrôlée
- Tests flaky (intermittents) → briser la confiance dans le pipeline → investiguer et fixer
- `mvnw` sans `chmod +x` → erreur de permission dans Docker
- Ne pas nettoyer le workspace (`cleanWs()`) → disque plein sur l'agent Jenkins
