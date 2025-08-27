# Containerization Docker Fundamentals + Kubernetes + Docker Swarm + Orchestration Patterns + Infrastructure as Code + Monitoring

## Professional PowerPoint Presentation

---

## Slide 1: Title Slide

### Containerization & Orchestration
#### Docker, Kubernetes, Infrastructure as Code & Monitoring

**Modern Container-Based Application Deployment**

*Professional Training Series*

---

## Slide 2: Containerization Fundamentals

### Understanding Container Technology

**Container Concepts:**
- **Containerization:** Packaging applications with dependencies
- **Isolation:** Process and resource separation
- **Portability:** Consistent runtime across environments
- **Efficiency:** Lightweight compared to virtual machines

**Container vs Virtual Machine:**

| Aspect | Containers | Virtual Machines |
|--------|------------|------------------|
| Resource Usage | Lightweight | Heavy |
| Startup Time | Seconds | Minutes |
| Isolation Level | Process-level | Hardware-level |
| Portability | High | Medium |
| Density | High | Low |

**Benefits:**
- Consistent development and production environments
- Simplified deployment and scaling
- Improved resource utilization
- Enhanced application portability
- Streamlined CI/CD pipelines

---

## Slide 3: Docker Fundamentals

### Container Platform and Ecosystem

**Docker Architecture:**
- **Docker Engine:** Container runtime and management
- **Docker Images:** Read-only templates for containers
- **Docker Containers:** Running instances of images
- **Docker Registry:** Image storage and distribution
- **Dockerfile:** Instructions for building images

**Basic Docker Commands:**
```bash
# Build image from Dockerfile
docker build -t myapp:latest .

# Run container
docker run -d -p 8000:8000 --name myapp-container myapp:latest

# List running containers
docker ps

# View container logs
docker logs myapp-container

# Execute commands in container
docker exec -it myapp-container /bin/bash

# Stop and remove container
docker stop myapp-container
docker rm myapp-container
```

**Dockerfile Example:**
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Slide 4: Docker Best Practices

### Production-Ready Container Images

**Image Optimization:**
```dockerfile
# Multi-stage build for smaller images
FROM python:3.9-slim as builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.9-slim

# Create non-root user
RUN useradd --create-home --shell /bin/bash app

# Copy dependencies from builder stage
COPY --from=builder /root/.local /home/app/.local

WORKDIR /app
COPY --chown=app:app . .

USER app

ENV PATH=/home/app/.local/bin:$PATH

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Security Best Practices:**
- Use official base images
- Run containers as non-root users
- Minimize image layers and size
- Scan images for vulnerabilities
- Use specific image tags, avoid 'latest'
- Implement proper secret management

**Performance Optimization:**
- Leverage Docker layer caching
- Use .dockerignore to exclude unnecessary files
- Optimize dependency installation order
- Implement health checks

---

## Slide 5: Container Orchestration Overview

### Managing Containerized Applications at Scale

**Orchestration Challenges:**
- Container lifecycle management
- Service discovery and load balancing
- Scaling and resource allocation
- Health monitoring and self-healing
- Configuration and secret management
- Network and storage management

**Orchestration Solutions:**

| Platform | Complexity | Use Case | Ecosystem |
|----------|------------|----------|-----------|
| Docker Compose | Low | Development, small deployments | Docker |
| Docker Swarm | Medium | Simple production clusters | Docker |
| Kubernetes | High | Enterprise, complex applications | CNCF |
| Amazon ECS | Medium | AWS-native applications | AWS |
| Nomad | Medium | Multi-cloud, flexible workloads | HashiCorp |

**Key Features:**
- Automated deployment and scaling
- Service mesh and networking
- Storage orchestration
- Configuration management
- Monitoring and logging integration

---

## Slide 6: Docker Swarm Implementation

### Native Docker Clustering Solution

**Docker Swarm Architecture:**
- **Manager Nodes:** Cluster management and orchestration
- **Worker Nodes:** Container execution
- **Services:** Declarative application definitions
- **Tasks:** Individual container instances
- **Overlay Networks:** Multi-host networking

**Swarm Setup:**
```bash
# Initialize swarm on manager node
docker swarm init --advertise-addr <manager-ip>

# Join worker nodes
docker swarm join --token <worker-token> <manager-ip>:2377

# Deploy service
docker service create \
  --name web-app \
  --replicas 3 \
  --publish 8000:8000 \
  myapp:latest

# Scale service
docker service scale web-app=5

# Update service
docker service update --image myapp:v2 web-app
```

**Docker Compose for Swarm:**
```yaml
version: '3.8'
services:
  web:
    image: myapp:latest
    ports:
      - "8000:8000"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    networks:
      - app-network

  redis:
    image: redis:alpine
    deploy:
      replicas: 1
    networks:
      - app-network

networks:
  app-network:
    driver: overlay
```

---

## Slide 7: Kubernetes Architecture

### Enterprise Container Orchestration Platform

**Kubernetes Components:**

**Control Plane:**
- **API Server:** Central management interface
- **etcd:** Distributed configuration store
- **Scheduler:** Pod placement decisions
- **Controller Manager:** Maintains desired state

**Node Components:**
- **kubelet:** Node agent managing pods
- **kube-proxy:** Network proxy and load balancer
- **Container Runtime:** Docker, containerd, CRI-O

**Kubernetes Objects:**
```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: myapp:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

---

## Slide 8: Kubernetes Advanced Features

### Production Kubernetes Patterns

**ConfigMaps and Secrets:**
```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgresql://db:5432/myapp"
  debug_mode: "false"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  db_password: <base64-encoded-password>
  api_key: <base64-encoded-key>

---
# Using in Deployment
spec:
  containers:
  - name: web
    image: myapp:latest
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db_password
```

**Persistent Volumes:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd

---
# Using in Deployment
spec:
  containers:
  - name: web
    volumeMounts:
    - name: app-data
      mountPath: /app/data
  volumes:
  - name: app-data
    persistentVolumeClaim:
      claimName: app-storage
```

---

## Slide 9: Infrastructure as Code

### Automated Infrastructure Management

**IaC Benefits:**
- Version-controlled infrastructure
- Reproducible environments
- Reduced manual errors
- Faster deployment cycles
- Cost optimization through automation

**Terraform Example:**
```hcl
# Provider configuration
provider "aws" {
  region = "us-west-2"
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.cluster.arn
  version  = "1.24"

  vpc_config {
    subnet_ids = [
      aws_subnet.private_1.id,
      aws_subnet.private_2.id,
      aws_subnet.public_1.id,
      aws_subnet.public_2.id,
    ]
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
  ]
}

# Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main-nodes"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = [aws_subnet.private_1.id, aws_subnet.private_2.id]

  scaling_config {
    desired_size = 2
    max_size     = 4
    min_size     = 1
  }

  instance_types = ["t3.medium"]

  depends_on = [
    aws_iam_role_policy_attachment.node_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.node_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.node_AmazonEC2ContainerRegistryReadOnly,
  ]
}
```

**Helm Charts:**
```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: My Application Helm Chart
version: 0.1.0
appVersion: "1.0"

# values.yaml
replicaCount: 3

image:
  repository: myapp
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

---

## Slide 10: Container Monitoring and Observability

### Production Monitoring Stack

**Monitoring Architecture:**
- **Metrics Collection:** Prometheus, Grafana
- **Log Aggregation:** ELK Stack, Fluentd
- **Distributed Tracing:** Jaeger, Zipkin
- **Application Performance:** New Relic, Datadog

**Prometheus Configuration:**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)

  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
    - role: node
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
```

**Application Metrics:**
```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Metrics definitions
REQUEST_COUNT = Counter('app_requests_total', 'Total requests', ['method', 'endpoint'])
REQUEST_LATENCY = Histogram('app_request_duration_seconds', 'Request latency')
ACTIVE_CONNECTIONS = Gauge('app_active_connections', 'Active connections')

# Middleware for FastAPI
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start_time = time.time()
    
    response = await call_next(request)
    
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path
    ).inc()
    
    REQUEST_LATENCY.observe(time.time() - start_time)
    
    return response

# Start metrics server
start_http_server(8001)
```

---

## Slide 11: CI/CD Pipeline Integration

### Automated Container Deployment

**GitLab CI/CD Pipeline:**
```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_REGISTRY: registry.gitlab.com
  IMAGE_NAME: $DOCKER_REGISTRY/$CI_PROJECT_PATH
  KUBECONFIG: /tmp/kubeconfig

build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
  only:
    - main

test:
  stage: test
  script:
    - docker run --rm $IMAGE_NAME:$CI_COMMIT_SHA pytest tests/
  only:
    - main

deploy:
  stage: deploy
  script:
    - echo $KUBECONFIG_CONTENT | base64 -d > $KUBECONFIG
    - helm upgrade --install myapp ./helm-chart 
        --set image.tag=$CI_COMMIT_SHA
        --namespace production
  only:
    - main
  environment:
    name: production
    url: https://myapp.example.com
```

**GitHub Actions Workflow:**
```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
    
    - name: Push to registry
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push myapp:${{ github.sha }}
    
    - name: Deploy to Kubernetes
      uses: azure/k8s-deploy@v1
      with:
        manifests: |
          k8s/deployment.yaml
          k8s/service.yaml
        images: |
          myapp:${{ github.sha }}
        kubeconfig: ${{ secrets.KUBECONFIG }}
```

---

## Slide 12: Security and Best Practices

### Container Security Implementation

**Security Strategies:**

**Image Security:**
```dockerfile
# Use specific, minimal base images
FROM python:3.9-slim@sha256:specific-hash

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install security updates
RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*

# Set proper permissions
COPY --chown=appuser:appuser . /app
USER appuser

# Use read-only filesystem
VOLUME ["/tmp"]
```

**Kubernetes Security Policies:**
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

**Network Policies:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-netpol
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

---

## Slide 13: Summary and Production Readiness

### Container Orchestration Mastery

**Implementation Checklist:**

**Development Phase:**
- Containerize applications with optimized Dockerfiles
- Implement proper logging and metrics collection
- Create comprehensive test suites
- Set up local development environments

**Deployment Phase:**
- Choose appropriate orchestration platform
- Implement Infrastructure as Code
- Set up CI/CD pipelines with automated testing
- Configure monitoring and alerting systems

**Production Phase:**
- Implement security policies and scanning
- Set up backup and disaster recovery
- Monitor performance and optimize resources
- Plan for scaling and capacity management

**Best Practices Summary:**
- Use multi-stage builds for smaller images
- Implement proper health checks and readiness probes
- Use secrets management for sensitive data
- Implement network segmentation and policies
- Regular security scanning and updates
- Comprehensive monitoring and observability
- Automated backup and recovery procedures

**Future Considerations:**
- Service mesh adoption (Istio, Linkerd)
- Serverless containers (Knative, AWS Fargate)
- GitOps deployment patterns (ArgoCD, Flux)
- Multi-cloud and hybrid deployments
- Edge computing and container optimization

---

## Presentation Notes

**Target Audience:** DevOps engineers, platform engineers, backend developers
**Duration:** 65-80 minutes
**Prerequisites:** Basic understanding of Linux, networking, and application deployment
**Learning Objectives:**
- Master container technologies and Docker
- Implement container orchestration solutions
- Design Infrastructure as Code workflows
- Build production-ready monitoring systems