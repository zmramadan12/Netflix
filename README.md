**Netflix Clone Deployment on AWS - DevSecOps Project
This repository contains a complete DevSecOps pipeline for deploying a Netflix clone application on Aws, implementing security scanning, monitoring, and Kubernetes deployment.
**

<img width="1919" height="1026" alt="Screenshot 2025-09-02 011125" src="https://github.com/user-attachments/assets/beff81bd-a5de-4858-b8ec-dbba5b2dc6d5" />

<img width="1918" height="1024" alt="Screenshot 2025-09-02 040500" src="https://github.com/user-attachments/assets/b83cf238-5575-48a8-a901-112fb03b22b4" />

<img width="1917" height="1019" alt="Screenshot 2025-09-03 120559" src="https://github.com/user-attachments/assets/66aa5a39-3b42-4536-8484-5a98c556a5f0" />



## Project Overview

This project demonstrates a complete CI/CD pipeline with integrated security (DevSecOps) for a Netflix clone application, deployed on Aws infrastructure. The pipeline includes:

1. **Code Build & Test**: Jenkins pipeline for building and testing the application
2. **Security Scanning**: SonarQube, OWASP Dependency Check, and Trivy for vulnerability scanning
3. **Containerization**: Docker for containerizing the application
4. **Orchestration**: Elastic Kubernetes Service (EKS) for container orchestration
5. **GitOps**: ArgoCD for declarative deployments
6. **Monitoring**: Prometheus and Grafana for monitoring

## Prerequisites

- AWS account with active subscription
-  AWS t3.medium
- Basic knowledge of Jenkins, Docker, and Kubernetes

## Infrastructure Setup

### 1. Create EC2

Instead of AWS EC2, this project uses t3.medium. EC2 automatically create public IPs, simplifying access to the services.

### 2. Configure Network Security Group

Ensure the following ports are open in your Aws Network Security Group:

- 8080: Jenkins
- 8081: Netflix application
- 9000: SonarQube
- 9090: Prometheus
- 3000: Grafana
- 9100: Node Exporter

## Application Deployment Steps

### Phase 1: Initial Setup and Deployment

#### Step 1: Connect to EC2 

```bash
ssh awseuser@<your-ec2-ip>
```

#### Step 2: Clone the Repository

```bash
git clone https://github.com/zmramadan12/Netflix.git
cd Netflix
```

#### Step 3: Install Docker

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

#### Step 4: Build and Run with Docker

Get an API key from [TMDB](https://www.themoviedb.org/) by creating an account and generating an API key in the settings.

```bash
# Build with your API key
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .

# Run the container
docker run -d --name netflix -p 8081:80 netflix:latest
```

Access the application at: http://ec2-ip:8081

### Phase 2: Security Setup

#### Step 1: Install SonarQube

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access SonarQube at: http://your-ec2-ip:9000 (default credentials: admin/admin)

#### Step 2: Install Trivy

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

### Phase 3: CI/CD Setup with Jenkins

#### Step 1: Install Java

```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre -y
```

#### Step 2: Install Jenkins

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Access Jenkins at: http://your-ec2-ip:8080

sudo cat /var/lib/jenkins/secrets/initialAdminPassword

#### Step 3: Install Required Jenkins Plugins

Install the following plugins:
- Eclipse Temurin Installer
- SonarQube Scanner
- NodeJs Plugin
- Email Extension Plugin
- OWASP Dependency-Check
- Docker plugins (Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step)

#### Step 4: Configure Jenkins Tools

1. Go to Manage Jenkins → Tools
2. Configure JDK 17 and NodeJS 16
3. Configure SonarQube Scanner
4. Configure Dependency-Check (name: "DP-Check")

#### Step 5: Add Docker Hub Credentials

1. Go to Manage Jenkins → Credentials → System → Global credentials
2. Add credentials:
   - For Docker Hub: Use Personal Access Token (PAT) with read/write access instead of password
   - For SonarQube: Create token from SonarQube admin panel

#### Step 6: Create Jenkins Pipeline

Create a Jenkins pipeline with the following configuration:

pipeline{
    agent any
    tools{
        jdk 'JDK 17'
        nodejs 'NodeJS'
    }
    environment {
        SCANNER_HOME=tool 'SonarQube Scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/zmramadan12/Netflix.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=bcc39d90918371b9a32da24935fd4c80 -t netflix ."
                       sh "docker tag netflix zhdmra/netflix:latest"
                       sh "docker push zhdmra/netflix:latest"
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image zhdmra/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d -p 8081:80 zhdmra/netflix:latest'
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }

    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'zahidmramadan@gmail.com', 
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}

**Note:** If you encounter a "docker login failed" error:

```bash
sudo su
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Phase 4: Monitoring Setup

For monitoring, a separate ec2 was created to host Prometheus and Grafana.

#### Step 1: Create Monitoring VM

# Open ports
ec2 open-port --resource-group netflix-devsecops --name monitoring-vm --port 9090,3000,9100 --priority 1001
```

#### Step 2: Install Prometheus

```bash
# Create user
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Download and install Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/

sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

Configure Prometheus service:

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Add the following content:

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Start Prometheus:

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

#### Step 3: Install Node Exporter

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

Configure Node Exporter service:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Add the following content:

```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

Start Node Exporter:

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

#### Step 4: Configure Prometheus to Monitor Jenkins

Update Prometheus configuration:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add the following configuration:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['jenkins-vm-ip:8080']
      
  - job_name: 'Netflix'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['jenkins-vm-ip:9100']
```

Reload Prometheus configuration:

```bash
curl -X POST http://localhost:9090/-/reload
```

#### Step 5: Install Grafana

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Access Grafana at http://monitoring-vm-ip:3000 (default credentials: admin/admin)

Configure Grafana:
1. Add Prometheus as a data source (URL: http://localhost:9090)
2. Import dashboard templates (e.g., Node Exporter dashboard #1860)

### Phase 5: Kubernetes Deployment with EKS

#### Step 1: Create EKS Cluster

# Install Aws CLI
# Create EKS cluster

#### Step 2: Install ArgoCD

Install ArgoCD on the Kubernetes cluster:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose ArgoCD UI
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get ArgoCD password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Access ArgoCD UI at the LoadBalancer IP (use `kubectl get svc -n argocd` to find it)

#### Step 3: Deploy Application with ArgoCD

Create an application in ArgoCD using the UI or CLI:

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: netflix-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zmramadan12/Netflix.git
    targetRevision: HEAD
    path: kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

The Kubernetes directory in the repository contains:
- deployment.yaml
- node-service.yaml
- service.yaml

#### Step 4: Access the Application

To access your application running in Kubernetes, you have two options:

1. **Using NodePort Service**:
   
   Verify the service is created with NodePort type:
   ```bash
   kubectl get svc
Access the application at: http://node-external-ip:30007

2. **Using kubectl port-forward**:
   ```bash
   # Forward the service port to your local machine
   kubectl port-forward svc/netflix-service 8081:80
   ```
   
   Access the application at: http://localhost:8081

3. **Verify deployment status**:
   ```bash
   # Check if pods are running
   kubectl get pods
   
   # Check logs if needed
   kubectl logs <pod-name>
   
   # Describe the service
   kubectl describe svc netflix-service
   ```

You can also see the deployment status in the ArgoCD UI, which will show you the sync status and health of your application components.

## Repository Structure

```
├── Dockerfile
├── README.md
├── Jenkinsfile
├── package.json
├── kubernetes/
│   ├── deployment.yaml
│   ├── node-service.yaml
│   └── service.yaml
└── src/
    └── [Application Source Files]
```

## Conclusion

This project demonstrates a complete DevSecOps pipeline for deploying a Netflix clone application on Aws infrastructure. The implementation includes:

- CI/CD with Jenkins
- Security scanning with SonarQube, OWASP Dependency Check, and Trivy
- Containerization with Docker
- Orchestration with Elastic Kubernetes Service
- GitOps with ArgoCD
- Monitoring with Prometheus and Grafana

