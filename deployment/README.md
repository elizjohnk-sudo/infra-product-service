# 📦 Deployment Directory

This directory contains all deployment-related configurations, **separate from application code**.

## 🏗️ Structure

```
deployment/
└── kubernetes/
    ├── namespace/
    │   └── namespace.yaml              # Namespace definition
    ├── postgres/
    │   ├── postgres-configmap.yaml     # PostgreSQL configuration
    │   ├── postgres-secret.yaml        # PostgreSQL credentials
    │   ├── postgres-pvc.yaml           # Persistent storage claim
    │   ├── postgres-deployment.yaml    # PostgreSQL deployment
    │   └── postgres-service.yaml       # PostgreSQL service (DNS)
    └── services/
        ├── inventory-service.yaml      # Inventory service deployment + service
        ├── product-service.yaml        # Product service deployment + service
        ├── order-service.yaml          # Order service deployment + service
        └── api-gateway.yaml            # API Gateway deployment + service (NodePort)
```

---

## 🎯 Why Separate Deployment Code?

### **Production Best Practice:**

In real-world scenarios:

| Aspect | Application Code | Deployment Code |
|--------|-----------------|-----------------|
| **Repository** | Separate Git repo | Separate Git repo |
| **Ownership** | Development team | DevOps/Platform team |
| **Changes** | Business logic, features | Infrastructure, scaling, configuration |
| **CI/CD** | Build → Test → Push image | Deploy → Monitor → Rollback |
| **Version** | Semantic versioning (1.2.3) | Git commit hash or date-based |

### **Example: Real Companies**

**Option 1: Separate Repositories**
```
github.com/company/inventory-service       # Application code
github.com/company/k8s-deployments         # All K8s manifests
```

**Option 2: Monorepo with Separation**
```
company-platform/
├── services/
│   ├── inventory-service/     # Application code
│   ├── product-service/
│   └── order-service/
└── deployment/
    ├── kubernetes/            # K8s manifests
    ├── helm/                  # Helm charts
    └── terraform/             # Infrastructure as Code
```

**Option 3: GitOps (ArgoCD)**
```
github.com/company/inventory-service       # Application code
github.com/company/gitops-configs          # ArgoCD syncs from here
```

---

## 🚀 Deployment Order

### **1. Create Namespace**
```bash
kubectl apply -f deployment/kubernetes/namespace/
```

### **2. Deploy PostgreSQL (Infrastructure)**
```bash
kubectl apply -f deployment/kubernetes/postgres/
```

### **3. Verify PostgreSQL is Running**
```bash
kubectl get pods -n microservices-dev
kubectl get svc -n microservices-dev
```

### **4. Deploy Microservices (Applications)**
```bash
kubectl apply -f deployment/kubernetes/services/
```

### **5. Verify All Services**
```bash
kubectl get all -n microservices-dev
```

---

## 🔍 Key Features in These Manifests

### **1. Resource Limits**
Every deployment has resource requests and limits:
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"
```

**Why:**
- Prevents resource starvation
- Kubernetes can schedule pods efficiently
- Protects cluster from memory leaks

---

### **2. Health Probes**
Every service has liveness and readiness probes:
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8083
  initialDelaySeconds: 60
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8083
  initialDelaySeconds: 30
  periodSeconds: 5
```

**Why:**
- **Liveness:** Kubernetes restarts pod if unhealthy
- **Readiness:** Pod receives traffic only when ready
- Prevents routing to broken pods

---

### **3. Labels & Selectors**
Consistent labeling for all resources:
```yaml
metadata:
  labels:
    app: inventory-service
    version: "1.0.0"
```

**Why:**
- Service discovery
- Monitoring and metrics
- Network policies
- Easy filtering with `kubectl`

---

### **4. Persistent Storage**
PostgreSQL uses PersistentVolumeClaim:
```yaml
volumeMounts:
- name: postgres-storage
  mountPath: /var/lib/postgresql/data
  subPath: postgres
```

**Why:**
- Data survives pod restarts
- Can snapshot and backup
- Can migrate to different nodes

---

### **5. Secrets & ConfigMaps**
Configuration separate from deployment:
```yaml
envFrom:
- configMapRef:
    name: postgres-config
- secretRef:
    name: postgres-secret
```

**Why:**
- Change config without rebuilding images
- Keep secrets encrypted
- Different configs per environment

---

### **6. ImagePullPolicy: Never**
For local Minikube development:
```yaml
imagePullPolicy: Never
```

**Why:**
- Uses locally built images (Minikube Docker)
- No need for Docker registry in dev
- **Change to `IfNotPresent` or `Always` in production!**

---

## 📋 Manifest Explanation

### **Namespace (namespace.yaml)**
Creates isolated environment for the application.

### **PostgreSQL Resources**

| File | Purpose |
|------|---------|
| `postgres-configmap.yaml` | Non-sensitive config (DB name, user) |
| `postgres-secret.yaml` | Sensitive data (password) |
| `postgres-pvc.yaml` | Storage request (1Gi) |
| `postgres-deployment.yaml` | PostgreSQL pod definition |
| `postgres-service.yaml` | DNS name: `postgres-service` |

### **Service Resources**

Each service file contains **both** Deployment and Service (separated by `---`):

**Deployment:**
- Docker image to use
- Number of replicas
- Resource limits
- Health checks
- Environment variables

**Service:**
- How to expose the deployment
- Port mappings
- Service type (ClusterIP, NodePort)

**⚠️ Note on api-gateway:**
- Uses **Spring Cloud Gateway** (reactive/WebFlux architecture)
- Excludes `spring-boot-starter-web` (Spring MVC) to avoid conflicts
- Runs on **Netty server** instead of Tomcat for non-blocking I/O
- This configuration is in `api-gateway/build.gradle`

---

## 🔄 Updating Deployments

### **Change Image Version**
```bash
# Edit deployment file, change image tag
image: inventory-service:1.0.1

# Apply changes
kubectl apply -f deployment/kubernetes/services/inventory-service.yaml

# Kubernetes will rolling update automatically
```

### **Scale Replicas**
```bash
# Edit deployment file
replicas: 3

# Or use kubectl directly
kubectl scale deployment inventory-service --replicas=3 -n microservices-dev
```

### **Change Configuration**
```bash
# Edit ConfigMap
kubectl edit configmap postgres-config -n microservices-dev

# Restart pods to pick up changes
kubectl rollout restart deployment inventory-service -n microservices-dev
```

---

## 🧹 Cleanup

### **Delete Services Only**
```bash
kubectl delete -f deployment/kubernetes/services/
```

### **Delete PostgreSQL Only**
```bash
kubectl delete -f deployment/kubernetes/postgres/
```

### **Delete Everything (including data!)**
```bash
kubectl delete namespace microservices-dev
```

---

## 🎓 Next Steps

After manual deployment, you can explore:

1. **Helm Charts** - Package these manifests as reusable charts
2. **Kustomize** - Manage configs for dev/staging/prod
3. **ArgoCD** - GitOps automated deployment
4. **CI/CD Pipeline** - Automate build → test → deploy

---

## 📚 Production Enhancements

These manifests are good for learning, but production needs:

### **1. Use ConfigMaps for Application Properties**
Instead of baking config into Docker images:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: inventory-config
data:
  application.properties: |
    spring.datasource.url=jdbc:postgresql://postgres-service:5432/inventorydb
    spring.jpa.hibernate.ddl-auto=validate
```

### **2. Use Ingress Instead of NodePort**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8080
```

### **3. Add Network Policies**
Control traffic between services:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: inventory-netpol
spec:
  podSelector:
    matchLabels:
      app: inventory-service
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: product-service
```

### **4. Use StatefulSet for PostgreSQL**
Better for stateful applications:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-service
  ...
```

### **5. Add HorizontalPodAutoscaler**
Auto-scale based on load:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inventory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inventory-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 🔐 Security Best Practices

### **1. Don't Commit Secrets**
Use tools like:
- **Sealed Secrets** (Bitnami)
- **External Secrets Operator**
- **Vault** (HashiCorp)

### **2. Use RBAC**
Limit what each service can do:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: inventory-service-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: inventory-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
```

### **3. Network Policies**
Deny all by default, allow specific traffic.

### **4. Pod Security Standards**
Enforce security policies:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices-dev
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

---

## 📖 References

- **Kubernetes Best Practices:** https://kubernetes.io/docs/concepts/configuration/overview/
- **12-Factor App:** https://12factor.net/
- **Production Readiness:** https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/

---

**Remember:** This deployment structure follows production best practices by separating infrastructure code from application code! 🚀
