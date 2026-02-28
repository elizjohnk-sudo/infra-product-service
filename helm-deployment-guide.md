# Helm Deployment Guide for Microservices Platform

## Table of Contents
1. [Overview](#overview)
2. [Helm Chart Structure](#helm-chart-structure)
3. [Chart Details](#chart-details)
4. [Configuration Management](#configuration-management)
5. [Deployment Instructions](#deployment-instructions)
6. [ArgoCD Integration](#argocd-integration)
7. [Troubleshooting](#troubleshooting)

---

## Overview

This guide explains the Helm charts for deploying the microservices platform consisting of:
- **Namespace Chart**: Creates the Kubernetes namespace
- **PostgreSQL Chart**: Deploys PostgreSQL database with persistent storage
- **Microservices Chart**: Deploys 4 microservices (Inventory, Product, Order, API Gateway)

### Benefits of Helm Charts

- **Templating**: Reusable patterns across services
- **Configuration Management**: Easy environment-specific overrides
- **Version Control**: Track chart versions independently
- **Rollback Support**: Easy rollback to previous versions
- **Package Management**: Bundle related resources together

---

## Helm Chart Structure

```
helm/
├── namespace/
│   ├── Chart.yaml              # Chart metadata
│   ├── values.yaml             # Default values
│   └── templates/
│       └── namespace.yaml      # Namespace template
│
├── postgres/
│   ├── Chart.yaml              # Chart metadata
│   ├── values.yaml             # Default values (dev)
│   ├── values-dev.yaml         # Development overrides
│   ├── values-prod.yaml        # Production overrides
│   └── templates/
│       ├── configmap.yaml      # Database configuration
│       ├── secret.yaml         # Database credentials
│       ├── pvc.yaml            # Persistent storage
│       ├── deployment.yaml     # PostgreSQL deployment
│       └── service.yaml        # PostgreSQL service
│
└── microservices/
    ├── Chart.yaml              # Chart metadata
    ├── values.yaml             # Default values (dev)
    ├── values-dev.yaml         # Development overrides
    ├── values-prod.yaml        # Production overrides
    └── templates/
        ├── configmap.yaml      # Service configurations
        ├── deployment.yaml     # Service deployments
        └── service.yaml        # Service exposures
```

---

## Chart Details

### 1. Namespace Chart

**Purpose**: Creates the Kubernetes namespace for all resources.

**Chart.yaml**:
```yaml
apiVersion: v2
name: microservices-namespace
description: Namespace for microservices platform
version: 1.0.0
```

**values.yaml**:
```yaml
namespace:
  name: product-services-dev
```

**Key Features**:
- Simple, single-resource chart
- Namespace name configurable via values
- Must be deployed first before other charts

---

### 2. PostgreSQL Chart

**Purpose**: Deploys PostgreSQL database with persistent storage for the Inventory service.

#### Files and Their Purpose

**Chart.yaml**:
- Defines chart metadata
- Version: 1.0.0
- AppVersion: 15 (PostgreSQL version)

**values.yaml** (Default Configuration):
```yaml
namespace: product-services-dev

postgres:
  image: postgres:15-alpine
  replicas: 1

  config:
    database: inventorydb
    user: postgres

  secret:
    password: postgres  # Plain text for learning

  storage:
    size: 1Gi
    storageClass: standard

  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

  service:
    name: postgres-service
    type: ClusterIP
    port: 5432
```

#### Templates

**configmap.yaml**:
- Creates ConfigMap with non-sensitive database configuration
- Stores: `POSTGRES_DB`, `POSTGRES_USER`
- Used by: PostgreSQL container, Inventory service

**secret.yaml**:
- Creates Secret with database password
- Stores: `POSTGRES_PASSWORD`
- Format: Plain text (stringData) for simplicity
- Used by: PostgreSQL container, Inventory service

**pvc.yaml**:
- Creates PersistentVolumeClaim for data storage
- Size: Configurable (default 1Gi)
- AccessMode: ReadWriteOnce
- Data persists across pod restarts

**deployment.yaml**:
- Deploys PostgreSQL container
- Mounts PVC at `/var/lib/postgresql/data`
- Uses ConfigMap and Secret for configuration
- Includes liveness and readiness probes
- Resource limits applied

**service.yaml**:
- Exposes PostgreSQL on port 5432
- Type: ClusterIP (internal only)
- DNS name: `postgres-service` (used by Inventory service)

#### Environment-Specific Values

**values-dev.yaml**:
- 1 replica (single instance)
- Lower resource limits for local development
- Slower storage class acceptable

**values-prod.yaml**:
- 2 replicas (high availability)
- 10Gi storage
- Fast SSD storage class
- Higher resource limits (2Gi RAM, 1 CPU)

---

### 3. Microservices Chart

**Purpose**: Deploys all 4 microservices using a common template pattern.

#### Architecture

The chart uses **dynamic templating** to generate resources for multiple services from a single template:

```
values.yaml (service definitions)
    ↓
templates/*.yaml (loop through services)
    ↓
Generated Kubernetes manifests
```

#### values.yaml Structure

**Global Configuration**:
```yaml
global:
  image:
    pullPolicy: Always
    registry: docker.io
    username: yourdockerhubusername  # CHANGE THIS!

  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

**Service Definitions**:

Each service follows this pattern:
```yaml
services:
  serviceName:
    enabled: true              # Enable/disable service
    name: service-name         # Kubernetes resource name
    replicas: 1                # Number of pods
    image:
      repository: image-name   # Docker image name
      tag: latest              # Image tag
    port: 8080                 # Container port
    configMap:                 # Service-specific configuration
      KEY: value
    secretRef:                 # Optional: Reference to secret
      name: secret-name
    service:
      type: ClusterIP          # Service type
      nodePort: 30080          # Only for NodePort type
```

#### Service Configurations

**1. Inventory Service** (Port 8083):
```yaml
inventory:
  enabled: true
  name: inventory-service
  port: 8083

  configMap:
    DB_HOST: postgres-service
    DB_NAME: inventorydb
    DB_USER: postgres
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-service:5432/inventorydb
    SPRING_JPA_HIBERNATE_DDL_AUTO: update
    SPRING_JPA_SHOW_SQL: "true"
    SPRING_JPA_PROPERTIES_HIBERNATE_DIALECT: org.hibernate.dialect.PostgreSQLDialect

  secretRef:
    name: postgres-secret  # References DB password

  service:
    type: ClusterIP
```

**Why these configs?**
- Connects to PostgreSQL for data persistence
- JPA auto-generates database schema (`ddl-auto: update`)
- Shows SQL queries in logs (`show-sql: true`) for debugging

**2. Product Service** (Port 8082):
```yaml
product:
  enabled: true
  name: product-service
  port: 8082

  configMap:
    INVENTORY_SERVICE_URL: http://inventory-service:8083

  service:
    type: ClusterIP
```

**Why these configs?**
- Calls Inventory Service to check stock availability
- Uses Kubernetes DNS (`inventory-service:8083`)

**3. Order Service** (Port 8081):
```yaml
order:
  enabled: true
  name: order-service
  port: 8081

  configMap:
    PRODUCT_SERVICE_URL: http://product-service:8082

  service:
    type: ClusterIP
```

**Why these configs?**
- Calls Product Service to validate products
- Uses Kubernetes DNS (`product-service:8082`)

**4. API Gateway** (Port 8080):
```yaml
apiGateway:
  enabled: true
  name: api-gateway
  port: 8080

  configMap: {}  # No configuration needed

  service:
    type: NodePort
    nodePort: 30080  # External access
```

**Why these configs?**
- Entry point for all external requests
- NodePort allows access from outside cluster
- Routes requests to backend services

#### Templates

**configmap.yaml**:
```yaml
{{- range $serviceName, $service := .Values.services }}
{{- if and $service.enabled $service.configMap }}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ $service.name }}-config
  ...
data:
  {{- range $key, $value := $service.configMap }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
{{- end }}
```

**How it works**:
1. Loops through all services in `values.yaml`
2. For each enabled service with configMap defined
3. Creates a ConfigMap named `{service-name}-config`
4. Adds all key-value pairs from service's configMap

**Example output**:
- `inventory-service-config` (DB connection details)
- `product-service-config` (Inventory service URL)
- `order-service-config` (Product service URL)

**deployment.yaml**:
```yaml
{{- range $serviceName, $service := .Values.services }}
{{- if $service.enabled }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $service.name }}
  ...
spec:
  replicas: {{ $service.replicas }}
  template:
    spec:
      containers:
      - name: {{ $service.name }}
        image: {{ $.Values.global.image.registry }}/{{ $.Values.global.image.username }}/{{ $service.image.repository }}:{{ $service.image.tag }}
        envFrom:
        - configMapRef:
            name: {{ $service.name }}-config
        - secretRef:
            name: {{ $service.secretRef.name }}
        ...
{{- end }}
{{- end }}
```

**Key features**:
- Dynamic image path: `docker.io/username/service:tag`
- Injects ConfigMap as environment variables
- Injects Secret as environment variables (if specified)
- Health probes for liveness and readiness
- Resource limits from global configuration

**service.yaml**:
```yaml
{{- range $serviceName, $service := .Values.services }}
{{- if $service.enabled }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ $service.name }}
  ...
spec:
  type: {{ $service.service.type }}
  ports:
  - port: {{ $service.port }}
    targetPort: {{ $service.port }}
    {{- if $service.service.nodePort }}
    nodePort: {{ $service.service.nodePort }}
    {{- end }}
{{- end }}
{{- end }}
```

**Creates**:
- ClusterIP services for internal services
- NodePort service for API Gateway (external access)

---

## Configuration Management

### Environment-Specific Overrides

Helm uses a layering approach:
```
values.yaml (base)
    ↓
values-dev.yaml (overrides for dev)
    ↓
values-prod.yaml (overrides for prod)
```

### Development Environment (values-dev.yaml)

```yaml
namespace: product-services-dev

global:
  image:
    pullPolicy: Always     # Always pull latest
    tag: latest            # Use latest tag

services:
  inventory:
    replicas: 1            # Single instance
    configMap:
      SPRING_JPA_SHOW_SQL: "true"  # Debug mode
```

**Characteristics**:
- Single replicas for all services
- Latest image tags (rapid iteration)
- SQL logging enabled
- Lower resource limits

### Production Environment (values-prod.yaml)

```yaml
namespace: product-services-prod

global:
  image:
    pullPolicy: IfNotPresent  # Use cached images
    tag: v1.0.0               # Specific version

  resources:
    requests:
      memory: "512Mi"         # Higher resources
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"

services:
  inventory:
    replicas: 3               # High availability
    configMap:
      SPRING_JPA_SHOW_SQL: "false"           # No debug logs
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate # Don't modify schema
```

**Characteristics**:
- Multiple replicas (high availability)
- Specific version tags (stability)
- SQL logging disabled (performance)
- Schema validation only (safety)
- Higher resource limits
- LoadBalancer for API Gateway

---

## Deployment Instructions

### Prerequisites

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version

# Ensure kubectl is configured
kubectl config current-context
```

### Local Testing (Without Installing)

**Test template rendering**:
```bash
# Navigate to infra-product-service directory
cd infra-product-service

# Test namespace chart
helm template namespace helm/namespace

# Test postgres chart
helm template postgres helm/postgres

# Test microservices chart
helm template microservices helm/microservices

# Test with dev values
helm template microservices helm/microservices -f helm/microservices/values-dev.yaml

# Test with prod values
helm template microservices helm/microservices -f helm/microservices/values-prod.yaml
```

**Validate against Kubernetes API**:
```bash
# Dry-run with validation
helm install namespace helm/namespace --dry-run --debug
helm install postgres helm/postgres --dry-run --debug
helm install microservices helm/microservices --dry-run --debug
```

**Lint charts**:
```bash
# Check for issues
helm lint helm/namespace
helm lint helm/postgres
helm lint helm/microservices
```

### Manual Deployment with Helm

**Step 1: Deploy Namespace**
```bash
helm install microservices-namespace helm/namespace \
  --create-namespace

# Verify
kubectl get namespace product-services-dev
```

**Step 2: Deploy PostgreSQL**
```bash
helm install postgres helm/postgres \
  -n product-services-dev \
  -f helm/postgres/values-dev.yaml

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n product-services-dev --timeout=300s

# Verify
kubectl get all -n product-services-dev
```

**Step 3: Update Microservices values.yaml**

**IMPORTANT**: Before deploying, update Docker Hub username:
```yaml
# helm/microservices/values.yaml
global:
  image:
    username: yourdockerhubusername  # ← CHANGE THIS to your Docker Hub username!
```

**Step 4: Deploy Microservices**
```bash
helm install microservices helm/microservices \
  -n product-services-dev \
  -f helm/microservices/values-dev.yaml

# Monitor deployment
kubectl get pods -n product-services-dev -w

# Verify all services
kubectl get all -n product-services-dev
```

**Step 5: Test the Application**
```bash
# Get API Gateway URL
kubectl get svc api-gateway -n product-services-dev

# Test health endpoints
curl http://localhost:30080/actuator/health

# Test full flow
# 1. Create inventory
curl -X POST http://localhost:30080/api/inventory \
  -H "Content-Type: application/json" \
  -d '{"productId":"PROD-001","quantity":100}'

# 2. Create product
curl -X POST http://localhost:30080/api/products \
  -H "Content-Type: application/json" \
  -d '{"productId":"PROD-001","name":"Laptop","price":999.99,"category":"Electronics"}'

# 3. Create order
curl -X POST http://localhost:30080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId":"CUST-001","productId":"PROD-001","quantity":2}'
```

### Upgrade Deployments

```bash
# Upgrade with new values
helm upgrade postgres helm/postgres \
  -n product-services-dev \
  -f helm/postgres/values-dev.yaml

helm upgrade microservices helm/microservices \
  -n product-services-dev \
  -f helm/microservices/values-dev.yaml

# Check upgrade history
helm history postgres -n product-services-dev
helm history microservices -n product-services-dev
```

### Rollback

```bash
# Rollback to previous version
helm rollback postgres -n product-services-dev
helm rollback microservices -n product-services-dev

# Rollback to specific revision
helm rollback microservices 2 -n product-services-dev
```

### Uninstall

```bash
# Uninstall releases
helm uninstall microservices -n product-services-dev
helm uninstall postgres -n product-services-dev
helm uninstall microservices-namespace

# Verify cleanup
kubectl get all -n product-services-dev
```

---

## ArgoCD Integration

### Update ArgoCD Applications for Helm

**argocd-apps-helm.yaml**:
```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: microservices-namespace
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/infra-product-service.git
    targetRevision: main
    path: helm/namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    syncOptions:
    - CreateNamespace=true

---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: postgres
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/infra-product-service.git
    targetRevision: main
    path: helm/postgres
    helm:
      valueFiles:
      - values.yaml
      - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: product-services-dev
  syncPolicy:
    syncOptions:
    - CreateNamespace=true

---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: microservices
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/infra-product-service.git
    targetRevision: main
    path: helm/microservices
    helm:
      valueFiles:
      - values.yaml
      - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: product-services-dev
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
```

### Deploy with ArgoCD

```bash
# Push Helm charts to Git
git add helm/
git commit -m "Add Helm charts for microservices platform"
git push

# Update ArgoCD applications
kubectl apply -f argocd-apps-helm.yaml

# Sync applications
argocd app sync microservices-namespace
argocd app sync postgres
argocd app sync microservices

# Monitor
argocd app list
argocd app get microservices
```

### Override Values in ArgoCD

**Option 1: Via ArgoCD Application Spec**
```yaml
spec:
  source:
    helm:
      valueFiles:
      - values.yaml
      - values-dev.yaml
      parameters:
      - name: services.inventory.replicas
        value: "2"
```

**Option 2: Via ArgoCD CLI**
```bash
argocd app set microservices \
  --helm-set services.inventory.replicas=2
```

---

## Troubleshooting

### Common Issues

**Issue 1: Helm template fails**
```bash
# Error: template rendering failed
helm template microservices helm/microservices --debug

# Check for:
# - YAML indentation errors
# - Missing required values
# - Template syntax errors
```

**Issue 2: Image pull errors**
```bash
# Check if image exists
docker pull yourusername/inventory-service:latest

# Verify values.yaml has correct Docker Hub username
grep username helm/microservices/values.yaml

# Check pod events
kubectl describe pod inventory-service-xxx -n product-services-dev
```

**Issue 3: ConfigMap not found**
```bash
# Check if ConfigMap exists
kubectl get configmap -n product-services-dev

# Check if service has configMap defined
helm template microservices helm/microservices | grep ConfigMap

# Verify template loop is working
helm template microservices helm/microservices --debug | grep "inventory-service-config"
```

**Issue 4: Service can't connect to database**
```bash
# Check postgres service DNS
kubectl exec -it deployment/inventory-service -n product-services-dev -- nslookup postgres-service

# Check environment variables in pod
kubectl exec deployment/inventory-service -n product-services-dev -- env | grep DB

# Check postgres logs
kubectl logs deployment/postgres -n product-services-dev

# Verify postgres secret exists
kubectl get secret postgres-secret -n product-services-dev
```

**Issue 5: ArgoCD shows OutOfSync**
```bash
# Check diff
argocd app diff microservices

# Common causes:
# - values.yaml changed but not committed
# - Manual kubectl changes
# - Image tag mismatch

# Force sync
argocd app sync microservices --force
```

### Validation Commands

```bash
# Check all resources
kubectl get all -n product-services-dev

# Check configmaps
kubectl get configmaps -n product-services-dev

# Check secrets
kubectl get secrets -n product-services-dev

# Check helm releases
helm list -n product-services-dev

# Check specific deployment
kubectl describe deployment inventory-service -n product-services-dev

# Check pod logs
kubectl logs -f deployment/inventory-service -n product-services-dev

# Check events
kubectl get events -n product-services-dev --sort-by='.lastTimestamp'
```

---

## Best Practices

### 1. Version Management
```yaml
# Always specify image versions in production
global:
  image:
    tag: v1.0.0  # Not 'latest'
```

### 2. Resource Limits
```yaml
# Always set resource limits
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 3. Health Probes
```yaml
# Include liveness and readiness probes
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8083
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8083
```

### 4. Labels
```yaml
# Use consistent labels
labels:
  app: {{ $service.name }}
  chart: {{ $.Chart.Name }}-{{ $.Chart.Version }}
  service: {{ $serviceName }}
  version: {{ $service.image.tag }}
```

### 5. ConfigMap vs Secret
- **ConfigMap**: Non-sensitive configuration (URLs, database names)
- **Secret**: Sensitive data (passwords, tokens)

### 6. Testing
```bash
# Always test before deploying
helm template microservices helm/microservices --debug
helm lint helm/microservices
kubectl apply --dry-run=client -f <(helm template microservices helm/microservices)
```

---

## Next Steps

1. **Add Monitoring**: Integrate Prometheus and Grafana
2. **Implement Secrets Management**: Use Sealed Secrets or External Secrets Operator
3. **Add Helm Tests**: Create test pods to validate deployments
4. **Implement Rolling Updates**: Configure update strategies
5. **Add Ingress**: Replace NodePort with Ingress for API Gateway
6. **Implement Autoscaling**: Add HorizontalPodAutoscaler resources

---

## Additional Resources

- **Helm Documentation**: https://helm.sh/docs/
- **Helm Best Practices**: https://helm.sh/docs/chart_best_practices/
- **ArgoCD Helm Support**: https://argo-cd.readthedocs.io/en/stable/user-guide/helm/
- **Kubernetes Configuration Best Practices**: https://kubernetes.io/docs/concepts/configuration/overview/

---

**Questions or Issues?** Check the troubleshooting section or review the chart templates for understanding how values are used.
