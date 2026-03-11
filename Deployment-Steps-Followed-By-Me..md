

# DevOps CI/CD Setup Guide

This project sets up a ** The Ultimate CI/CD Corporate DevOps Pipeline Project** using Jenkins, SonarQube, Nexus, Docker, and Kubernetes.

# Final CI/CD Workflow

```
Code Commit from Git
     ↓
Jenkins Pipeline Trigger
     ↓
Compile the code
     ↓
Run Unit tests
     ↓
SonarQube Code Analysis
     ↓
OWASP Dependency Check
     ↓
Maven Build Package
     ↓
Deploy the artifact to Nexus repository
     ↓
Docker Image Build
     ↓
Scan Docker Image with Trivy
     ↓
Push Image to Registry
     ↓
Deploy to Kubernetes
```

## Infrastructure Setup
Create Security Group which allows Inbouncd traffic for multiple ports which can be used commonly for all upcoming servers.
<img width="1597" height="469" alt="Security Group" src="https://github.com/user-attachments/assets/b658bee5-1cac-4740-b161-65359642e18b" />


Create total 6 AWS EC2 Ubuntu Instances (c7i-flex.large instance type is enough for testing our environment) :-
| Service | Instances |
|-------|-------|
| Jenkins  | 1 |
| SonarQube | 1 |
| Nexus / Artifactory | 1 |
| Kubernetes Cluster | 3 (1 Master + 2 Worker Nodes) |

---
<img width="1646" height="389" alt="Screenshot (121)" src="https://github.com/user-attachments/assets/b7fa4a5f-5ce9-4330-91ac-afbbb4e00630" />


# 1. Jenkins Server Setup

## Install Required Tools on Jenkins AWS EC2 Instance :
- Jenkins (installation with suggested plugins)
- Docker
- Trivy
- Kubectl (via snap)



## Configure Docker Permissions

```bash
sudo usermod -aG docker $USER
sudo usermod -aG docker jenkins
sudo newgrp docker
```
---

## Install Jenkins Plugins

Install the following plugins which are required for our environment :
Dashboard → Manage Jenkins → Plugins -> Available Plugins -> Search and Install new plugins.

* SonarQube Scanner
* Docker
* Docker Pipeline
* Nexus Artifact Uploader
* OWASP Dependency Check
* Eclipse Temurin Installer (JDK)
* Pipeline Maven Integration
* Config File Provider
* Kubernetes
* Kubernetes CLI

---

## Configure Global Tools in Jenkins

Go to **Manage Jenkins → Global Tool Configuration** and configure:

| Tool              | Version           |
| ----------------- | ----------------- |
| JDK               | 17, 21            |
| Maven             | Maven 3           |
| SonarQube Scanner | Latest            |
| Dependency Check  | 6.5.1             |
| Docker            | 28.5.1 |

---

## Configure Jenkins Credentials

Add the following credentials:

### SonarQube Token

* Type: **Secret Text**
* Store SonarQube authentication token.

### Docker Credentials

* Docker registry login credentials.

### Nexus Credentials

* Username & Password for:

  * `maven-releases`
  * `maven-snapshots`

### Kubernetes Token

* Token generated from Kubernetes service account.

---
<img width="1580" height="579" alt="Screenshot (137)" src="https://github.com/user-attachments/assets/08c8d4be-2378-4e15-a035-31867f9dd7d4" />

## Jenkins System Configuration

1. Go to **Manage Jenkins → System Configuration**
2. Add **SonarQube Server**
3. Provide:

   * SonarQube URL
   * Authentication Token

---
<img width="1557" height="631" alt="Screenshot (138)" src="https://github.com/user-attachments/assets/b98a40ba-c23d-47a4-94ac-4ffcb30212b7" />

## Configure Maven Repository Credentials

Use **Config File Provider Plugin**.

Add Nexus credentials for:

* `maven-releases`
* `maven-snapshots`

---

# 2. SonarQube Server Setup

Install Docker on SonarQube AWS EC2 Instance.

Run SonarQube container:

```bash
docker run -d -p 9000:9000 --name sonarqube sonarqube:lts-community
```

Access SonarQube:

```
http://<sonarqube-server-ip>:9000
```

### Create SonarQube Token

1. Login to SonarQube
2. Go to **My Account → Security** -> Enter token name -> User Type -> Generate
3. Generate a **Token**
4. Save it in **Jenkins Credentials** as secret text

---
<img width="1609" height="134" alt="Screenshot (132)" src="https://github.com/user-attachments/assets/01af82e7-d619-4331-8471-f75e6c92bddc" />


# 3. Nexus Repository Setup

Install Docker on Nexus AWS EC2 Instance.

Run Nexus container:

```bash
docker run -d -p 8081:8081 --name nexus sonatype/nexus3
```

Access Nexus:

```
http://<nexus-server-ip>:8081
```

### Get Default Admin Password

```bash
docker exec -it container_id /bin/bash cat /sonatype-work/nexus3/admin.password
```

Login with:

```
Username: admin
Password: <password-from-file>
```

---
<img width="1608" height="133" alt="Screenshot (131)" src="https://github.com/user-attachments/assets/c56d14d7-d185-4b6a-a6e1-c43fb7800ec6" />
<img width="1920" height="848" alt="Screenshot (139)" src="https://github.com/user-attachments/assets/5fc4edb5-147c-487b-aa4f-76a51b5bcfad" />


# 4. Maven Configuration

Update **pom.xml** with Nexus repositories.

```xml
<distributionManagement>
    <repository>
        <id>maven-releases</id>
        <url>http://NEXUS-INSTANCE-IP:8081/repository/maven-releases/</url>
    </repository>

    <snapshotRepository>
        <id>maven-snapshots</id>
        <url>http://NEXUS-INSTANCE-IP:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

---

# 5. Kubernetes Cluster Setup

## Prepare 3 Ubuntu Instances

* 1 Master Node
* 2 Worker Nodes

Install on all nodes:

* Docker
* Kubernetes (kubeadm, kubelet, kubectl)

---

## Initialize Kubernetes Cluster (Master Node)

```bash
kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubeconfig (Master Node):

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If you face issue with config file or port 8080 connection error you can run followwing command \
and again run the kubeconfig commands (Master Node) :
```bash
sudo systemctl status containerd
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
```


---

## Join Worker Nodes (Worker Node) :

Run the join command generated by `kubeadm init` on both worker nodes.

Example:

```bash
kubeadm join 172.31.26.74:6443 --token yu4p9a.swbi31qhq7fq8hfd \
--discovery-token-ca-cert-hash sha256:148af2f6f5b2684b47f70a27da5e9df2982996a27519b07030251d25adc5a1c6 
```
<img width="1601" height="847" alt="Screenshot (144)" src="https://github.com/user-attachments/assets/959278e0-d1c4-43d6-9f20-69d896d84ef7" />

---

## Create Namespace (Master Node)

```bash
kubectl create namespace webapps
```

---

## Create Service Account (jenkins) (Master Node)

Create `serviceaccount.yml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

Apply:

```bash
kubectl apply -f serviceaccount.yml
```

---

## Create Role (Master Node)

Create `role.yml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply:

```bash
kubectl apply -f role.yml
```

---

## Bind Role to Service Account (Master Node)

Create `rolebinding.yml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```

Apply:

```bash
kubectl apply -f rolebinding.yml
```

---

## Create Secret Token (Master Node)
Create `secret-token.yml`

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins
```

```bash
kubectl apply -f secret-token.yml -n webapps
```

View secret token:
```bash
kubectl -n webapps describe secret mysecretname
```
```
Save this Kubernetes Secret Token into Jenkins Credentials .
```

View kubeconfig for checking kubeadm server URL: (Master Node)

```bash
cat ~/.kube/config
```

---

# 6. Jenkins Pipeline Integration

### Configure Docker Pipeline Steps

Use Jenkins pipeline to:
1. Git Code checkout
2. Compile the code
3. Run unit tests
4. Scan with SonarQube
5. Run OWASP Dependency Check
6. Build the package
7. Deploy the artifact to Nexus
8. Build Docker Image & Tag
9. Scan Docker Image with Trivy
10. Push Docker Image to Registry
12. Deploy to Kubernetes Cluster

---

# Required Jenkins Plugins

| Plugin                     |
| -------------------------- |
| Docker                     |
| Docker Pipeline            |
| SonarQube Scanner          |
| OWASP Dependency Check     |
| Nexus Artifact Uploader    |
| Pipeline Maven Integration |
| Config File Provider       |
| Kubernetes                 |
| Kubernetes CLI             |
---
<img width="1920" height="863" alt="Screenshot (122)" src="https://github.com/user-attachments/assets/4e2585cd-a6cc-4ca3-ba91-ef0da8afb8fe" />
<img width="1920" height="845" alt="Screenshot (123)" src="https://github.com/user-attachments/assets/710b2fee-1fc2-4f0f-a24b-3fc6fc2bfade" />
<img width="1920" height="299" alt="Screenshot (124)" src="https://github.com/user-attachments/assets/96309986-499c-4737-a084-5a1617ede0a9" />
<img width="1920" height="445" alt="Screenshot (125)" src="https://github.com/user-attachments/assets/b098f4a9-bfec-4d12-a0e4-ce28ff3dc2df" />
<img width="1920" height="859" alt="Screenshot (126)" src="https://github.com/user-attachments/assets/5e654a0c-8a94-4416-b233-72f8ccd185e3" />
<img width="1920" height="447" alt="Screenshot (127)" src="https://github.com/user-attachments/assets/142c17a9-1b2a-4f74-bc22-994897f3d3ce" />
<img width="1920" height="795" alt="Screenshot (128)" src="https://github.com/user-attachments/assets/28b655df-4772-4286-9402-b6f20ba125db" />
<img width="1920" height="375" alt="Screenshot (129)" src="https://github.com/user-attachments/assets/439f3c17-9f72-4244-9b0c-145a694e68e7" />
<img width="812" height="168" alt="Screenshot (130)" src="https://github.com/user-attachments/assets/6b87829e-5408-4795-8264-486d02134246" />
<img width="1020" height="324" alt="Screenshot (133)" src="https://github.com/user-attachments/assets/b25ba249-014b-4a9f-bd9a-1d8b696cd3b5" />

---

     pipeline {
         agent any
         tools {
             maven 'maven3'
             jdk 'jdk17'
         }
         environment {
             SCANNER_HOME=tool 'sonar-scanner'
         }

    stages {
        stage('Git checkout') {
            steps {
               git branch: 'main', url: 'https://github.com/pankajmasaye88/Ekart.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Unit test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
               withSonarQubeEnv('sonar') {
                sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=EKart -Dsonar.projectName=EKart \
                    -Dsonar.java.binaries=. '''
                }
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: ' --scan ./', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
         stage('Build & Tag Docker Image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t docker123pankaj/ekart:latest -f docker/Dockerfile ."
                    }
                }
            }
        }
        
         stage('Trivy Scan') {
            steps {
                sh "trivy image docker123pankaj/ekart:latest > trivy-report.txt "
            }
        }
        
         stage('Push Docker Image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push docker123pankaj/ekart:latest"
                    }
                }
            }
        }
        
         stage('Kubernetes Deploy') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.26.74:6443') {
                 sh "kubectl apply -f deploymentservice.yml -n webapps"
                 sh "kubectl get svc -n webapps"
                }   
            }
        }
         }
     }

---





