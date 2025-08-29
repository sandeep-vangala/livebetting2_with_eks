# livebetting2_with_eks


# Complete Project: Deploying Live Betting Application on EKS with Monitoring (Prometheus, Grafana, AlertManager)

This guide provides a complete, self-contained project for provisioning an Amazon EKS (Elastic Kubernetes Service) cluster using Terraform, deploying the livebetting application from the GitHub repository (https://github.com/sandeep-vangala/livebetting.git), setting up Prometheus, Grafana, and AlertManager from scratch, and testing/troubleshooting alerts on a live cluster. The setup uses Terraform modules for cluster provisioning, Kubernetes manifests for deployment, and a Jenkinsfile for CI/CD automation.

I'll assume you have:
- AWS CLI configured with admin privileges.
- Terraform installed (v1.5+).
- kubectl installed.
- Jenkins installed (for CI/CD).
- Git installed to clone the repo.
- A Gmail account for AlertManager email notifications (as per the repo's .env example).

The project is designed to be modular and reproducible. We'll create an EKS cluster, deploy the app and monitoring stack, configure alerts, and test by simulating an issue (e.g., high JVM heap usage).

## Project Folder Structure

Organize your project in a new Git repository (e.g., `git init livebetting-eks-project`). Here's the structure:

```
livebetting-eks-project/
├── README.md                  # Project documentation (copy this guide here)
├── .env                       # Environment variables (e.g., for Gmail alerts)
├── terraform/                 # Terraform code for EKS provisioning
│   ├── main.tf                # Main Terraform configuration
│   ├── variables.tf           # Input variables
│   ├── outputs.tf             # Outputs (e.g., kubeconfig)
│   ├── provider.tf            # AWS provider
│   └── modules/               # (Optional: If using custom modules; here we'll use official ones)
├── k8s/                       # Kubernetes manifests (adapted from the repo's k8s/ folder)
│   ├── namespace.yaml         # Default namespace (optional)
│   ├── mysql/                 # MySQL deployment
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── secret.yaml        # MySQL secrets
│   ├── livebetting/           # App deployment
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml     # App config (e.g., application.properties)
│   ├── prometheus/            # Prometheus setup
│   │   ├── configmap.yaml     # Prometheus config and rules
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── grafana/               # Grafana setup
│   │   ├── secret.yaml        # Grafana secrets (including SMTP for alerts)
│   │   ├── pvc.yaml           # Persistent volume for data
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── alertmanager/          # AlertManager setup
│   │   ├── configmap.yaml     # AlertManager config
│   │   ├── secret.yaml        # AlertManager env secrets
│   │   ├── deployment.yaml    # With init container for config substitution
│   │   └── service.yaml
│   └── apply-all.sh           # Bash script to apply all manifests
├── jenkins/                   # CI/CD
│   └── Jenkinsfile            # Pipeline for build/deploy
├── scripts/                   # Helper scripts
│   ├── setup-cluster.sh       # Post-Terraform: Update kubeconfig
│   ├── test-alert.sh          # Script to simulate issue and test alerts
│   └── troubleshoot.sh        # Script for basic troubleshooting
├── postman/                   # Postman collection (from repo)
│   └── livebetting.postman_collection.json
└── docker/                    # Dockerfile for app (from repo)
    └── Dockerfile             # Multi-stage build for livebetting
```

Clone the original repo and copy relevant files:
```
git clone https://github.com/sandeep-vangala/livebetting.git temp-repo
cp -r temp-repo/k8s k8s/  # Copy original k8s manifests and adapt as below
cp temp-repo/Dockerfile docker/
cp temp-repo/postman_collection/* postman/
rm -rf temp-repo
```

Adapt the k8s manifests as shown below (the originals are for Minikube/k3d; we'll adjust for EKS with LoadBalancers).

## Step 1: Provision EKS Cluster with Terraform

Use the official Terraform AWS EKS module. Create the files in `terraform/`.

### terraform/provider.tf
```hcl
provider "aws" {
  region = var.region
}
```

### terraform/variables.tf
```hcl
variable "region" {
  default = "us-west-2"
  type    = string
}

variable "cluster_name" {
  default = "livebetting-eks"
  type    = string
}

variable "cluster_version" {
  default = "1.30"  # As of Aug 2025; check latest
  type    = string
}
```

### terraform/main.tf
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"  # Latest as of 2025

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id  # Create VPC first (see below)
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      min_size     = 1
      max_size     = 3
      desired_size = 2
      instance_types = ["t3.medium"]
    }
  }

  enable_cluster_creator_admin_permissions = true
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["${var.region}a", "${var.region}b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}
```

### terraform/outputs.tf
```hcl
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_name" {
  value = module.eks.cluster_name
}
```

Run Terraform:
```
cd terraform
terraform init
terraform apply -auto-approve
```

This creates an EKS cluster with 2 nodes. Note the outputs.

## Step 2: Configure kubectl

In `scripts/setup-cluster.sh`:
```bash
#!/bin/bash
aws eks update-kubeconfig --region us-west-2 --name livebetting-eks
kubectl get nodes  # Verify
```

Run: `./scripts/setup-cluster.sh`

## Step 3: Setup Prometheus, Grafana, AlertManager from Scratch

We'll use Kubernetes manifests adapted from the repo and Medium article in the document. These are deployed from scratch (no Helm; pure YAML for simplicity).

Create `.env` at root (from repo example):
```
# MySQL
DATABASE_USERNAME=root
DATABASE_PASSWORD=your_secure_password  # Change this!

# Grafana SMTP (for alerts)
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.gmail.com:587
GF_SMTP_USER=your_gmail@gmail.com
GF_SMTP_PASSWORD=your_gmail_app_password  # Generate app password in Google
GF_SMTP_SKIP_VERIFY=true
GF_SMTP_FROM_ADDRESS=your_gmail@gmail.com

# AlertManager
ALERT_RESOLVE_TIMEOUT=5m
SMTP_SMARTHOST=smtp.gmail.com:587
SMTP_FROM=your_gmail@gmail.com
SMTP_AUTH_USERNAME=your_gmail@gmail.com
SMTP_AUTH_PASSWORD=your_gmail_app_password
SMTP_REQUIRE_TLS=true
ALERT_EMAIL_TO=your_gmail@gmail.com  # Or another email
```

### MySQL (Database for App)
`k8s/mysql/secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
type: Opaque
data:
  username: cm9vdA==  # base64 of root
  password: eW91cl9zZWN1cmVfcGFzc3dvcmQ=  # base64 of your password
```

`k8s/mysql/deployment.yaml` (adapted from repo):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        envFrom:
          - secretRef:
              name: mysql-secrets
        ports:
        - containerPort: 3306
```

`k8s/mysql/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  selector:
    app: database
  ports:
  - port: 3306
  type: ClusterIP
```

### Livebetting App
Build and push Docker image (use your ECR repo; create via AWS CLI or Terraform add-on).
`docker/Dockerfile` (from repo).

Build:
```
docker build -t your-ecr-repo/livebetting:latest .
aws ecr get-login-password | docker login --username AWS --password-stdin your-account.dkr.ecr.us-west-2.amazonaws.com
docker push your-ecr-repo/livebetting:latest
```

`k8s/livebetting/configmap.yaml` (adapt application.properties):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: livebetting-config
data:
  application.properties: |
    spring.datasource.url=jdbc:mysql://database-service:3306/livebetting
    spring.datasource.username=root
    spring.datasource.password=your_secure_password
    # Other props from repo
    bet.slip.timeout=10
```

`k8s/livebetting/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: livebetting
spec:
  replicas: 1
  selector:
    matchLabels:
      app: livebetting
  template:
    metadata:
      labels:
        app: livebetting
    spec:
      containers:
      - name: livebetting
        image: your-ecr-repo/livebetting:latest
        ports:
        - containerPort: 5160
        envFrom:
          - configMapRef:
              name: livebetting-config
```

`k8s/livebetting/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: livebetting-service
spec:
  selector:
    app: livebetting
  ports:
  - port: 80
    targetPort: 5160
  type: LoadBalancer  # EKS exposes external IP
```

### Prometheus
`k8s/prometheus/configmap.yaml` (from document, adapted for heap alert):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    alerting:
      alertmanagers:
      - static_configs:
        - targets: ['alertmanager-service:9093']
    rule_files:
    - /etc/prometheus/alerts_rules.yml
    scrape_configs:
    - job_name: prometheus
      static_configs:
      - targets: ['localhost:9090']
    - job_name: livebetting
      metrics_path: /actuator/prometheus
      static_configs:
      - targets: ['livebetting-service:5160']
  alerts_rules.yml: |
    groups:
    - name: livebetting-alerts
      rules:
      - alert: HighHeapUsage
        expr: jvm_memory_used_bytes{area="heap"} > 200000000  # >200MB heap; adjust to trigger
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: High JVM Heap Usage in LiveBetting
          description: Heap usage exceeded 200MB for 1 min.
```

`k8s/prometheus/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
        - --config.file=/etc/prometheus/prometheus.yml
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
```

`k8s/prometheus/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
  type: LoadBalancer
```

### Grafana
`k8s/grafana/secret.yaml` (base64 encode from .env):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-secret
type: Opaque
data:
  GF_SECURITY_ADMIN_PASSWORD: YWRtaW4=  # admin
  # Add base64 for all GF_* from .env
  GF_SMTP_HOST: c210cC5nbWFpbC5jb206NTg3  # Example base64
  # ... (encode each)
```

`k8s/grafana/pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

`k8s/grafana/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana-oss:latest
        ports:
        - containerPort: 3000
        envFrom:
        - secretRef:
            name: grafana-secret
        volumeMounts:
        - name: storage
          mountPath: /var/lib/grafana
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: grafana-data
```

`k8s/grafana/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
spec:
  selector:
    app: grafana
  ports:
  - port: 3000
  type: LoadBalancer
```

### AlertManager
`k8s/alertmanager/configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
data:
  alertmanager.yml: |
    global:
      resolve_timeout: ${ALERT_RESOLVE_TIMEOUT}
      smtp_smarthost: ${SMTP_SMARTHOST}
      # ... (placeholders from .env)
    route:
      group_by: ['alertname']
      receiver: default
    receivers:
    - name: default
      email_configs:
      - to: ${ALERT_EMAIL_TO}
```

`k8s/alertmanager/secret.yaml` (base64 from .env):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-secret
type: Opaque
data:
  ALERT_RESOLVE_TIMEOUT: NW0=  # 5m
  # ... (all placeholders base64)
```

`k8s/alertmanager/deployment.yaml` (with init container for env substitution):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      initContainers:
      - name: config-init
        image: busybox
        envFrom:
        - secretRef:
            name: alertmanager-secret
        command: ['sh', '-c', 'envsubst < /template/alertmanager.yml > /config/alertmanager.yml']
        volumeMounts:
        - name: template
          mountPath: /template
        - name: config
          mountPath: /config
      containers:
      - name: alertmanager
        image: prom/alertmanager:latest
        args:
        - --config.file=/etc/alertmanager/alertmanager.yml
        ports:
        - containerPort: 9093
        volumeMounts:
        - name: config
          mountPath: /etc/alertmanager
      volumes:
      - name: template
        configMap:
          name: alertmanager-config
      - name: config
        emptyDir: {}
```

`k8s/alertmanager/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-service
spec:
  selector:
    app: alertmanager
  ports:
  - port: 9093
  type: LoadBalancer
```

Deploy all:
In `k8s/apply-all.sh`:
```bash
#!/bin/bash
kubectl apply -f mysql/
kubectl apply -f livebetting/
kubectl apply -f prometheus/
kubectl apply -f grafana/
kubectl apply -f alertmanager/
```

Run: `./k8s/apply-all.sh`

Get external IPs:
```
kubectl get svc  # Note LoadBalancer IPs for prometheus, grafana, alertmanager, livebetting
```

Access:
- Prometheus: http://<prometheus-external-ip>:9090
- Grafana: http://<grafana-external-ip>:3000 (admin/admin)
- AlertManager: http://<alertmanager-external-ip>:9093

In Grafana:
- Add Prometheus datasource: http://prometheus-service:9090
- Add AlertManager datasource: http://alertmanager-service:9093
- Import JVM Micrometer dashboard (ID 4701).
- Set up alert in Grafana on "Heap Used" panel: Threshold >200MB, notify via email.

## Step 4: Jenkinsfile for CI/CD

In `jenkins/Jenkinsfile` (Groovy DSL):
```groovy
pipeline {
    agent any
    environment {
        ECR_REPO = 'your-ecr-repo/livebetting'
        AWS_REGION = 'us-west-2'
        CLUSTER_NAME = 'livebetting-eks'
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/sandeep-vangala/livebetting.git', branch: 'main'
            }
        }
        stage('Build & Test') {
            steps {
                sh 'mvn clean install'
                // JaCoCo report: target/site/jacoco/index.html
                sh 'mvn sonar:sonar'  // If SonarQube setup
            }
        }
        stage('Docker Build & Push') {
            steps {
                withAWS(region: AWS_REGION, credentials: 'aws-creds') {
                    sh """
                    aws ecr get-login-password | docker login --username AWS --password-stdin your-account.dkr.ecr.${AWS_REGION}.amazonaws.com
                    docker build -t ${ECR_REPO}:latest .
                    docker push ${ECR_REPO}:latest
                    """
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                withAWS(region: AWS_REGION, credentials: 'aws-creds') {
                    sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}"
                    sh 'kubectl apply -f k8s/'  // Assumes k8s/ in repo
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts 'target/site/jacoco/index.html'
        }
    }
}
```

Setup in Jenkins: New Pipeline job, point to this Jenkinsfile. Trigger on push.

## Step 5: Testing and Troubleshooting

### Test Setup
1. Create a match via Postman: POST http://<livebetting-external-ip>/api/v1/matches/create (body from repo).
2. Place bets: POST /api/v1/betslips/create.

Verify in Prometheus: Query `jvm_memory_used_bytes{area="heap"}`.

### Create an Issue to Test Alerts
In `scripts/test-alert.sh` (simulate high heap by stressing the app):
```bash
#!/bin/bash
# Stress app to increase heap
for i in {1..100}; do
  curl -X POST http://<livebetting-external-ip>/api/v1/betslips/create \
    -H "X-Customer-Id: 123" \
    -H "Content-Type: application/json" \
    -d '{"eventId": "your-event-id", "betType": "HOME_WIN", "stake": 100, "multipleSlipCount": 2, "selectedRate": 1.5}'
done
```

Run: `./scripts/test-alert.sh`

Monitor:
- Prometheus: Check alert firing at http://<prometheus-ip>:9090/alerts.
- AlertManager: http://<alertmanager-ip>:9093 (see fired alerts).
- Email: Check your inbox for alert notification.
- Grafana: Dashboard shows spike; alert panel triggers.

### Troubleshooting
In `scripts/troubleshoot.sh`:
```bash
#!/bin/bash
kubectl get pods -o wide
kubectl logs -l app=livebetting  # App logs
kubectl logs -l app=prometheus  # Prometheus logs
kubectl describe svc prometheus-service  # Check endpoints
# Fix common: Scale down/up if heap high: kubectl scale deployment livebetting --replicas=0 && kubectl scale --replicas=1
```

Run: `./scripts/troubleshoot.sh`

Common Issues:
- Alert not firing: Check expr in rules; adjust threshold.
- Email not sent: Verify Gmail app password; check AlertManager logs.
- Pod crashes: Check MySQL connection; secrets.
- Access denied: Recreate secrets with correct base64.
