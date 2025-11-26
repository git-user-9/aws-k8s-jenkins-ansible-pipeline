# CDD Project: Automated Kubernetes Deployment Pipeline

![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Jenkins](https://img.shields.io/badge/jenkins-%232C5263.svg?style=for-the-badge&logo=jenkins&logoColor=white)
![Ansible](https://img.shields.io/badge/ansible-%23EE0000.svg?style=for-the-badge&logo=ansible&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)

## Documentation & Demo
[![Demo Video](https://img.shields.io/badge/Watch-Demo_Video-red?style=for-the-badge&logo=youtube&logoColor=white)](https://drive.google.com/file/d/1O1qMwqNF7bEx_HFP203xzYETr3QvB-mA/view?usp=drive_link)
[![Documentation](https://img.shields.io/badge/View-Documentation_PDF-blue?style=for-the-badge&logo=google-drive&logoColor=white)](https://drive.google.com/file/d/1RkjEaUwDW2s_avlP-o9zmb7GhtcJAPUJ/view?usp=drive_link)

## 📖 Overview

This project demonstrates a complete **DevOps lifecycle (CDD - Continuous Delivery/Deployment)** for deploying the *Loxury Jewellery* web application.

The pipeline automates the fetching of code from GitHub, building a Docker image via Jenkins/Ansible, pushing it to Docker Hub, and deploying it to a high-availability **Kubernetes cluster on AWS** managed by Kops.

## 🏗 Architecture

The infrastructure consists of **5 EC2 instances** in total (3 Management + 2 Cluster Nodes):

| Instance | Type | Role |
| :--- | :--- | :--- |
| **Jenkins Server** | `t2.micro` | Handles CI, source code management, and triggers build artifacts. |
| **Ansible Server** | `t2.micro` | Configuration management node that builds Docker images and triggers K8s deployments. |
| **Controller VM** | `t2.micro` | Jump Host used to provision the K8s cluster using `kops`. |
| **Kubernetes Master** | `t3.medium` | Control plane for the K8s cluster. |
| **Kubernetes Node** | `t3.medium` | Worker node hosting the application pods. |

## 🚀 Tech Stack

*   **Cloud Provider**: AWS (EC2, Route53, S3, IAM, VPC)
*   **Orchestration**: Kubernetes (via Kops)
*   **CI/CD**: Jenkins & Ansible
*   **Containerization**: Docker
*   **Web Server**: Apache (serving a static HTML template)
*   **OS**: Ubuntu Linux

## 🛠 Prerequisites

Before running this pipeline, ensure you have:

- [ ] **AWS Account** with permissions for EC2, S3, Route53, and IAM.
- [ ] **Domain Name** configured in Route53 (e.g., `devops.cdd.in`).
- [ ] **Docker Hub Account** (e.g., `docky9`) for pushing images.

## ⚙️ Infrastructure Setup

### 1. AWS IAM & S3

*   Create an IAM Role `cdd-project-kube-jenk` with **AdministratorAccess** (or specific EC2/S3/Route53/VPC full access).
*   Create an S3 bucket (e.g., `clusters.devops.cdd.in`) to store the Kops cluster state.

### 2. Server Provisioning

Launch instances as shown in the architecture. Install necessary tools:

*   **Jenkins**: Java 17, Jenkins, Git, "Publish Over SSH" Plugin.
*   **Ansible**: Ansible, Docker.
*   **Controller**: `kubectl`, `kops`, AWS CLI.

### 3. Kubernetes Cluster (Kops)

Run the following on the **Controller VM** to create the cluster:

```bash
export KOPS_STATE_STORE=s3://clusters.devops.cdd.in
kops create cluster --cloud aws --zones ap-south-1a --name=dev.k8s.project.com --dns-zone=project.com --dns private
kops update cluster dev.k8s.project.com --yes --admin
kops validate cluster --wait 10m
```

## 🔄 Jenkins Pipeline Configuration

The project uses a **Jenkinsfile** to automate the following steps:

### 1. Source Code Management
*   **Repository URL**: `https://github.com/git-user-9/cdd-react.git`
*   **Branch**: `*/main`

### 2. Build Triggers
*   **GitHub Hook**: Triggers on push events.
*   **Webhook URL**: `http://<JENKINS_IP>:8080/github-webhook/`

### 3. Build Steps (File Transfer)
Uses `rsync` to sync workspace files to the Ansible server:

```bash
rsync -avh /var/lib/jenkins/workspace/cddproj/* root@<ANSIBLE_IP>:/opt
```

### 4. Post-Build Actions (Docker Build & Deploy)
Executes commands on the Ansible Server via SSH:

**Step A: Build and Push Docker Image**
```bash
cd /opt/app
docker image build -t $JOB_NAME:v1.$BUILD_ID .
docker image tag $JOB_NAME:v1.$BUILD_ID docky9/$JOB_NAME:v1.$BUILD_ID
docker image tag $JOB_NAME:v1.$BUILD_ID docky9/$JOB_NAME:latest
docker image push docky9/$JOB_NAME:v1.$BUILD_ID
docker image push docky9/$JOB_NAME:latest
docker image rmi $JOB_NAME:v1.$BUILD_ID docky9/$JOB_NAME:v1.$BUILD_ID docky9/$JOB_NAME:latest
```

**Step B: Trigger Kubernetes Deployment**
```bash
ansible-playbook /opt/ansible/ansible.yml
```

## 📂 Configuration Snippets

### Dockerfile (Apache Web App)
*Located in `app/Dockerfile`*

```dockerfile
FROM ubuntu:latest
RUN apt-get update && \
    apt-get install -y apache2 zip unzip && \
    rm -rf /var/lib/apt/lists/*
ADD https://www.free-css.com/assets/files/free-css-templates/download/page258/loxury.zip /var/www/html/
WORKDIR /var/www/html
RUN unzip loxury.zip && \
    cp -rvf loxury/* . && \
    rm -rf loxury loxury.zip
CMD ["apache2ctl", "-D", "FOREGROUND"]
EXPOSE 80
```

### Kubernetes Deployment
*Located in `k8s/deployment.yaml`*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cddproj
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cddproj
  template:
    metadata:
      labels:
        app: cddproj
    spec:
      containers:
      - name: cddproj
        image: docky9/cddproj:latest
        ports:
        - containerPort: 80
```

## 📸 Verification

1.  **Jenkins Console**: Ensure the "SSH: EXEC: completed" messages appear for both the rsync and docker commands.
2.  **Kubernetes**: Run `kubectl get all` on the Controller VM to see the running pods and LoadBalancer service.
3.  **Live URL**: Access the application via `http://<Worker-Node-Public-IP>:31200`.
