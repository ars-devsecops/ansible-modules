# 🚀 ansible-jenkins-deployment

> Production-grade Ansible playbooks + Jenkins pipeline for Java application deployment
> Real-world implementation based on Godrej Housing Finance (FIC) deployment workflow
> by [@ars-devsecops](https://github.com/ars-devsecops)

---

## 🏗️ Architecture

```
Jenkins Server                          App Server
172.24.x.x                              172.24.5.128
┌─────────────────────────┐             ┌──────────────────────────────────┐
│  Jenkins Pipeline       │             │  /home/ec2-user/PDF/             │
│  UNI-FIC-BE-UAT         │──Ansible──▶ │    PDF_Handler/                  │
│                         │             │      Final_Cert_PDF_Handler/     │
│  Playbooks at:          │             │        *.jar  ← deployed here    │
│  /var/lib/jenkins/      │             │        Backup_jar/ ← backups     │
│  core-unicom/           │             │                                  │
│  UNI-FIC-BE-UAT/        │             │  /home/ec2-user/OnDemandAPIs/    │
│                         │             │    OnDemandCommAPIs/             │
│  JFrog: 172.24.20.44    │             │      application.properties      │
└─────────────────────────┘             └──────────────────────────────────┘
              │
              ▼
     JFrog Artifactory
     172.24.20.44
     Core-unicom/UNI-FIC-BE-UAT/
       ├── *.jar
       └── application.properties
```

---

## 📂 Repo Structure

```
ansible-jenkins-deployment/
├── playbooks/
│   ├── fic-jar-file-deployment.yaml    ← Downloads & deploys JAR from JFrog
│   ├── fic-app-conf-deployment.yaml    ← Downloads & deploys application.properties
│   ├── fic-service-restart.yaml        ← Restarts systemd service
│   └── fic-rollback.yaml               ← Emergency rollback to last backup
├── inventory/
│   └── hosts.ini                       ← Target server definitions
├── group_vars/
│   └── app_servers.yml                 ← All paths and variables
├── jenkins/
│   └── UNI-FIC-BE-UAT.jenkinsfile      ← Complete Jenkins pipeline
└── README.md
```

---

## 🔄 Pipeline Flow

```
Git Checkout (tag)
      ↓
Maven Build (mvn clean install)
      ↓
Upload JAR → JFrog Artifactory
      ↓
Upload Config → JFrog Artifactory
      ↓
Ansible: Backup + Deploy JAR     (fic-jar-file-deployment.yaml)
      ↓
Ansible: Backup + Deploy Config  (fic-app-conf-deployment.yaml)
      ↓
Ansible: Service Restart         (fic-service-restart.yaml)
      ↓
Health Check → ✅ or ❌ Fail
```

---

## ⚡ Flexible Deploy Types

The Jenkins pipeline supports 4 deploy modes via parameter:

| `DEPLOY_TYPE` | What runs |
|---------------|-----------|
| `full` | Build → Upload → Deploy JAR → Deploy Config → Restart |
| `jar-only` | Build → Upload JAR → Deploy JAR → Restart |
| `config-only` | Upload Config → Deploy Config → Restart |
| `service-restart-only` | Restart service only |

---

## ⏪ Rollback

To rollback to the previous JAR + config:
```
Jenkins → UNI-FIC-BE-UAT → Build with Parameters
→ Set ROLLBACK = true
→ Build
```
Or manually:
```bash
ansible-playbook playbooks/fic-rollback.yaml -i inventory/hosts.ini
```

---

## 🔧 Setup on Jenkins Server

### 1 — Copy playbooks to Jenkins server
```bash
sudo mkdir -p /var/lib/jenkins/core-unicom/UNI-FIC-BE-UAT
sudo cp -r playbooks/ inventory/ group_vars/ /var/lib/jenkins/core-unicom/UNI-FIC-BE-UAT/
sudo chown -R jenkins:jenkins /var/lib/jenkins/core-unicom/
```

### 2 — Add credentials in Jenkins
| Credential ID | Type | Used For |
|---------------|------|----------|
| `git-repo-credentials` | Username/Password | Git clone |
| `cp-fe-creds` | Username/Password | JFrog Artifactory |
| `ansible-vault-password` | Secret text | Ansible vault |

### 3 — Create Jenkins Pipeline job
```
New Item → Pipeline → UNI-FIC-BE-UAT
→ Pipeline script from SCM
→ SCM: Git → this repo
→ Script Path: jenkins/UNI-FIC-BE-UAT.jenkinsfile
→ Save
```

### 4 — Required Jenkins plugins
- Artifactory Plugin
- Ansible Plugin
- AnsiColor
- Credentials Binding

---

## 🔐 Security Notes

- JFrog credentials stored in Jenkins Credentials Manager — never hardcoded
- Ansible vault used for any sensitive vars in playbooks
- SSH key for target server stored at `/var/lib/jenkins/.ssh/id_rsa`
- `StrictHostKeyChecking=no` set only for internal network (172.24.x.x)

---

## 🤝 Author

**Amol Shinde** · Cloud & DevOps Engineer · Flentas Technologies  
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/amol-shinde-profile)
