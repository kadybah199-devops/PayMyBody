# 💪 PayMyBody — CI/CD Pipeline Jenkins

---

## ✅ Prérequis

### Comptes requis
- [DockerHub](https://hub.docker.com) — compte `kadybah199`
- [SonarCloud](https://sonarcloud.io) — connecté avec GitHub
- [Slack](https://slack.com) — workspace avec channel `#ci-cd-notifications`
- [AWS](https://aws.amazon.com) — compte avec accès EC2

### Serveurs AWS EC2 à créer
| Serveur | Type | Rôle |
|---|---|---|
| Jenkins | t2.medium | Exécution pipeline |
| Staging | t2.micro | Pré-production |
| Production | t2.micro | Production |

### Sur chaque serveur EC2, installer Docker
```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

### Lancer Jenkins
```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# Récupérer le mot de passe initial
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

---

## 🔧 Ce qu'il faut modifier dans le Jenkinsfile

**Ouvrez le fichier `Jenkinsfile` et remplacez uniquement ces 2 lignes :**

```groovy
STAGING_HOST = "IP_STAGING"   // ← IP de votre EC2 staging
PROD_HOST    = "IP_PROD"      // ← IP de votre EC2 production
```

> Tout le reste est déjà configuré ✅

---

## 🔑 Credentials à créer dans Jenkins

> **Manage Jenkins → Credentials → Global → Add Credentials**

| ID | Type | Valeur |
|---|---|---|
| `dockerhub-credentials` | Username/Password | Login DockerHub |
| `sonar-token` | Secret text | Token généré sur sonarcloud.io |
| `ssh-staging-key` | SSH Private Key | Clé privée `~/.ssh/staging_key` |
| `ssh-prod-key` | SSH Private Key | Clé privée `~/.ssh/prod_key` |
| `slack-token` | Secret text | Token Bot Slack `xoxb-...` |

### Générer les clés SSH
```bash
# Clé staging
ssh-keygen -t rsa -b 4096 -f ~/.ssh/staging_key -N ""
ssh-copy-id -i ~/.ssh/staging_key.pub ubuntu@IP_STAGING

# Clé production
ssh-keygen -t rsa -b 4096 -f ~/.ssh/prod_key -N ""
ssh-copy-id -i ~/.ssh/prod_key.pub ubuntu@IP_PROD
```

---

## 🔌 Plugins Jenkins à installer

> **Manage Jenkins → Plugins → Available**

```
✅ Docker Pipeline
✅ SonarQube Scanner
✅ Slack Notification
✅ SSH Agent
✅ Multibranch Pipeline
```

---

## ⚙️ Configurer Slack dans Jenkins

> **Manage Jenkins → System → Section Slack**

```
Workspace        : votre-workspace-slack
Credential       : slack-token
Default channel  : #ci-cd-notifications
```

Cliquez **Test Connection** pour vérifier ✅

---

## 🚀 Créer la pipeline dans Jenkins

```
1. Jenkins → New Item
2. Nom  : PayMyBody
3. Type : Multibranch Pipeline
4. Branch Sources → GitHub
5. URL  : https://github.com/kadybah199-devops/PayMyBody.git
6. Save → Jenkins détecte automatiquement les branches
```

---

## 🔗 Webhook GitHub

```
GitHub → Settings → Webhooks → Add Webhook
Payload URL  : http://IP_JENKINS:8080/github-webhook/
Content type : application/json
Événement    : Just the push event ✅
```

---

## ▶️ Lancer la pipeline

```bash
git add .
git commit -m "votre message"
git push origin main
```

La pipeline se déclenche automatiquement sur GitHub push.

### Vérifier les déploiements
```bash
# Staging
curl http://IP_STAGING:8080/actuator/health

# Production
curl http://IP_PROD/actuator/health

# Résultat attendu
{"status":"UP"}
```

---

## 📋 Checklist avant le premier lancement

```
□ EC2 Jenkins, Staging, Production créés sur AWS
□ Docker installé sur les 3 serveurs
□ Jenkins lancé et accessible sur :8080
□ Plugins installés
□ IP_STAGING et IP_PROD remplacés dans le Jenkinsfile
□ 5 credentials créés dans Jenkins
□ Slack configuré dans Jenkins
□ Pipeline Multibranch créée
□ Webhook GitHub configuré
□ git push origin main 🚀
```

---

## 👩‍💻 Auteur

**Kadiatou** · Formation DevOps · [github.com/kadybah199-devops](https://github.com/kadybah199-devops)
