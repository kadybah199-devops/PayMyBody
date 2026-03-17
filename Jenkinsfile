pipeline {
    agent none

    environment {
        // DockerHub
        DOCKERHUB_USERNAME  = "kady199"
        IMAGE_NAME          = "${DOCKERHUB_USERNAME}/spring-app"
        IMAGE_TAG           = "${BUILD_NUMBER}"

        // SonarCloud
        SONAR_PROJECT_KEY   = "jenkins"
        SONAR_ORG           = "kadybah199-devops"

        // Serveurs SSH
        STAGING_HOST        = "13.220.182.218"
        PROD_HOST           = "18.212.245.161"
        SSH_USER            = "ubuntu"

        // Slack
        SLACK_CHANNEL       = "#ci-cd-notifications"
    }

    stages {

        // ─────────────────────────────────────────
        // ÉTAPE 1 : Tests Automatisés
        // ─────────────────────────────────────────
        stage('Tests Automatisés') {
            agent {
                docker {
                    image 'maven:3.9.6-eclipse-temurin-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        // ─────────────────────────────────────────
        // ÉTAPE 2 : Qualité de Code (SonarCloud)
        // ─────────────────────────────────────────
        stage('Vérification Qualité Code') {
            agent {
                docker {
                    image 'maven:3.9.6-eclipse-temurin-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.organization=${SONAR_ORG} \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        // ─────────────────────────────────────────
        // ÉTAPE 3 : Build + Package + Push DockerHub
        // ─────────────────────────────────────────
        stage('Compilation et Packaging') {
            agent {
                docker {
                    image 'maven:3.9.6-eclipse-temurin-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                // Compilation Maven
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build et Push Image Docker') {
            agent { label 'built-in' } // Docker-in-Docker sur le nœud Jenkins
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                            echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                            docker push ${IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${IMAGE_NAME}:latest
                            docker logout
                        """
                    }
                }
            }
        }

        // ─────────────────────────────────────────
        // ÉTAPE 4 : Staging (main uniquement)
        // ─────────────────────────────────────────
        stage('Déploiement Staging') {
            when {
                branch 'main'
            }
            agent { label 'built-in' }
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ssh-staging-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${STAGING_HOST} '
                            docker pull ${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker stop spring-app-staging || true &&
                            docker rm spring-app-staging || true &&
                            docker run -d \
                                --name spring-app-staging \
                                --restart always \
                                -p 8080:8080 \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        // ─────────────────────────────────────────
        // ÉTAPE 5 : Production (main uniquement)
        // ─────────────────────────────────────────
        stage('Déploiement Production') {
            when {
                branch 'main'
            }
            agent { label 'built-in' }
            steps {
                // Validation manuelle avant prod
                timeout(time: 5, unit: 'MINUTES') {
                    input message: 'Déployer en Production ?', ok: 'Confirmer'
                }
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ssh-prod-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${PROD_HOST} '
                            docker pull ${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker stop spring-app-prod || true &&
                            docker rm spring-app-prod || true &&
                            docker run -d \
                                --name spring-app-prod \
                                --restart always \
                                -p 80:8080 \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        // ─────────────────────────────────────────
        // ÉTAPE 6 : Tests de Validation
        // ─────────────────────────────────────────
        stage('Tests de Validation Staging') {
            when {
                branch 'main'
            }
            agent {
                docker {
                    image 'curlimages/curl:latest'
                }
            }
            steps {
                sh """
                    sleep 15
                    curl -f http://${STAGING_HOST}:8080/actuator/health || exit 1
                    echo "✅ Staging opérationnel"
                """
            }
        }

        stage('Tests de Validation Production') {
            when {
                branch 'main'
            }
            agent {
                docker {
                    image 'curlimages/curl:latest'
                }
            }
            steps {
                sh """
                    sleep 10
                    curl -f http://${PROD_HOST}/actuator/health || exit 1
                    echo "✅ Production opérationnelle"
                """
            }
        }
    }

    // ─────────────────────────────────────────
    // NOTIFICATION SLACK
    // ─────────────────────────────────────────
    post {
        success {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: "good",
                message: """
✅ *Pipeline réussie !*
- *Job* : ${JOB_NAME}
- *Build* : #${BUILD_NUMBER}
- *Branche* : ${GIT_BRANCH}
- *Durée* : ${currentBuild.durationString}
- *Lien* : ${BUILD_URL}
                """
            )
        }
        failure {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: "danger",
                message: """
❌ *Pipeline échouée !*
- *Job* : ${JOB_NAME}
- *Build* : #${BUILD_NUMBER}
- *Branche* : ${GIT_BRANCH}
- *Étape* : ${currentBuild.currentResult}
- *Lien* : ${BUILD_URL}
                """
            )
        }
        always {
            cleanWs()
        }
    }
}
```

---

## Étape 3 : Configurer Jenkins

### Plugins à installer
Dans **Manage Jenkins → Plugins**, installez :
```
✅ Docker Pipeline
✅ SonarQube Scanner
✅ Slack Notification
✅ SSH Agent
✅ Multibranch Pipeline
