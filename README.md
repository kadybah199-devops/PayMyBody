# 🚀 Jenkins on AWS — Terraform Modules

Déploiement automatisé d'un serveur Jenkins conteneurisé sur AWS EC2 via des modules Terraform.

---

## 📁 Structure du projet

```
jenkins-terraform/
├── modules/
│   ├── ec2/            # Instance EC2 Ubuntu Jammy
│   ├── ebs/            # Volume EBS
│   ├── eip/            # Elastic IP publique
│   ├── security_group/ # Security Group (ports 80, 443, 8080, 22)
│   └── key_pair/       # Paire de clés SSH générée dynamiquement
└── app/
    ├── main.tf         # Assemblage des 5 modules
    ├── variables.tf    # Déclaration des variables
    ├── outputs.tf      # Sorties (IP, DNS, URL Jenkins)
    ├── terraform.tfvars # Surcharge des variables
    └── user_data.sh    # Script d'installation Docker + Jenkins
```

---

## ⚙️ Prérequis

- [Terraform](https://developer.hashicorp.com/terraform/downloads) >= 1.3.0
- Un compte AWS avec les droits EC2, EBS, EIP, Security Group
- AWS CLI configuré : `aws configure`

---

## 🚀 Déploiement

```bash
# 1. Se placer dans le dossier app
cd app/

# 2. Initialiser Terraform
terraform init

# 3. Vérifier le plan
terraform plan

# 4. Appliquer (déployer)
terraform apply -auto-approve
```

---

## 📤 Résultats après déploiement

- **`jenkins_ec2.txt`** — créé automatiquement dans `app/` avec l'IP et le DNS
- **`jenkins-key-prod.pem`** — clé SSH privée générée automatiquement dans `app/`
- Jenkins accessible sur : `http://<PUBLIC_IP>:8080`

---

## 🔐 Connexion SSH à l'instance

```bash
ssh -i jenkins-key-prod.pem ubuntu@<PUBLIC_IP>
```

---

## 🔧 Variables personnalisables

| Variable           | Description                   | Défaut         |
|--------------------|-------------------------------|----------------|
| `aws_region`       | Région AWS                    | `eu-west-3`    |
| `availability_zone`| Zone de disponibilité         | `eu-west-3a`   |
| `instance_type`    | Type d'instance EC2           | `t2.medium`    |
| `instance_tag`     | Nom de l'instance             | `jenkins-prod` |
| `ebs_size`         | Taille du volume EBS (Go)     | `30`           |
| `key_name`         | Nom de la paire de clés SSH   | `jenkins-key`  |

---

## 🗑️ Destruction de l'infrastructure

```bash
terraform destroy -auto-approve
```
