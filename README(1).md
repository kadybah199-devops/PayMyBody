# 🚀 Pipeline CI/CD avec Jenkins - Spring Boot

> Projet de mise en œuvre d'une pipeline d'intégration et de déploiement continu (CI/CD) pour une application Spring Boot, avec Jenkins, Docker, SonarCloud et Slack.

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

Concevoir une pipeline CI/CD complète permettant de :

- ✅ Exécuter des **tests automatisés** (unitaires et d'intégration)
- ✅ Analyser la **qualité du code** avec SonarCloud
- ✅ **Compiler, packager** et publier une image Docker sur DockerHub
- ✅ Déployer automatiquement sur les environnements **Staging** et **Production**
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
       ├── Tests Automatisés (Docker: maven)
       ├── Qualité Code      (Docker: maven + SonarCloud)
       ├── Build + Push      (Docker: maven + DockerHub)
       │
       └── [main uniquement]
           ├── Déploiement Staging  ──► Serveur Staging (SSH)
           ├── Déploiement Prod     ──► Serveur Production (SSH)
           └── Tests de Validation  ──► curl /actuator/health
                    │
                    ▼
             Notification Slack ✅ / ❌
```

---

## ⚙️ Prérequis

### Infrastructure requise

| Serveur | Rôle | Configuration minimale |
|---|---|---|
| Jenkins | Orchestration CI/CD | 2 vCPU, 4 Go RAM |
| Staging | Pré-production | 1 vCPU, 2 Go RAM |
| Production | Production | 1 vCPU, 2 Go RAM |

### Comptes et services

| Service | Utilisation | Lien |
|---|---|---|
| **GitHub** | Hébergement du code | [github.com](https://github.com) |
| **DockerHub** | Registry d'images Docker | [hub.docker.com](https://hub.docker.com) |
| **SonarCloud** | Analyse qualité de code | [sonarcloud.io](https://sonarcloud.io) |
| **Slack** | Notifications pipeline | [slack.com](https://slack.com) |
| **AWS** | Infrastructure cloud (EC2) | [aws.amazon.com](https://aws.amazon.com) |

### Outils locaux

```bash
git --version
terraform --version
aws --version
docker --version
```

---

## 📁 Structure du Projet

```
terraform_jenkins/
├── app/                          # Code source Spring Boot
│   ├── src/
│   │   ├── main/java/            # Code source principal
│   │   └── test/java/            # Tests unitaires et d'intégration
│   └── pom.xml                   # Configuration Maven
├── Dockerfile                    # Image Docker multi-stage
├── Jenkinsfile                   # Définition de la pipeline CI/CD
├── .gitignore                    # Fichiers exclus de Git
└── README.md                     # Documentation du projet
```

---

## 🔧 Configuration

### 1. Variables d'environnement Jenkins

Mettez à jour les variables suivantes dans le `Jenkinsfile` :

```groovy
environment {
    DOCKERHUB_USERNAME  = "votre_dockerhub_username"
    IMAGE_NAME          = "${DOCKERHUB_USERNAME}/spring-app"
    SONAR_PROJECT_KEY   = "votre_sonar_project_key"
    SONAR_ORG           = "votre_sonar_organisation"
    STAGING_HOST        = "IP_SERVEUR_STAGING"
    PROD_HOST           = "IP_SERVEUR_PRODUCTION"
    SSH_USER            = "ubuntu"
    SLACK_CHANNEL       = "#ci-cd-notifications"
}
```

### 2. Credentials Jenkins

Créez les credentials suivants dans **Manage Jenkins → Credentials → Global** :

| ID Credential | Type | Description |
|---|---|---|
| `dockerhub-credentials` | Username/Password | Identifiants DockerHub |
| `sonar-token` | Secret text | Token d'accès SonarCloud |
| `ssh-staging-key` | SSH Private Key | Clé SSH serveur staging |
| `ssh-prod-key` | SSH Private Key | Clé SSH serveur production |
| `slack-token` | Secret text | Token Bot Slack |

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

### 4. Dépendances pom.xml

```xml
<!-- SonarQube -->
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.10.0.2594</version>
</plugin>

<!-- Spring Boot Actuator (tests de validation) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 5. Génération des clés SSH

```bash
# Clé pour le serveur Staging
ssh-keygen -t rsa -b 4096 -f ~/.ssh/staging_key -N ""
ssh-copy-id -i ~/.ssh/staging_key.pub ubuntu@IP_STAGING

# Clé pour le serveur Production
ssh-keygen -t rsa -b 4096 -f ~/.ssh/prod_key -N ""
ssh-copy-id -i ~/.ssh/prod_key.pub ubuntu@IP_PROD
```

### 6. .gitignore recommandé

```gitignore
# Terraform
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.backup
*.tfvars
crash.log

# Clés SSH
*.pem
*.key

# IDE
.idea/
.vscode/
*.iml
```

---

## 🔄 Pipeline CI/CD

### Étapes de la pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                      TOUTES LES BRANCHES                    │
├──────────────┬──────────────────┬───────────────────────────┤
│   Étape 1    │     Étape 2      │         Étape 3           │
│    Tests     │  Qualité Code    │  Build + Push DockerHub   │
│  Automatisés │  (SonarCloud)    │  (Maven + Docker)         │
└──────────────┴──────────────────┴───────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   BRANCHE MAIN UNIQUEMENT                   │
├──────────────┬──────────────────┬───────────────────────────┤
│   Étape 4    │     Étape 5      │         Étape 6           │
│  Déploiement │   Déploiement    │   Tests de Validation     │
│   Staging    │   Production     │   (Staging + Prod)        │
└──────────────┴──────────────────┴───────────────────────────┘
                                            │
                                            ▼
                                   Notification Slack
```

### Description des étapes

#### Étape 1 — Tests Automatisés
- Agent Docker : `maven:3.9.6-eclipse-temurin-17`
- Commande : `mvn test`
- Publication des rapports JUnit

#### Étape 2 — Vérification Qualité Code
- Agent Docker : `maven:3.9.6-eclipse-temurin-17`
- Analyse SonarCloud via `mvn sonar:sonar`
- Vérification des normes de qualité et de sécurité

#### Étape 3 — Compilation et Packaging
- Compilation : `mvn clean package -DskipTests`
- Build image Docker multi-stage
- Push sur DockerHub avec tag `BUILD_NUMBER` et `latest`

#### Étape 4 — Déploiement Staging *(main uniquement)*
- Connexion SSH au serveur staging
- Pull de la nouvelle image Docker
- Redémarrage du conteneur sur le port `8080`

#### Étape 5 — Déploiement Production *(main uniquement)*
- Validation manuelle requise (timeout 5 min)
- Connexion SSH au serveur de production
- Déploiement sur le port `80`

#### Étape 6 — Tests de Validation *(main uniquement)*
- Agent Docker : `curlimages/curl:latest`
- Vérification de l'endpoint `/actuator/health`
- Validation staging sur le port `8080`
- Validation production sur le port `80`

---

## 🌿 Modèle Gitflow

```
main ──────────────────────────────────────────────►
  ✅ Tests   ✅ Qualité   ✅ Build
  ✅ Staging ✅ Prod      ✅ Validation

develop ───────────────────────────────────────────►
  ✅ Tests   ✅ Qualité   ✅ Build
  ❌ Staging ❌ Prod      ❌ Validation

feature/* ─────────────────────────────────────────►
  ✅ Tests   ✅ Qualité   ✅ Build
  ❌ Staging ❌ Prod      ❌ Validation
```

---

## 🚀 Déploiement

### Créer la pipeline dans Jenkins

```
1. Jenkins → New Item
2. Nom      : terraform_jenkins
3. Type     : Multibranch Pipeline
4. Source   : GitHub → votre repo
5. Save     → Jenkins scanne les branches automatiquement
```

### Webhook GitHub (déclenchement automatique)

```
Settings → Webhooks → Add Webhook
Payload URL  : http://IP_JENKINS:8080/github-webhook/
Content type : application/json
Événement    : push
```

### Vérifier l'application déployée

```bash
# Staging
curl http://IP_STAGING:8080/actuator/health

# Production
curl http://IP_PROD/actuator/health

# Réponse attendue
{"status":"UP"}
```

---

## 🔔 Notifications Slack

**✅ Succès**
```
✅ Pipeline réussie !
• Job     : terraform_jenkins
• Build   : #12
• Branche : main
• Durée   : 4 min 32 sec
• Lien    : http://jenkins:8080/job/...
```

**❌ Échec**
```
❌ Pipeline échouée !
• Job     : terraform_jenkins
• Build   : #12
• Branche : main
• Lien    : http://jenkins:8080/job/...
```

---

## 🛠️ Dépannage

### Credentials AWS expirés (AWS Academy)
```bash
# Les credentials expirent toutes les 4h sur AWS Academy
aws configure set aws_access_key_id "NOUVELLE_KEY"
aws configure set aws_secret_access_key "NOUVEAU_SECRET"
aws configure set aws_session_token "NOUVEAU_TOKEN"

# Vérifier
aws sts get-caller-identity
```

### Modules Terraform non installés
```bash
cd app/
terraform init
terraform validate
terraform apply --auto-approve
```

### Gros fichiers bloqués par GitHub (>100MB)
```bash
# Supprimer .terraform de tout l'historique Git
git filter-branch --force --index-filter \
  "git rm -rf --cached --ignore-unmatch app/.terraform/" \
  --prune-empty --tag-name-filter cat -- --all

git remote add origin https://github.com/votre-repo.git
git push origin main --force
```

### Nettoyage après terraform destroy
```powershell
# PowerShell - Supprimer les fichiers locaux Terraform
Remove-Item -Recurse -Force .terraform
Remove-Item -Force .terraform.lock.hcl
Remove-Item -Force terraform.tfstate
Remove-Item -Force terraform.tfstate.backup
Remove-Item -Force *.pem
```

---

## 📚 Ressources

- [Documentation Jenkins](https://www.jenkins.io/doc/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [SonarCloud Documentation](https://docs.sonarcloud.io/)
- [Docker Hub](https://hub.docker.com/)
- [Meilleures pratiques CI/CD](https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

## 👩‍💻 Auteur

**Kadiatou** — Formation DevOps  
Projet réalisé dans le cadre du TP Jenkins CI/CD Pipeline
