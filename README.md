# BankApp CI/CD Pipeline

## 🚀 Project Overview
A complete CI/CD implementation for a Spring Boot banking application deployed on AWS EKS using Jenkins, Docker, and Kubernetes.

## 🏗️ Architecture
- **Application**: Spring Boot + MySQL
- **CI Pipeline**: Jenkins (Build, Test, Docker Image)
- **CD Pipeline**: Jenkins + Kubernetes (EKS Deployment)
- **Infrastructure**: AWS EKS with 3 nodes
- **Container Registry**: Docker Hub

## 📋 Prerequisites
- AWS Account with EKS cluster
- Jenkins Server (Ubuntu)
- Docker installed
- kubectl configured
- GitHub account
- Docker Hub account

## 🔧 Technology Stack
- **Backend**: Spring Boot 3.x, Java 17
- **Database**: MySQL 8.0
- **Testing**: JUnit 5, H2 (in-memory for tests)
- **CI/CD**: Jenkins Pipeline
- **Containerization**: Docker
- **Orchestration**: Kubernetes (AWS EKS)
- **Version Control**: Git/GitHub

---

## 📦 CI Pipeline Setup

### Step 1: Configure Jenkins Server

1. **Install Java 17**:
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
```

2. **Install Maven**:
```bash
sudo apt install -y maven
```

3. **Install Docker**:
```bash
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

4. **Install Trivy** (Security Scanning):
```bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install -y trivy
```

### Step 2: Configure Jenkins Tools

1. Go to **Manage Jenkins** → **Tools**
2. Add JDK:
   - Name: `Java17`
   - JAVA_HOME: `/usr/lib/jvm/java-17-openjdk-amd64`
3. Add Maven:
   - Name: `maven3`
   - Install automatically from Apache

### Step 3: Configure Docker Hub Credentials

1. Go to **Manage Jenkins** → **Credentials**
2. Add new credentials:
   - Kind: Username with password
   - ID: `docker-cred`
   - Username: Your Docker Hub username
   - Password: Your Docker Hub password/token

### Step 4: Create CI Pipeline

1. Create new Pipeline job named `CI`
2. Configure:
   - Pipeline script from SCM
   - Repository: `https://github.com/YOUR_USERNAME/BankApp-CI.git`
   - Branch: `main`
   - Script Path: `Jenkinsfile`

### Step 5: CI Jenkinsfile Structure

```groovy
pipeline {
    agent any
    
    tools {
        maven 'maven3'
        jdk 'Java17'
    }
    
    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/YOUR_USERNAME/BankApp-CI.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t YOUR_USERNAME/bankapp:${IMAGE_TAG} ."
                    }
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push YOUR_USERNAME/bankapp:${IMAGE_TAG}"
                    }
                }
            }
        }
    }
}
```

---

## 🧪 Test Configuration (H2 Database)

### Problem
Spring Boot tests were failing because they tried to connect to MySQL database which wasn't available during CI builds.

### Solution: H2 In-Memory Database for Tests

#### 1. Add H2 Dependency to `pom.xml`:
```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

#### 2. Create Test Configuration File:
**File**: `src/test/resources/application-test.properties`
```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.H2Dialect
```

#### 3. Update Test Class:
**File**: `src/test/java/com/example/bankapp/BankappApplicationTests.java`
```java
package com.example.bankapp;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")  // This activates the test profile
class BankappApplicationTests {

    @Test
    void contextLoads() {
    }
}
```

### How It Works:
1. **@ActiveProfiles("test")**: Tells Spring to use `application-test.properties`
2. **H2 Database**: In-memory database that starts/stops with tests
3. **No External Dependencies**: Tests run independently without MySQL
4. **Fast Execution**: In-memory database is much faster than real database

---

## 🚢 CD Pipeline Setup

### Step 1: Install kubectl and AWS CLI

```bash
# Install kubectl
curl -LO https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Step 2: Configure AWS Credentials

```bash
# For Jenkins user
sudo -u jenkins mkdir -p /var/lib/jenkins/.aws
sudo -u jenkins aws configure
# Enter: Access Key, Secret Key, Region (ap-south-1), Output (json)
```

### Step 3: Configure kubectl for EKS

```bash
sudo -u jenkins aws eks update-kubeconfig --region ap-south-1 --name YOUR_CLUSTER_NAME
```

### Step 4: Create Kubernetes Manifests

**Database Deployment** (`database/database-ds.yml`):
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "Test@123"
        - name: MYSQL_DATABASE
          value: "bankappdb"
        ports:
        - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
```

**Application Deployment** (`bankapp/bankapp-ds.yml`):
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bankapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bankapp
  template:
    metadata:
      labels:
        app: bankapp
    spec:
      containers:
      - name: bankapp
        image: YOUR_USERNAME/bankapp:v24
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://mysql-service:3306/bankappdb?useSSL=false&serverTimezone=UTC
        - name: SPRING_DATASOURCE_USERNAME
          value: root
        - name: SPRING_DATASOURCE_PASSWORD
          value: Test@123
---
apiVersion: v1
kind: Service
metadata:
  name: bankapp-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: bankapp
```

### Step 5: Create CD Pipeline

1. Create new Pipeline job named `CD`
2. Configure:
   - Pipeline script from SCM
   - Repository: `https://github.com/YOUR_USERNAME/BankApp-CD.git`
   - Branch: `main`
   - Script Path: `Jenkinsfile`

### Step 6: CD Jenkinsfile

```groovy
pipeline {
    agent any
    
    environment {
        EKS_CLUSTER = 'YOUR_CLUSTER_NAME'
        AWS_REGION = 'ap-south-1'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/YOUR_USERNAME/BankApp-CD.git'
            }
        }
        
        stage('Update Kubeconfig') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
            }
        }
        
        stage('Deploy Database') {
            steps {
                sh "kubectl apply -f database/database-ds.yml"
                sh "kubectl rollout status deployment/mysql --timeout=300s"
            }
        }
        
        stage('Deploy Application') {
            steps {
                sh "kubectl apply -f bankapp/bankapp-ds.yml"
                sh "kubectl rollout status deployment/bankapp --timeout=300s"
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh "kubectl get pods"
                sh "kubectl get svc"
            }
        }
    }
}
```

---

## 🎯 Deployment Steps

### Complete Deployment Flow:

1. **Push Code to GitHub**:
```bash
git add .
git commit -m "Your changes"
git push origin main
```

2. **Trigger CI Pipeline**:
   - Go to Jenkins → CI job
   - Click "Build Now"
   - Pipeline will: Compile → Test → Build → Create Docker Image → Push to Docker Hub

3. **Trigger CD Pipeline**:
   - Go to Jenkins → CD job
   - Click "Build Now"
   - Pipeline will: Deploy MySQL → Deploy Application → Verify

4. **Access Application**:
```bash
kubectl get svc bankapp-service
# Copy the EXTERNAL-IP (LoadBalancer URL)
# Access: http://EXTERNAL-IP
```

---

## 🔍 Verification Commands

```bash
# Check pods
kubectl get pods

# Check services
kubectl get svc

# Check deployments
kubectl get deployments

# View logs
kubectl logs -l app=bankapp

# Describe pod
kubectl describe pod POD_NAME
```

---

## 🐛 Troubleshooting

### Issue 1: Tests Failing with MySQL Connection Error
**Solution**: Implemented H2 in-memory database for tests (see Test Configuration section)

### Issue 2: Docker Image Pull Error
**Solution**: Ensure image is pushed to Docker Hub and tag matches in Kubernetes manifest

### Issue 3: Pods in CrashLoopBackOff
**Solution**: Check logs with `kubectl logs POD_NAME` and verify environment variables

### Issue 4: LoadBalancer Pending
**Solution**: Wait 2-3 minutes for AWS to provision the LoadBalancer

---

## 📊 Pipeline Stages Explained

### CI Pipeline:
1. **Git Checkout**: Clone repository
2. **Compile**: Compile Java code
3. **Test**: Run unit tests with H2 database
4. **Build**: Create JAR file
5. **Docker Build**: Create container image
6. **Docker Push**: Push to Docker Hub

### CD Pipeline:
1. **Checkout**: Clone CD repository
2. **Update Kubeconfig**: Configure kubectl
3. **Deploy Database**: Deploy MySQL to Kubernetes
4. **Deploy Application**: Deploy BankApp to Kubernetes
5. **Verify**: Check deployment status

---

## 🎓 Key Learnings

1. **Test Isolation**: Use H2 for tests to avoid external dependencies
2. **Environment Separation**: Different configs for dev/test/prod
3. **High Availability**: Multiple replicas for zero-downtime
4. **Security**: Never hardcode credentials, use Kubernetes secrets
5. **Monitoring**: Always verify deployments with kubectl commands

---

## 📝 Best Practices Implemented

✅ Automated testing with isolated test database  
✅ Docker multi-stage builds for smaller images  
✅ Kubernetes health checks and readiness probes  
✅ LoadBalancer for external access  
✅ Environment-based configuration  
✅ Git-based workflow for version control  
✅ Separate CI and CD pipelines  
✅ High availability with 2 replicas  

---

## 🔗 Repository Links

- **CI Repository**: https://github.com/YOUR_USERNAME/BankApp-CI
- **CD Repository**: https://github.com/YOUR_USERNAME/BankApp-CD

---

## 👨‍💻 Author
Your Name  
DevOps Engineer

---

## 📄 License
MIT License
