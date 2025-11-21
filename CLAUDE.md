# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Technology Stack

- **Kubernetes**: Deployment orchestration (Minikube for dev, AWS EKS for production)
- **Kustomize**: Configuration management and overlays
- **Helm**: Package management for monitoring stack (kube-prometheus-stack)
- **Language**: YAML/Kubernetes manifests (no application source code; all microservices are pre-built container images)

## Project Overview

This is a **microservices fleet management system** deployed on Kubernetes (both Minikube and AWS EKS). The architecture includes:

- **Position Simulator**: Generates GPS positions and publishes to a message queue
- **Position Tracker**: Consumes GPS messages, calculates metrics (speed, direction), stores in MongoDB
- **API Gateway**: REST endpoint for the frontend, routes requests to Position Tracker
- **ActiveMQ Queue**: Message broker for asynchronous communication between microservices
- **MongoDB**: Data persistence for vehicle tracking data
- **Web Application**: Angular frontend served via nginx
- **Logging Stack**: Fluent Bit (collection) → Elasticsearch (storage) → Kibana (visualization)
- **Monitoring Stack**: Prometheus (metrics collection) → Grafana (dashboards)

### Architecture Flow

```
Web Browser → Reverse Proxy (nginx) → API Gateway → Position Tracker → MongoDB
                                           ↑
                                           |
Position Simulator → ActiveMQ Queue ────────┘
```

**Namespaces**: The system uses multiple namespaces:
- `default`: Core application workloads (simulator, tracker, gateway, webapp, queue, MongoDB)
- `logging`: Fluent Bit (DaemonSet), Elasticsearch (StatefulSet), Kibana
- `monitoring`: Prometheus, Grafana (via kube-prometheus-stack Helm chart)

---

## Quick Reference: Essential Commands

```bash
# Local development (Minikube)
minikube start --driver=docker                      # Start Minikube
kubectl apply -k k8s/overlays/dev                   # Deploy app
minikube service fleetman-webapp                    # Open app in browser
kubectl logs -f deployment/position-tracker         # View logs
kubectl delete -k k8s/overlays/dev                  # Clean up

# AWS deployment
eksctl create cluster -f ./infra/cluster.yaml       # Create EKS cluster
kubectl apply -k ./k8s/overlays/aws                 # Deploy base app + logging
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  -f k8s/overlays/aws/monitoring/kube-prometheus-values.yaml \
  --namespace monitoring --create-namespace         # Deploy monitoring

# Verification
kubectl get pods -A                                 # Check all pods
kubectl get svc -A                                  # Check all services
kubectl get events --sort-by='.lastTimestamp'       # View recent events
```

---

## Directory Structure

```
kubernetes-aws-microservices/
├── infra/                      # AWS EKS cluster configuration (eksctl)
│   └── cluster.yaml            # EKS cluster definition (t3.medium nodes, add-ons)
├── k8s/                        # Kubernetes manifests (Kustomize-based)
│   ├── base/                   # Base resources (shared across environments)
│   │   ├── kustomization.yaml
│   │   ├── workloads.yaml      # Core deployments: queue, simulator, tracker, gateway, webapp
│   │   └── mongo.yaml          # MongoDB StatefulSet with PVC
│   └── overlays/               # Environment-specific configurations
│       ├── dev/                # Minikube development overlay
│       │   ├── kustomization.yaml
│       │   └── storage.yaml    # Local storage provisioner config
│       └── aws/                # AWS EKS overlay
│           ├── kustomization.yaml
│           ├── storage.yaml    # AWS EBS storage configuration (cloud-ssd)
│           ├── logging/        # Logging components (ELK stack)
│           │   ├── elasticsearch.yaml
│           │   ├── fluentbit.yaml
│           │   ├── fluentbit-prometheus.yaml  # ServiceMonitor for Prometheus scraping
│           │   └── kibana.yaml
│           └── monitoring/     # Monitoring components
│               └── kube-prometheus-values.yaml # Helm chart values (Prometheus + Grafana)
├── aws-test/                   # Quick AWS deployment test (simple nginx example)
├── README.md                   # Main documentation with architecture diagrams
├── README-minikube-deploy.md  # Step-by-step Minikube deployment guide
├── README-aws-install-config.md # AWS CLI & eksctl setup instructions
├── README-infra-aws.md        # AWS EKS deployment walkthrough
└── README-diagnostic.md       # Kubernetes debugging & troubleshooting guide
```

---

## Common Development Commands

### **Minikube (Local Development)**

```bash
# Start Minikube with Docker driver
minikube start --driver=docker

# Deploy to Minikube (dev overlay)
kubectl apply -k k8s/overlays/dev

# View running deployments
kubectl get deployments
kubectl get all

# Expose services locally
minikube service fleetman-queue     # Access ActiveMQ UI (port 8161)
minikube service fleetman-api-gateway   # Access API Gateway (port 8080)
minikube service fleetman-webapp    # Access webapp (port 80)

# Stream logs from a pod
kubectl logs -f deployment/position-tracker
kubectl logs -f deployment/position-simulator

# Check pod status
kubectl get pods
kubectl describe pod <pod-name>

# Delete deployment
kubectl delete -k k8s/overlays/dev
```

### **AWS EKS Deployment**

```bash
# Create EKS cluster
eksctl create cluster -f ./infra/cluster.yaml

# Verify nodes are ready
kubectl get nodes

# Deploy base application
kubectl apply -k ./k8s/overlays/aws

# Deploy logging stack
kubectl apply -k ./k8s/overlays/aws/logging

# Deploy monitoring stack (requires Helm)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  -f k8s/overlays/aws/monitoring/kube-prometheus-values.yaml \
  --namespace monitoring --create-namespace

# Verify deployments
kubectl get all
kubectl get all -n logging
kubectl get all -n monitoring

# Get service endpoints
kubectl get svc
kubectl get svc -n logging
kubectl get svc -n monitoring
```

### **Accessing Services**

| Service | Access Method | Default Credentials |
|---------|--------------|-------------------|
| **Minikube ActiveMQ UI** | `minikube service fleetman-queue` | admin/admin |
| **AWS Kibana** | LoadBalancer external IP (port 5601) | Default (no auth) |
| **AWS Grafana** | LoadBalancer external IP (port 3000) | admin/admin123 |
| **AWS Prometheus** | LoadBalancer external IP (port 9090) | None |

---

## Key Configuration Files

### **Base Workloads** (`k8s/base/workloads.yaml`)
- **queue (ActiveMQ)**: Message broker using `richardchesterwood/k8s-fleetman-queue:release2`
  - Port 8161 (UI), 61616 (endpoint)
- **position-simulator**: GPS data generator using `richardchesterwood/k8s-fleetman-position-simulator:release2`
  - Env: `SPRING_PROFILES_ACTIVE=production-microservice`
- **position-tracker**: Message consumer & metrics calculator using `richardchesterwood/k8s-fleetman-position-tracker:release3`
  - Port 8080, Env: `SPRING_PROFILES_ACTIVE=production-microservice`
- **api-gateway**: REST API gateway using `richardchesterwood/k8s-fleetman-api-gateway:release2-multi`
  - Port 8080, Env: `SPRING_PROFILES_ACTIVE=production-microservice`
- **webapp**: Angular frontend using `richardchesterwood/k8s-fleetman-webapp-angular:release2-multi`
  - Port 80, LoadBalancer service

### **Storage Configuration**
- **Minikube** (`k8s/overlays/dev/storage.yaml`): Uses local storage provisioner
- **AWS** (`k8s/overlays/aws/storage.yaml`): Uses EBS volumes (StorageClass: `cloud-ssd`)

### **Logging Stack** (`k8s/overlays/aws/logging/`)
- **Fluent Bit**: DaemonSet that reads `/var/log/containers/*.log` from all nodes
  - Enriches logs with Kubernetes metadata
  - Exposes `/api/v1/metrics` on port 2020 (for Prometheus scraping)
  - Outputs to Elasticsearch:9200
- **Elasticsearch**: StatefulSet for log storage & indexing
  - Port 9200 (HTTP API), 9300 (node communication)
  - Service: `elasticsearch-logging.logging.svc.cluster.local:9200`
- **Kibana**: Visualization UI connected to Elasticsearch
  - Port 5601, LoadBalancer service

### **Monitoring Stack** (`k8s/overlays/aws/monitoring/kube-prometheus-values.yaml`)
- **Prometheus**: Helm chart (kube-prometheus-stack)
  - Retention: 15 days
  - Storage: 10Gi on `cloud-ssd`
  - Port 9090, LoadBalancer service
  - Scrapes: Kubernetes components, Node exporter, Fluent Bit (via ServiceMonitor)
- **Grafana**: Integrated with Prometheus & Elasticsearch
  - Port 3000, LoadBalancer service
  - Includes Elasticsearch as secondary datasource
  - Credentials: admin/admin123

---

## Development Workflow

### **Testing Deployments**

1. **Test on Minikube first**: Use `k8s/overlays/dev` for local validation
2. **Deploy to AWS**: Use `k8s/overlays/aws` for production-like testing
3. **Verify components**: Check pod status, logs, service connectivity

### **Making Changes**

- **Adding/Modifying Workloads**: Edit `k8s/base/workloads.yaml` (affects all environments)
- **Environment-Specific Changes**: Use overlays in `k8s/overlays/{dev,aws}/`
- **Logging Configuration**: Modify `k8s/overlays/aws/logging/` files and reapply
- **Monitoring Configuration**: Update `kube-prometheus-values.yaml` and reinstall Helm chart

### **Debugging**

```bash
# View pod logs
kubectl logs -f deployment/<name>
kubectl logs -f pod/<pod-name>

# Check pod events
kubectl describe pod <pod-name>

# Check service connectivity
kubectl exec -it <pod> -- curl http://<service-name>:<port>

# Access Kubernetes dashboard
minikube dashboard

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

---

## Storage Architecture

### **MongoDB Persistence** (`k8s/base/mongo.yaml`)
- Uses PersistentVolumeClaim (PVC) for data persistence
- **Minikube**: Uses local provisioner
- **AWS**: Uses `gp3` EBS volumes via `cloud-ssd` StorageClass

### **Prometheus/Grafana Storage** (Helm-managed)
- **Grafana**: 5Gi on `cloud-ssd`
- **Prometheus**: 10Gi on `cloud-ssd` with 15-day retention

---

## Helm Chart Management

The monitoring stack is deployed via Helm. Key commands:

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  -f k8s/overlays/aws/monitoring/kube-prometheus-values.yaml \
  --namespace monitoring --create-namespace

# Upgrade existing installation
helm upgrade kube-prometheus prometheus-community/kube-prometheus-stack \
  -f k8s/overlays/aws/monitoring/kube-prometheus-values.yaml \
  --namespace monitoring

# Check Helm releases
helm list -A
helm get values kube-prometheus -n monitoring
```

---

## Kustomize Patterns Used

This project uses **Kustomize** for configuration management:

1. **Base Layer** (`k8s/base/`): Shared across all environments
   - Contains core microservices, MongoDB, services
   - Applied by all overlays via `../../base` reference

2. **Overlay Layers** (`k8s/overlays/{dev,aws}/`):
   - **dev**: Minikube-specific (local storage, node ports)
   - **aws**: EKS-specific (EBS volumes, LoadBalancers, logging, monitoring)

3. **Nested Overlays**: Logging and monitoring are additional layers within the AWS overlay

**Typical deployment patterns**:
```bash
kubectl apply -k k8s/overlays/dev          # Minikube: base + dev
kubectl apply -k k8s/overlays/aws          # AWS: base + aws (excludes logging/monitoring)
kubectl apply -k k8s/overlays/aws/logging  # AWS logging only (separate deployment)
helm install ... kube-prometheus-stack     # AWS monitoring (Helm-based, not Kustomize)
```

---

## Service Communication

### **Internal Service DNS**
- **Minikube**: Services accessible via `<service-name>` within cluster
- **AWS**: Services accessible via `<service-name>.<namespace>.svc.cluster.local`

### **Key Service Endpoints**
- `fleetman-queue:8161` - ActiveMQ UI
- `fleetman-queue:61616` - ActiveMQ endpoint (for microservices)
- `fleetman-api-gateway:8080` - API Gateway
- `fleetman-position-tracker:8080` - Position Tracker
- `mongodb:27017` - MongoDB
- `elasticsearch-logging.logging.svc.cluster.local:9200` - Elasticsearch (only accessible from logging namespace)

---

## Useful Documentation References

- **Main README**: High-level architecture and component descriptions
- **Minikube Deployment**: Step-by-step testing guide (`README-minikube-deploy.md`)
- **AWS Setup**: Prerequisites and cluster creation (`README-aws-install-config.md`)
- **AWS Deployment**: Full deployment walkthrough with logging/monitoring (`README-infra-aws.md`)
- **Diagnostics**: Debugging and troubleshooting guide (`README-diagnostic.md`)

---

## Verifying Deployments Are Working

```bash
# Check all pods are running (should be in Running state)
kubectl get pods -A

# View a pod's logs for errors
kubectl logs -f pod/<pod-name> -n <namespace>

# Check if services have endpoints (should not be empty)
kubectl get endpoints -A

# For Minikube: verify all services are accessible
minikube service fleetman-queue          # ActiveMQ should open in browser
minikube service fleetman-webapp         # Web app should be accessible

# For AWS: get LoadBalancer external IPs
kubectl get svc -n logging               # Get Kibana IP
kubectl get svc -n monitoring            # Get Grafana & Prometheus IPs

# Test service connectivity from within cluster
kubectl exec -it <pod-name> -- curl http://<service-name>:8080

# Verify MongoDB is persisting data
kubectl exec -it mongodb-0 -- mongosh mongodb://localhost:27017
# Inside mongo shell: show dbs; use fleet; db.collection.findOne();
```

### Common Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| Pods stuck in `Pending` | Storage not available | Check PVC status: `kubectl get pvc -A` |
| Pods in `CrashLoopBackOff` | Container failed to start | Check logs: `kubectl logs <pod> --previous` |
| Services have no endpoints | Pods not running | Verify pods are `Running` and match label selectors |
| Cannot access Minikube services | Service not exposed | Use `minikube service <name>` to access |
| ActiveMQ/Elasticsearch slow | High memory usage | Increase resource limits in YAML files |

---

## Modifying Configurations

### **Updating a Microservice**

1. **Edit base configuration**: Modify `k8s/base/workloads.yaml` (affects both dev and prod)
2. **Apply changes**: `kubectl apply -k k8s/overlays/dev` (or `k8s/overlays/aws`)
3. **Watch rollout**: `kubectl rollout status deployment/<name>`
4. **View new pod logs**: `kubectl logs -f deployment/<name>`

### **Adding a New Microservice**

1. Add deployment spec to `k8s/base/workloads.yaml`
2. Add service definition to the same file
3. Ensure it references `fleetman-queue` if it needs messaging
4. Apply: `kubectl apply -k k8s/overlays/dev`
5. Verify: `kubectl get pods -w` and `kubectl logs -f deployment/<name>`

### **Environment-Specific Overrides**

Create patches in `k8s/overlays/{dev,aws}/` to override base configs:
- Minikube: Use NodePort instead of LoadBalancer
- AWS: Use specific storage classes, increase resource limits, add monitoring labels

---

## Important Notes

1. **Image Sources**: All microservice images are from `richardchesterwood/` registry (pre-built, not source code in this repo)
2. **Spring Boot Profiles**: Services use `SPRING_PROFILES_ACTIVE=production-microservice` environment variable for configuration
3. **Multi-Namespace Design**: Logging and monitoring use separate namespaces for isolation and reusability
4. **LoadBalancer vs NodePort**:
   - Minikube uses NodePort for exposing services
   - AWS EKS uses LoadBalancer for external access
5. **Kustomize vs Helm**: This project mixes both—Kustomize for manifests, Helm for the monitoring stack
6. **No Application Source Code**: This repo contains only Kubernetes configuration. All microservices are pre-built images from Docker registry.
7. **Deployment Order Matters**: MongoDB should start before Position Tracker; ActiveMQ should be ready before Simulator
