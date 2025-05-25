# 🚀 Jenkins CI/CD Pipeline: Java App → Docker → ECR (IAM Role Auth)

This project demonstrates how to build a Java application using Jenkins, create a Docker image on a worker node, and **push it directly to AWS ECR** using EC2 IAM role authentication (without credentials or login commands).

---

## 🧰 Tech Stack

- **Java** (Simple Java CLI app)
- **Jenkins** (Master + Worker setup)
- **Docker**
- **AWS EC2** (with IAM role)
- **AWS ECR**

---

## ✅ Prerequisites

- AWS account
- IAM Role with:
  - `AmazonEC2ContainerRegistryFullAccess`
  - `AmazonEC2FullAccess`
- 2 EC2 instances:
  - Jenkins Master
- Docker installed on master


---

## ⚙️ Setup Steps

### 1. Launch EC2 Instances

- One for Jenkins Master
- One for Jenkins Worker

Attach the **same IAM role** (with ECR permissions) to both instances or **at least to the worker node** (where Docker runs).

---

### 2. Install Jenkins on Master

```bash
sudo yum update -y
sudo yum install java-11-openjdk -y
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

---

### 3. Configure Jenkins Agent (Worker)

- Install Docker and Java
- Add worker node via Jenkins > Manage Nodes
- Use SSH credentials for agent setup

---

### 4. Create ECR Repository

```bash
aws ecr create-repository --repository-name java-app-repo
```

Copy the ECR URI:  
`<aws_account_id>.dkr.ecr.<region>.amazonaws.com/java-app-repo`

---

## 📁 Project Structure

```
java-ecr-pipeline/
├── Dockerfile
├── Jenkinsfile
├── README.md
└── src/
    └── main/
        └── java/
            └── App.java
```

---

### 🐳 Dockerfile

```Dockerfile
FROM openjdk:11
WORKDIR /app
COPY . .
CMD ["java", "src/main/java/App"]
```

---

### 🔧 Jenkinsfile

```groovy
pipeline {
    agent { label 'worker-node' }

    environment {
        ECR_REPO = '<your_ecr_uri>' // e.g., 123456789012.dkr.ecr.ap-south-1.amazonaws.com/java-app-repo
        IMAGE_TAG = "java-app:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-username/java-ecr-pipeline.git'
            }
        }

        stage('Build Java App') {
            steps {
                sh 'javac src/main/java/App.java'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_TAG .'
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                docker tag $IMAGE_TAG $ECR_REPO:$IMAGE_TAG
                docker push $ECR_REPO:$IMAGE_TAG
                '''
            }
        }
    }
}
```

> ✅ No `aws ecr get-login-password` or credentials block needed.

---

## 🧪 Triggering the Pipeline

1. Go to Jenkins
2. Create a new pipeline job
3. Point it to your GitHub repo (with `Jenkinsfile`)
4. Click **Build Now**

---

## 🔐 IAM Role Clarification

- IAM Role is **automatically used** by AWS CLI & Docker if attached to the EC2
- No need to configure AWS credentials
- Works because AWS SDK detects EC2 role via instance metadata

---

## 📌 Notes

- Make sure the worker node has Docker and IAM role attached
- The master doesn’t need Docker unless you build on it
- Use pipeline label to target the worker node

---
