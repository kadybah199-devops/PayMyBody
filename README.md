# 💪 PayMyBody — Pipeline CI/CD avec Jenkins

> Application Spring Boot de gestion de paiements fitness, déployée via une pipeline CI/CD complète avec Jenkins, Docker, SonarCloud et Slack.

---

## 📋 Table des Matières

- [Objectif](#objectif)
- [Architecture](#architecture)
- [Prérequis](#prérequis)
- [Structure du Projet](#structure-du-projet)
- [Configuration](#configuration)
- [Pipeline CI/CD](#pipeline-cicd)
- [Modèle Gitflow](#modèle-gitflow)
- [Déploiement](#déploiement)
- [Notifications Slack](#notifications-slack)
- [Dépannage](#dépannage)

---

## 🎯 Objectif

Concevoir une pipeline CI/CD complète pour l'application **PayMyBody** permettant de :

- ✅ Exécuter des **tests automatisés** (unitaires et d'intégration)
- ✅ Analyser la **qualité du code** avec SonarCloud
- ✅ **Compiler, packager** et publier une image Docker sur DockerHub
- ✅ Déployer automatiquement sur les environnements **Staging** et **Production** via SSH
- ✅ Valider les déploiements via des **tests de disponibilité**
- ✅ Envoyer des **notifications Slack** sur le statut de la pipeline

---

## 🏗️ Architecture

```
GitHub (push)
     │
     ▼
┌─────────────┐
│   Jenkins   │  ◄── Multibranch Pipeline
└──────┬──────┘
       │
       ├── 1. Tests Automatisés     (Docker: maven)
       ├── 2. Qualité Code          (Docker: maven + SonarCloud)
       ├── 3. Compilation + Docker  (Maven build + DockerHub push)
       │
       └── [main uniquement]
           ├── 4. Déploiement Staging  ──► Serveur Staging :8080 (SSH)
           ├── 5. Déploiement Prod     ──► Serveur Production :80 (SSH)
           └── 6. Tests de Validation  ──► curl /actuator/health
                        │
                        ▼
                 Notification Slack ✅ / ❌
```

---

## ⚙️ Prérequis

### Infrastructure requise

| Serveur    | Rôle             | Configuration minimale |
|------------|------------------|------------------------|
| Jenkins    | Orchestration    | 2 vCPU, 4 Go RAM       |
| Staging    | Pré-production   | 1 vCPU, 2 Go RAM       |
| Production | Production       | 1 vCPU, 2 Go RAM       |

> Les serveurs Staging et Production sont provisionnés sur **AWS EC2** via Terraform.

### Comptes et services requis

| Service        | Utilisation                  | Lien |
|----------------|------------------------------|------|
| **GitHub**     | Hébergement du code          | [github.com](https://github.com) |
| **DockerHub**  | Registry des images Docker   | [hub.docker.com](https://hub.docker.com) |
| **SonarCloud** | Analyse qualité du code      | [sonarcloud.io](https://sonarcloud.io) |
| **Slack**      | Notifications pipeline       | [slack.com](https://slack.com) |
| **AWS**        | Infrastructure cloud (EC2)   | [aws.amazon.com](https://aws.amazon.com) |

### Outils locaux

```bash
git --version        # >= 2.x
docker --version     # >= 20.x
terraform --version  # >= 1.x
aws --version        # >= 2.x
```

---

## 📁 Structure du Projet

```
PayMyBody/
├── .mvn/wrapper/              # Maven wrapper
├── src/
│   ├── main/java/             # Code source Spring Boot
│   └── test/java/             # Tests unitaires et d'intégration
├── Dockerfile                 # Image Docker multi-stage
├── Jenkinsfile                # Définition de la pipeline CI/CD
├── pom.xml                    # Configuration Maven + dépendances
├── mvnw                       # Maven wrapper Linux
├── mvnw.cmd                   # Maven wrapper Windows
├── build_docker.md            # Guide build Docker manuel
└── README.md                  # Documentation du projet
```

---

## 🔧 Configuration

### 1. Variables d'environnement dans le Jenkinsfile

```groovy
environment {
    DOCKERHUB_USERNAME  = "kadybah199"
    IMAGE_NAME          = "${DOCKERHUB_USERNAME}/paymybody"
    IMAGE_TAG           = "${BUILD_NUMBER}"
    SONAR_PROJECT_KEY   = "kadybah199-devops_PayMyBody"
    SONAR_ORG           = "kadybah199-devops"
    STAGING_HOST        = "IP_SERVEUR_STAGING"
    PROD_HOST           = "IP_SERVEUR_PRODUCTION"
    SSH_USER            = "ubuntu"
    SLACK_CHANNEL       = "#ci-cd-notifications"
}
```

### 2. Credentials Jenkins

Créez les credentials dans **Manage Jenkins → Credentials → Global** :

| ID Credential           | Type              | Description                    |
|-------------------------|-------------------|--------------------------------|
| `dockerhub-credentials` | Username/Password | Login DockerHub                |
| `sonar-token`           | Secret text       | Token d'accès SonarCloud       |
| `ssh-staging-key`       | SSH Private Key   | Clé SSH serveur staging        |
| `ssh-prod-key`          | SSH Private Key   | Clé SSH serveur production     |
| `slack-token`           | Secret text       | Token Bot Slack                |

### 3. Plugins Jenkins requis

Installez via **Manage Jenkins → Plugins → Available** :

```
✅ Docker Pipeline
✅ SonarQube Scanner
✅ Slack Notification
✅ SSH Agent
✅ Multibranch Pipeline
✅ Git
```

### 4. Dépendances pom.xml requises

```xml
<!-- Plugin SonarCloud -->
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.10.0.2594</version>
</plugin>

<!-- Spring Boot Actuator (tests de validation health check) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 5. Génération des clés SSH

```bash
# Clé pour Staging
ssh-keygen -t rsa -b 4096 -f ~/.ssh/staging_key -N ""
ssh-copy-id -i ~/.ssh/staging_key.pub ubuntu@IP_STAGING

# Clé pour Production
ssh-keygen -t rsa -b 4096 -f ~/.ssh/prod_key -N ""
ssh-copy-id -i ~/.ssh/prod_key.pub ubuntu@IP_PROD
```

### 6. Docker sur les serveurs Staging et Production

```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

---

## 🔄 Pipeline CI/CD

### Vue d'ensemble des étapes

```
┌──────────────────────────────────────────────────────────────┐
│                       TOUTES LES BRANCHES                    │
├───────────────┬──────────────────┬───────────────────────────┤
│    Étape 1    │     Étape 2      │         Étape 3           │
│     Tests     │  Qualité Code    │   Build + Push DockerHub  │
│  Automatisés  │  (SonarCloud)    │   (Maven + Docker)        │
└───────────────┴──────────────────┴───────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    BRANCHE MAIN UNIQUEMENT                   │
├───────────────┬──────────────────┬───────────────────────────┤
│    Étape 4    │     Étape 5      │         Étape 6           │
│  Déploiement  │   Déploiement    │   Tests de Validation     │
│   Staging     │   Production     │  (health check curl)      │
└───────────────┴──────────────────┴───────────────────────────┘
                                             │
                                             ▼
                                    Notification Slack
```

### Détail de chaque étape

#### 🧪 Étape 1 — Tests Automatisés
- **Agent Docker** : `maven:3.9.6-eclipse-temurin-17`
- **Commande** : `mvn test`
- Publication automatique des rapports JUnit

#### 🔍 Étape 2 — Vérification Qualité Code
- **Agent Docker** : `maven:3.9.6-eclipse-temurin-17`
- Analyse via `mvn sonar:sonar` sur **SonarCloud**
- Vérification des normes de qualité, bugs et vulnérabilités

#### 📦 Étape 3 — Compilation et Packaging
- Compilation : `mvn clean package -DskipTests`
- Build image Docker multi-stage
- Push DockerHub avec tags `BUILD_NUMBER` et `latest`

#### 🧪 Étape 4 — Déploiement Staging *(main uniquement)*
- Connexion SSH au serveur staging
- Pull + redémarrage du conteneur `paymybody-staging`
- Exposition sur le port `8080`

#### 🚀 Étape 5 — Déploiement Production *(main uniquement)*
- **Validation manuelle** requise (timeout 5 minutes)
- Connexion SSH au serveur de production
- Redémarrage du conteneur `paymybody-prod`
- Exposition sur le port `80`

#### ✅ Étape 6 — Tests de Validation *(main uniquement)*
- **Agent Docker** : `curlimages/curl:latest`
- Vérification `GET /actuator/health` sur staging et production
- Résultat attendu : `{"status":"UP"}`

---

## 🌿 Modèle Gitflow

```
main ────────────────────────────────────────────────────►
  ✅ Tests   ✅ Qualité   ✅ Build & Push Docker
  ✅ Staging ✅ Prod      ✅ Tests de Validation

develop ─────────────────────────────────────────────────►
  ✅ Tests   ✅ Qualité   ✅ Build & Push Docker
  ❌ Staging ❌ Prod      ❌ Tests de Validation

feature/* ───────────────────────────────────────────────►
  ✅ Tests   ✅ Qualité   ✅ Build & Push Docker
  ❌ Staging ❌ Prod      ❌ Tests de Validation
```

---

## 🚀 Déploiement

### Créer la pipeline dans Jenkins

```
1. Jenkins → New Item
2. Nom      : PayMyBody
3. Type     : Multibranch Pipeline
4. Source   : GitHub → https://github.com/kadybah199-devops/PayMyBody.git
5. Save     → Jenkins scanne les branches automatiquement
```

### Configurer le Webhook GitHub

```
GitHub → Settings → Webhooks → Add Webhook
Payload URL  : http://IP_JENKINS:8080/github-webhook/
Content type : application/json
Événement    : Just the push event ✅
```

### Vérifier l'application déployée

```bash
# Health check Staging
curl http://IP_STAGING:8080/actuator/health
# → {"status":"UP"}

# Health check Production
curl http://IP_PROD/actuator/health
# → {"status":"UP"}
```

### Build Docker manuel (optionnel)

```bash
# Builder l'image localement
docker build -t kadybah199/paymybody:latest .

# Lancer en local
docker run -d -p 8080:8080 --name paymybody kadybah199/paymybody:latest

# Tester
curl http://localhost:8080/actuator/health
```

---

## 🔔 Notifications Slack

La pipeline envoie automatiquement une notification à la fin de chaque build :

**✅ Succès**
```
✅ Pipeline PayMyBody réussie !
• Job     : PayMyBody
• Build   : #12
• Branche : main
• Durée   : 4 min 32 sec
• Lien    : http://jenkins:8080/job/PayMyBody/12/
```

**❌ Échec**
```
❌ Pipeline PayMyBody échouée !
• Job     : PayMyBody
• Build   : #12
• Branche : main
• Lien    : http://jenkins:8080/job/PayMyBody/12/
```

---

## 🛠️ Dépannage

### Credentials AWS expirés (AWS Academy)
```bash
# Les credentials expirent toutes les 4h
aws configure set aws_access_key_id     "NOUVELLE_KEY"
aws configure set aws_secret_access_key "NOUVEAU_SECRET"
aws configure set aws_session_token     "NOUVEAU_TOKEN"

# Vérifier
aws sts get-caller-identity
```

### Erreur SonarCloud — connexion
```
→ Utilisez "Log in with GitHub" sur sonarcloud.io
→ Ne pas utiliser email/password directement
```

### Gros fichiers bloqués par GitHub (>100MB)
```bash
git filter-branch --force --index-filter \
  "git rm -rf --cached --ignore-unmatch .terraform/" \
  --prune-empty --tag-name-filter cat -- --all

git push origin main --force
```

### Modules Terraform non installés
```bash
cd app/
terraform init
terraform apply --auto-approve
```

### Conteneur Docker déjà existant sur le serveur
```bash
# Le Jenkinsfile gère déjà ce cas avec :
docker stop paymybody-staging || true
docker rm   paymybody-staging || true
```

---

## 📚 Ressources

- [Documentation Jenkins](https://www.jenkins.io/doc/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [SonarCloud Documentation](https://docs.sonarcloud.io/)
- [Docker Hub — kadybah199](https://hub.docker.com/u/kadybah199)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Meilleures pratiques CI/CD Jenkins](https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/)

---

## 👩‍💻 Auteur

**Kadiatou** — Formation DevOps  
🔗 [github.com/kadybah199-devops](https://github.com/kadybah199-devops)  
Projet : Pipeline CI/CD Jenkins pour l'application PayMyBody
