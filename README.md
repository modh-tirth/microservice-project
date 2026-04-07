# Microservices Notes App — End-to-End Deployment Guide

## Architecture

```
Browser → Frontend (nginx:80)
             ↓
       API Gateway (:3000)
        ↙          ↘
User Service(:3001)  Notes Service(:3002)
        ↘          ↙
          MySQL DB
```

## Services
| Service | Port | Description |
|---|---|---|
| frontend | 80 | Static HTML served by nginx |
| api-gateway | 3000 | Routes requests to microservices |
| user-service | 3001 | Register/Login |
| notes-service | 3002 | CRUD notes |
| mysql | 3306 | Persistent database |

---

## Step 1 — EC2 Instance Setup

1. Launch an **Ubuntu 22.04** EC2 instance (t2.medium or higher recommended)
2. Open inbound ports in Security Group: `22`, `80`, `3000`, `8080` (Jenkins), `9000` (SonarQube)
3. SSH into the instance and run:

```bash
chmod +x ec2-setup.sh
./ec2-setup.sh
```

---

## Step 2 — Update Frontend IP

In `project/frontend/index.html`, replace:
```js
const API = "http://<EC2-PUBLIC-IP>:3000";
```
with your actual EC2 public IP, then commit and push.

---

## Step 3 — Jenkins Setup

### Install Jenkins on EC2
```bash
sudo apt-get install -y openjdk-17-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update && sudo apt-get install -y jenkins
sudo systemctl start jenkins
```

### Install required Jenkins plugins
- Git, Pipeline, Docker Pipeline, SonarQube Scanner, SSH Agent, Credentials Binding

### Add Jenkins Credentials (Manage Jenkins → Credentials)
| ID | Type | Value |
|---|---|---|
| `dockerhub-creds` | Username/Password | Docker Hub login |
| `ec2-host` | Secret text | `ubuntu@<EC2-PUBLIC-IP>` |
| `ec2-ssh-key` | SSH private key | Your EC2 `.pem` key |
| `sonar-token` | Secret text | SonarQube user token |

### Configure SonarQube server
Manage Jenkins → Configure System → SonarQube servers:
- Name: `SonarQube`
- URL: `http://localhost:9000` (or your SonarQube host)

---

## Step 4 — SonarQube Setup

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
```
- Open `http://<EC2-IP>:9000`, login with `admin/admin`, change password
- Create a project → generate a token → save as `sonar-token` in Jenkins

---

## Step 5 — Create Jenkins Pipeline

1. New Item → Pipeline
2. Pipeline Definition: **Pipeline script from SCM**
3. SCM: Git → your GitHub repo URL
4. Script Path: `Jenkinsfile`
5. Save and click **Build Now**

---

## Pipeline Stages

```
Checkout → SonarQube Analysis → Quality Gate → Build Images
       → Trivy Scan → Push to Docker Hub → Deploy to EC2
```

---

## Local Development

```bash
cd project
docker compose up --build
```
App available at `http://localhost`

---

## Security Notes
- Passwords in `docker-compose.yml` are for demo only — use Docker secrets or AWS Secrets Manager in production
- Trivy will **fail the build** on HIGH/CRITICAL CVEs — fix them before merging
- SonarQube Quality Gate will **block deployment** if code quality thresholds are not met
