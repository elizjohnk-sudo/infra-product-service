# 🚀 Step-by-Step Deployment Guide to Minikube

## Overview

This guide will walk you through:
1. ✅ Building services locally with Gradle
2. ✅ Creating Docker images for each service
3. ✅ Deploying PostgreSQL to Minikube
4. ✅ Deploying all 4 microservices to Minikube
5. ✅ Testing the complete application in Kubernetes

---

## 📋 Prerequisites Check

Before starting, verify you have:

```powershell
# Check Java version (need 17+)
java --version

# Check Docker in WSL
wsl -e bash -c "docker --version"

# Check kubectl
wsl -e bash -c "kubectl version --client"

# Check Minikube
wsl -e bash -c "minikube status"
```

Expected outputs:
- Java: 17 or higher
- Docker: Any recent version
- kubectl: Any version
- Minikube: Running

---

## 🎯 Phase 1: Build Services Locally

### **Step 1.1: Clean Previous Builds**

```powershell
# In PowerShell (Windows)
cd D:\D-Downloads\trial-ms

# Clean all previous builds
.\gradlew.bat clean
```

**What this does:**
- Removes all `build/` folders
- Clears compiled `.class` files
- Fresh start for building

**Expected output:**
```
BUILD SUCCESSFUL in 2s
```

---

### **Step 1.2: Build All Services**

```powershell
# Build all 4 services
.\gradlew.bat build
```

**What this does:**
- Compiles Java code for all 4 services
- Runs unit tests
- Packages each service into a JAR file
- Creates:
  - `api-gateway/build/libs/api-gateway-1.0.0.jar`
  - `order-service/build/libs/order-service-1.0.0.jar`
  - `product-service/build/libs/product-service-1.0.0.jar`
  - `inventory-service/build/libs/inventory-service-1.0.0.jar`

**Expected output:**
```
BUILD SUCCESSFUL in 15s
```

**⚠️ If build fails:**
- Read the error message carefully
- Common issues: Java version, syntax errors, missing dependencies
- Fix and run `.\gradlew.bat build` again

---

### **Step 1.3: Verify JAR Files Created**

```powershell
# Check that JAR files exist
Get-ChildItem -Recurse -Filter "*.jar" | Where-Object { $_.Directory.Name -eq "libs" } | Select-Object FullName
```

**Expected output:**
```
api-gateway/build/libs/api-gateway-1.0.0.jar
order-service/build/libs/order-service-1.0.0.jar
product-service/build/libs/product-service-1.0.0.jar
inventory-service/build/libs/inventory-service-1.0.0.jar
```

---

## ☸️ Phase 2: Deploy PostgreSQL to Kubernetes

### **Important: Deploy Infrastructure First**

Following production best practices, we deploy the database (infrastructure) before the services that depend on it.

---

### **Step 2.1: Switch to WSL and Verify Cluster**

```powershell
# Switch to WSL
wsl
```

```bash
# In WSL - verify Minikube is running
minikube status

# Verify kubectl works
kubectl cluster-info

# If Minikube is stopped, start it
minikube start
```

**Expected output:**
```
minikube
type: Control Plane
host: Running
kubelet: Running
```

---

### **Step 2.2: Create Kubernetes Namespace**

```bash
# Apply namespace manifest
kubectl apply -f deployment/kubernetes/namespace/namespace.yaml

# Verify namespace created
kubectl get namespaces | grep product-services-dev
```

**Expected output:**
```
namespace/product-services-dev created
```

**Verify:**
```bash
kubectl get namespace product-services-dev
```

**Output:**
```
NAME                   STATUS   AGE
product-services-dev   Active   5s
```

**What this does:**
- Creates isolated environment for your application
- All resources will be deployed in this namespace
- Prevents conflicts with other projects

**Note:** We won't set this as the default namespace. All commands will explicitly use `-n product-services-dev` for clarity and safety.

---

### **Step 2.3: Deploy PostgreSQL Configuration**

```bash
# Apply ConfigMap (non-sensitive configuration)
kubectl apply -f deployment/kubernetes/postgres/postgres-configmap.yaml

# Apply Secret (password)
kubectl apply -f deployment/kubernetes/postgres/postgres-secret.yaml

# Apply PersistentVolumeClaim (storage)
kubectl apply -f deployment/kubernetes/postgres/postgres-pvc.yaml
```

**Expected output:**
```
configmap/postgres-config created
secret/postgres-secret created
persistentvolumeclaim/postgres-pvc created
```

**What these do:**
- **ConfigMap:** Stores database name, username
- **Secret:** Stores password (base64 encoded)
- **PVC:** Requests 1Gi of persistent storage

---

### **Step 2.4: Deploy PostgreSQL**

```bash
# Apply PostgreSQL Deployment
kubectl apply -f deployment/kubernetes/postgres/postgres-deployment.yaml

# Apply PostgreSQL Service (creates DNS name)
kubectl apply -f deployment/kubernetes/postgres/postgres-service.yaml
```

**Expected output:**
```
deployment.apps/postgres created
service/postgres-service created
```

**⏱️ Wait:** ~30-60 seconds for PostgreSQL to start

---

### **Step 2.5: Verify PostgreSQL is Running**

```bash
# Check PostgreSQL pod status
kubectl get pods -n product-services-dev

# Expected output:
# NAME                        READY   STATUS    RESTARTS   AGE
# postgres-xxxxxxxxxx-xxxxx   1/1     Running   0          45s
```

**If pod is not ready:**
```bash
# Check pod details
kubectl describe pod <postgres-pod-name> -n product-services-dev

# Check logs
kubectl logs <postgres-pod-name> -n product-services-dev
```

**Wait until:**
- STATUS = `Running`
- READY = `1/1`

---

### **Step 2.6: Verify PostgreSQL Service**

```bash
# Check service
kubectl get svc -n product-services-dev

# Expected output:
# NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# postgres-service   ClusterIP   10.96.xxx.xxx   <none>        5432/TCP   1m
```

**Important details:**
- **Service Name:** `postgres-service` (this is the DNS name)
- **Port:** `5432`
- **Type:** `ClusterIP` (internal only, perfect for database)

---

### **Step 2.7: Test PostgreSQL Connection**

```bash
# Get pod name dynamically
export POSTGRES_POD=$(kubectl get pods -n product-services-dev -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Connect to PostgreSQL
kubectl exec -it $POSTGRES_POD -n product-services-dev -- psql -U postgres -d inventorydb
```

**Inside psql:**
```sql
-- List databases
\l

-- Connect to inventorydb
\c inventorydb

-- List tables (should be empty for now)
\dt

-- Exit
\q
```

**Expected:**
```
inventorydb=# \dt
Did not find any relations.
```

**✅ This is correct!** The `inventory` table will be created automatically by the inventory-service when it first starts.

---

### **Step 2.8: Confirm Connection Details**

You now have PostgreSQL running with these confirmed details:

| Setting | Value |
|---------|-------|
| **Host** | `postgres-service` (Kubernetes DNS) |
| **Port** | `5432` |
| **Database** | `inventorydb` |
| **Username** | `postgres` |
| **Password** | `postgres` |

**Connection String for Spring Boot:**
```
jdbc:postgresql://postgres-service:5432/inventorydb
```

---

## 🔧 Phase 3: Update Application Configuration

Now that PostgreSQL is running, update your services to use the **confirmed** connection details.

**Note:** For inventory-service, we've externalized the database configuration using Kubernetes ConfigMaps and Secrets. The configuration is managed in `deployment/kubernetes/services/inventory-config.yaml` and will be injected as environment variables at runtime. The `application.properties` file uses placeholders like `${DB_HOST:localhost}` that automatically read these environment variables.

---

### **Step 3.1: Verify inventory-service Configuration**

The inventory-service is already configured with environment variable placeholders:

**File:** `inventory-service/src/main/resources/application.properties`

```properties
server.port=8083
spring.application.name=inventory-service

# PostgreSQL Configuration (uses environment variables from ConfigMap/Secret)
spring.datasource.url=jdbc:postgresql://${DB_HOST:localhost}:5432/${DB_NAME:inventorydb}
spring.datasource.username=${DB_USER:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Actuator
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
```

**Key change:**
- `spring.datasource.url=jdbc:postgresql://postgres-service:5432/inventorydb`
- Uses Kubernetes service name instead of `localhost`

---

### **Step 3.2: Update product-service Configuration**

**File:** `product-service/src/main/resources/application.properties`

```properties
server.port=8082
spring.application.name=product-service

# Inventory Service URL (Kubernetes DNS)
inventory.service.url=http://inventory-service:8083

# Actuator
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
```

**Key change:**
- `inventory.service.url=http://inventory-service:8083`
- Uses Kubernetes service name

---

### **Step 3.3: Update order-service Configuration**

**File:** `order-service/src/main/resources/application.properties`

```properties
server.port=8081
spring.application.name=order-service

# Product Service URL (Kubernetes DNS)
product.service.url=http://product-service:8082

# Actuator
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
```

**Key change:**
- `product.service.url=http://product-service:8082`

---

### **Step 3.4: Update api-gateway Configuration**

**File:** `api-gateway/src/main/resources/application.properties`

```properties
server.port=8080
spring.application.name=api-gateway

# Gateway Routes (Kubernetes DNS)
spring.cloud.gateway.routes[0].id=order-service
spring.cloud.gateway.routes[0].uri=http://order-service:8081
spring.cloud.gateway.routes[0].predicates[0]=Path=/api/orders/**

spring.cloud.gateway.routes[1].id=product-service
spring.cloud.gateway.routes[1].uri=http://product-service:8082
spring.cloud.gateway.routes[1].predicates[0]=Path=/api/products/**

spring.cloud.gateway.routes[2].id=inventory-service
spring.cloud.gateway.routes[2].uri=http://inventory-service:8083
spring.cloud.gateway.routes[2].predicates[0]=Path=/api/inventory/**

# Actuator
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
```

**Key changes:**
- All service URIs use Kubernetes service names
- Added health state endpoints for probes

---

### **Step 3.5: Rebuild Services with Updated Configuration**

```powershell
# In PowerShell (Windows)
cd D:\D-Downloads\trial-ms

# Clean and rebuild all services
.\gradlew.bat clean build
```

**Expected output:**
```
BUILD SUCCESSFUL in 15s
```

**⏱️ Time:** ~15-20 seconds

---

## 🐳 Phase 4: Create Docker Images

Now that PostgreSQL is deployed and configuration is updated, create Docker images.

---

### **Step 4.1: Point Docker to Minikube**

```powershell
# Switch to WSL (if not already there)
wsl
```

```bash
# Point Docker CLI to Minikube's Docker daemon
eval $(minikube docker-env)

# Verify you're using Minikube Docker
echo $DOCKER_HOST
# Should show: tcp://127.0.0.1:xxxxx

# Verify
docker info | grep "Name:"
# Should show: Name: minikube
```

**Why this is critical:**
- Builds images **inside Minikube's Docker**
- Kubernetes can use these images **without a registry**
- Images are immediately available

**⚠️ Important Note on api-gateway:**
The api-gateway uses **Spring Cloud Gateway** which requires **Spring WebFlux** (reactive/non-blocking). It excludes `spring-boot-starter-web` (Spring MVC) to avoid conflicts. This is configured in `api-gateway/build.gradle` and allows the gateway to use Netty server instead of Tomcat for better performance.

---

### **Step 4.2: Create Dockerfiles**

You need to create a `Dockerfile` in each service directory.

#### **Create: api-gateway/Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY build/libs/api-gateway-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### **Create: order-service/Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY build/libs/order-service-1.0.0.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### **Create: product-service/Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY build/libs/product-service-1.0.0.jar app.jar
EXPOSE 8082
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### **Create: inventory-service/Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY build/libs/inventory-service-1.0.0.jar app.jar
EXPOSE 8083
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Dockerfile Explanation:**
- `FROM eclipse-temurin:17-jre-alpine`: Base image (Java 17 runtime, lightweight)
- `WORKDIR /app`: Set working directory inside container
- `COPY build/libs/*.jar app.jar`: Copy JAR from build to container
- `EXPOSE 8080`: Document which port the service uses
- `ENTRYPOINT ["java", "-jar", "app.jar"]`: Command to run when container starts

---

### **Step 2.2: Build Docker Images**

**In WSL (with Minikube Docker environment active):**

```bash
# Navigate to project directory
cd /mnt/d/D-Downloads/trial-ms

# Build api-gateway image
docker build -t api-gateway:1.0.0 ./api-gateway

# Build order-service image
docker build -t order-service:1.0.0 ./order-service

# Build product-service image
docker build -t product-service:1.0.0 ./product-service

# Build inventory-service image
docker build -t inventory-service:1.0.0 ./inventory-service
```

**What each command does:**
- `-t api-gateway:1.0.0`: Tags the image with name and version
- `./api-gateway`: Path to the directory containing Dockerfile

**Expected output for each:**
```
Successfully built abc123def456
Successfully tagged api-gateway:1.0.0
```

**⏱️ Time:** ~2-3 minutes per image (first time downloads base image)

---

### **Step 4.3: Build Docker Images**

**In WSL (with Minikube Docker environment active):**

```bash
# Navigate to project directory
cd /mnt/d/D-Downloads/trial-ms

# Build api-gateway image
docker build -t api-gateway:1.0.0 ./api-gateway

# Build order-service image
docker build -t order-service:1.0.0 ./order-service

# Build product-service image
docker build -t product-service:1.0.0 ./product-service

# Build inventory-service image
docker build -t inventory-service:1.0.0 ./inventory-service
```

**What each command does:**
- `-t api-gateway:1.0.0`: Tags the image with name and version
- `./api-gateway`: Path to the directory containing Dockerfile

**Expected output for each:**
```
Successfully built abc123def456
Successfully tagged api-gateway:1.0.0
```

**⏱️ Time:** ~2-3 minutes per image (first time downloads base image)

---

### **Step 4.4: Verify Images Are Built**

```bash
# List all your images
docker images | grep -E "api-gateway|order-service|product-service|inventory-service"
```

**Expected output:**
```
api-gateway          1.0.0   abc123   2 minutes ago   250MB
order-service        1.0.0   def456   2 minutes ago   250MB
product-service      1.0.0   ghi789   2 minutes ago   250MB
inventory-service    1.0.0   jkl012   2 minutes ago   250MB
```

---

## 🚀 Phase 5: Deploy Microservices to Kubernetes

Now deploy your services to Kubernetes using the manifests in the `deployment/` folder.

---

### **Step 5.1: Deploy Service Configuration (ConfigMaps)**

First, deploy the ConfigMaps that contain the application configuration:

```bash
# Deploy inventory-service configuration
kubectl apply -f deployment/kubernetes/services/inventory-config.yaml
```

**Expected output:**
```
configmap/inventory-config created
```

**What this does:**
- Creates a ConfigMap with database connection details (DB_HOST, DB_NAME, DB_USER)
- Externalizes configuration from application code
- The inventory-service will inject these as environment variables
- Database password comes from the existing `postgres-secret`

**Verify:**
```bash
# Check the ConfigMap
kubectl get configmap inventory-config -n product-services-dev

# View ConfigMap contents
kubectl describe configmap inventory-config -n product-services-dev
```

---

### **Step 5.2: Deploy All Services**

```bash
# Deploy in order (bottom-up: inventory → product → order → gateway)
kubectl apply -f deployment/kubernetes/services/inventory-service.yaml
kubectl apply -f deployment/kubernetes/services/product-service.yaml
kubectl apply -f deployment/kubernetes/services/order-service.yaml
kubectl apply -f deployment/kubernetes/services/api-gateway.yaml
```

**Expected output:**
```
deployment.apps/inventory-service created
service/inventory-service created
deployment.apps/product-service created
service/product-service created
deployment.apps/order-service created
service/order-service created
deployment.apps/api-gateway created
service/api-gateway created
```

---

### **Step 5.3: Verify All Pods Are Running**

```bash
# Check all pods
kubectl get pods -n product-services-dev

# Expected output:
# NAME                                 READY   STATUS    RESTARTS   AGE
# postgres-xxxxxxxxxx-xxxxx            1/1     Running   0          10m
# inventory-service-xxxxxxxxxx-xxxxx   1/1     Running   0          45s
# product-service-xxxxxxxxxx-xxxxx     1/1     Running   0          45s
# order-service-xxxxxxxxxx-xxxxx       1/1     Running   0          45s
# api-gateway-xxxxxxxxxx-xxxxx         1/1     Running   0          45s
```

**⏱️ Wait:** 30-60 seconds for all pods to be `Running` and `1/1 Ready`

**If a pod is not ready:**
```bash
# Describe the pod (shows events)
kubectl describe pod <pod-name> -n product-services-dev

# Check logs
kubectl logs <pod-name> -n product-services-dev

# Follow logs in real-time
kubectl logs -f <pod-name> -n product-services-dev
```

**Common startup time:**
- PostgreSQL: ~30 seconds
- Inventory Service: ~45 seconds (waits for PostgreSQL + JPA setup)
- Product/Order Services: ~30 seconds
- API Gateway: ~30 seconds

---

### **Step 5.4: Check Services**

```bash
# List all services
kubectl get svc -n product-services-dev
kubectl get svc -n product-services-dev

# Expected output:
# NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
# postgres-service    ClusterIP   10.96.1.1        <none>        5432/TCP         10m
# inventory-service   ClusterIP   10.96.1.2        <none>        8083/TCP         2m
# product-service     ClusterIP   10.96.1.3        <none>        8082/TCP         2m
# order-service       ClusterIP   10.96.1.4        <none>        8081/TCP         2m
# api-gateway         NodePort    10.96.1.5        <none>        8080:30080/TCP   2m
```

**Key observations:**
- PostgreSQL and microservices are `ClusterIP` (internal only)
- API Gateway is `NodePort` (externally accessible on port 30080)

---

### **Step 5.5: Check Database Table Creation**

```bash
# Get postgres pod name
export POSTGRES_POD=$(kubectl get pods -n product-services-dev -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Check if inventory table was created
kubectl exec -it $POSTGRES_POD -n product-services-dev -- psql -U postgres -d inventorydb -c "\dt"
```

**Expected output:**
```
           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | inventory | table | postgres
```

**✅ Success!** The `inventory` table was automatically created by Spring Boot JPA when inventory-service started!

---

## 🧪 Phase 6: Test the Application

### **Step 6.1: Get Minikube IP**

```bash
# Get Minikube IP address
minikube ip
```

**Example output:**
```
192.168.49.2
```

Your API Gateway is accessible at: `http://192.168.49.2:30080`

---

### **Step 6.2: Test Health Endpoints**

```bash
# Set Minikube IP as variable
export MINIKUBE_IP=$(minikube ip)

# Test API Gateway health
curl http://$MINIKUBE_IP:30080/actuator/health

# Expected output:
# {"status":"UP","groups":["liveness","readiness"]}
```

---

### **Step 6.3: Create Inventory**

```bash
# Create inventory for product PROD-001
curl -X POST http://$MINIKUBE_IP:30080/api/inventory \
  -H "Content-Type: application/json" \
  -d '{
    "productId": "PROD-001",
    "quantity": 100
  }'
```

**Expected output:**
```json
{
  "id": 1,
  "productId": "PROD-001",
  "quantity": 100,
  "inStock": true,
  "lastUpdated": "2026-02-11T..."
}
```

**🎯 This proves:** 
- API Gateway → Inventory Service communication works
- Inventory Service → PostgreSQL connection works
- Data is persisted in database

---

### **Step 6.4: Verify Data in PostgreSQL**

```bash
# Check inventory table
kubectl exec -it $POSTGRES_POD -n product-services-dev -- psql -U postgres -d inventorydb -c "SELECT * FROM inventory;"
```

**Expected output:**
```
 id | product_id | quantity | in_stock | last_updated
----+------------+----------+----------+--------------
  1 | PROD-001   |      100 | t        | 2026-02-11...
```

**✅ Data persisted!**

---

### **Step 6.5: Create Product**

```bash
# Create product
curl -X POST http://$MINIKUBE_IP:30080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "productId": "PROD-001",
    "name": "Gaming Laptop",
    "description": "High-performance gaming laptop",
    "price": 1299.99,
    "category": "Electronics"
  }'
```

**Expected output:**
```json
{
  "productId": "PROD-001",
  "name": "Gaming Laptop",
  "description": "High-performance gaming laptop",
  "price": 1299.99,
  "category": "Electronics",
  "availableQuantity": 100,
  "inStock": true
}
```

**🎯 This proves:** 
- Product Service → Inventory Service communication works
- Product Service correctly fetched inventory quantity (100)

---

### **Step 6.6: Create Order**

```bash
# Create order
curl -X POST http://$MINIKUBE_IP:30080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "CUST-001",
    "productId": "PROD-001",
    "quantity": 2
  }'
```

**Expected output:**
```json
{
  "orderId": "...",
  "customerId": "CUST-001",
  "productId": "PROD-001",
  "productName": "Gaming Laptop",
  "quantity": 2,
  "totalPrice": 2599.98,
  "status": "PENDING",
  "orderDate": "2026-02-11T..."
}
```

**🎯 This proves:** Complete chain works!
```
API Gateway (8080)
    ↓
Order Service (8081)
    ↓
Product Service (8082)
    ↓
Inventory Service (8083)
    ↓
PostgreSQL (5432)
```

---

### **Step 6.7: Get All Orders**

```bash
# Get all orders
curl http://$MINIKUBE_IP:30080/api/orders
```

**Expected output:**
```json
[
  {
    "orderId": "...",
    "customerId": "CUST-001",
    "productId": "PROD-001",
    "productName": "Gaming Laptop",
    "quantity": 2,
    "totalPrice": 2599.98,
    "status": "PENDING",
    "orderDate": "2026-02-11T..."
  }
]
```

---

## 🎉 Phase 7: Verify Complete Deployment

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: product-services-dev
data:
  POSTGRES_DB: inventorydb
  POSTGRES_USER: postgres
```

**What this does:**
- Stores PostgreSQL configuration
- Non-sensitive data (database name, username)

---

#### **Create: k8s/postgres/postgres-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: product-services-dev
type: Opaque
data:
  POSTGRES_PASSWORD: cG9zdGdyZXM=  # base64 encoded "postgres"
```

**What this does:**
- Stores sensitive data (password)
- `cG9zdGdyZXM=` is base64 encoding of "postgres"

**To encode your own password:**
```bash
echo -n "your-password" | base64
```

---

#### **Create: k8s/postgres/postgres-pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: product-services-dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**What this does:**
- Requests persistent storage for PostgreSQL data
- `1Gi`: 1 gigabyte of storage
- Data persists even if pod is deleted

---

#### **Create: k8s/postgres/postgres-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: product-services-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        envFrom:
        - configMapRef:
            name: postgres-config
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
          subPath: postgres
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

**What this does:**
- Creates 1 PostgreSQL pod
- Uses ConfigMap and Secret for configuration
- Mounts persistent volume for data storage
- Exposes port 5432

---

#### **Create: k8s/postgres/postgres-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: product-services-dev
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

**What this does:**
- Creates internal DNS name: `postgres-service`
- Other services can connect using: `postgres-service:5432`
- `ClusterIP`: Only accessible within cluster

---

### **Step 3.3: Deploy PostgreSQL**

```bash
# Apply all PostgreSQL manifests
kubectl apply -f k8s/postgres/postgres-configmap.yaml
kubectl apply -f k8s/postgres/postgres-secret.yaml
kubectl apply -f k8s/postgres/postgres-pvc.yaml
kubectl apply -f k8s/postgres/postgres-deployment.yaml
kubectl apply -f k8s/postgres/postgres-service.yaml
```

**Expected output:**
```
configmap/postgres-config created
secret/postgres-secret created
persistentvolumeclaim/postgres-pvc created
deployment.apps/postgres created
service/postgres-service created
```

---

### **Step 3.4: Verify PostgreSQL is Running**

```bash
# Check pods
kubectl get pods

# Expected output:
# NAME                        READY   STATUS    RESTARTS   AGE
# postgres-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

# Check service
kubectl get svc

# Expected output:
# NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# postgres-service   ClusterIP   10.96.123.456   <none>        5432/TCP   30s

# Check PVC
kubectl get pvc

# Expected output:
# NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   AGE
# postgres-pvc   Bound    pvc-xxxxx...  1Gi        RWO            30s
```

**⏱️ Wait:** PostgreSQL pod should be `Running` and `1/1 Ready`

**If pod is not ready:**
```bash
# Check pod details
kubectl describe pod postgres-xxxxxxxxxx-xxxxx

# Check logs
kubectl logs postgres-xxxxxxxxxx-xxxxx
```

---

### **Step 3.5: Test PostgreSQL Connection**

```bash
# Get pod name
export POSTGRES_POD=$(kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Connect to PostgreSQL
kubectl exec -it $POSTGRES_POD -- psql -U postgres -d inventorydb

# Inside psql:
inventorydb=# \dt
# Should show: No relations found (table will be created by inventory-service)

inventorydb=# \q
# Exit psql
```

---

## 🌐 Access Options

You have multiple ways to access your deployed services:

### **Option 1: Minikube IP with NodePort** ⭐ **Production-like**

```bash
# Get Minikube IP
minikube ip
# Example: 192.168.49.2

# Access from WSL or Windows browser
curl http://192.168.49.2:30080/actuator/health
```

**Browser URLs:**
```
http://192.168.49.2:30080/actuator/health
http://192.168.49.2:30080/api/inventory
http://192.168.49.2:30080/api/products
http://192.168.49.2:30080/api/orders
```

**Advantages:**
- ✅ No port conflicts (not using localhost)
- ✅ Production-like setup (NodePort)
- ✅ Always accessible if Minikube is running

---

### **Option 2: Port Forwarding** ⭐ **Development-friendly**

Port forward allows you to access services on `localhost` from Windows browser.

#### **Forward api-gateway (Access All Services):**

```bash
# In WSL
kubectl port-forward service/api-gateway 9090:8080 -n product-services-dev
```

**Keep this terminal open!** Port forwarding runs in foreground.

**Access from Windows browser:**
```
http://localhost:9090/actuator/health
http://localhost:9090/api/inventory
http://localhost:9090/api/products
http://localhost:9090/api/orders
```

**Advantages:**
- ✅ No port conflicts (uses port 9090 instead of 8080)
- ✅ Works from Windows browser on localhost
- ✅ Familiar localhost URLs

---

#### **Forward Individual Services (For Debugging):**

**Open multiple WSL terminals:**

**Terminal 1 - api-gateway:**
```bash
kubectl port-forward service/api-gateway 9090:8080 -n product-services-dev
```

**Terminal 2 - inventory-service:**
```bash
kubectl port-forward service/inventory-service 9093:8083 -n product-services-dev
```

**Terminal 3 - product-service:**
```bash
kubectl port-forward service/product-service 9092:8082 -n product-services-dev
```

**Terminal 4 - order-service:**
```bash
kubectl port-forward service/order-service 9091:8081 -n product-services-dev
```

**Access URLs:**
```
Gateway:    http://localhost:9090/actuator/health
Inventory:  http://localhost:9093/actuator/health
Product:    http://localhost:9092/actuator/health
Order:      http://localhost:9091/actuator/health
```

---

#### **Port Forward in Background (tmux method):**

```bash
# Install tmux (if not already)
sudo apt update && sudo apt install tmux -y

# Start tmux session
tmux new -s portforwards

# Run port forward
kubectl port-forward service/api-gateway 9090:8080 -n product-services-dev

# Detach from tmux: Press Ctrl+b, then d
# Terminal is free, but port forward keeps running!

# To reattach later:
tmux attach -t portforwards

# To kill the session:
tmux kill-session -t portforwards
```

---

#### **Port Forward Script (Multiple Services):**

Create `~/port-forward.sh`:

```bash
#!/bin/bash

echo "Starting port forwards for microservices..."

# Port forward api-gateway
kubectl port-forward service/api-gateway 9090:8080 -n product-services-dev &
PF1=$!

# Port forward inventory-service
kubectl port-forward service/inventory-service 9093:8083 -n product-services-dev &
PF2=$!

# Port forward product-service
kubectl port-forward service/product-service 9092:8082 -n product-services-dev &
PF3=$!

# Port forward order-service
kubectl port-forward service/order-service 9091:8081 -n product-services-dev &
PF4=$!

echo "Port forwards started!"
echo "Gateway:    http://localhost:9090"
echo "Inventory:  http://localhost:9093"
echo "Product:    http://localhost:9092"
echo "Order:      http://localhost:9091"
echo ""
echo "Press Ctrl+C to stop all port forwards"

# Wait and cleanup on exit
trap "kill $PF1 $PF2 $PF3 $PF4 2>/dev/null" EXIT

wait
```

**Run:**
```bash
chmod +x ~/port-forward.sh
~/port-forward.sh
```

**Press Ctrl+C to stop all port forwards.**

---

### **Option 3: Minikube Service Command** ⭐ **Automatic**

```bash
# This automatically opens browser with correct URL
minikube service api-gateway -n product-services-dev

# Or just get the URL
minikube service api-gateway -n product-services-dev --url
```

**Advantages:**
- ✅ Automatically finds correct IP and port
- ✅ Opens browser automatically
- ✅ Handles port forwarding if needed

---

### **Port Mapping Summary**

| Service | Kubernetes Port | Port Forward | Access URL |
|---------|----------------|--------------|------------|
| **api-gateway** | 8080 | 9090 | `localhost:9090` |
| **inventory-service** | 8083 | 9093 | `localhost:9093` |
| **product-service** | 8082 | 9092 | `localhost:9092` |
| **order-service** | 8081 | 9091 | `localhost:9091` |
| **postgres** | 5432 | 5433 | `localhost:5433` |

---

### **Recommended Setup**

**For Quick Testing:**
```bash
# Use Minikube IP (no port forward needed)
minikube ip
# Access: http://192.168.49.2:30080
```

**For Windows Browser Development:**
```bash
# Use port forward
kubectl port-forward service/api-gateway 9090:8080 -n product-services-dev
# Access: http://localhost:9090
```

**For Individual Service Debugging:**
```bash
# Run port-forward script for all services
~/port-forward.sh
# Access each service on different port
```

---

## 🔍 Troubleshooting

### **Problem: api-gateway pod fails with "SpringMvcFoundOnClasspathConfiguration" error**

**Error message:**
```
Error creating bean with name 'org.springframework.cloud.gateway.config.GatewayClassPathWarningAutoConfiguration$SpringMvcFoundOnClasspathConfiguration'
```

**Root Cause:**
Spring Cloud Gateway requires Spring WebFlux (reactive) and cannot work with Spring MVC (blocking).

**Solution:**
This is already fixed in `api-gateway/build.gradle` which excludes `spring-boot-starter-web`. If you see this error:

```bash
# Rebuild the project
.\gradlew.bat clean build

# Rebuild Docker image
eval $(minikube docker-env)
docker build -t api-gateway:1.0.0 ./api-gateway

# Restart the pod
kubectl delete pod <api-gateway-pod> -n product-services-dev
```

---

### **Problem: Pod stuck in "ImagePullBackOff"**

```bash
# Check pod events
kubectl describe pod <pod-name>
```

**Solution:** Make sure you built images with `eval $(minikube docker-env)` active

```bash
# Rebuild with Minikube Docker
eval $(minikube docker-env)
docker build -t inventory-service:1.0.0 ./inventory-service
```

---

### **Problem: Pod stuck in "CrashLoopBackOff"**

```bash
# Check logs
kubectl logs <pod-name>
```

**Common causes:**
- Can't connect to PostgreSQL (check service name in application.properties)
- Can't connect to other services (check service names)
- Application error (check logs for Java exceptions)

---

### **Problem: "Connection refused" between services**

**Check:**
1. Service is running: `kubectl get pods`
2. Service exists: `kubectl get svc`
3. Correct service name in application.properties
4. Correct port number

**Test connectivity:**
```bash
# From one pod to another
kubectl exec -it <order-service-pod> -- wget -O- http://product-service:8082/actuator/health
```

---

### **Problem: Can't access API Gateway from browser**

**Check:**
1. Minikube is running: `minikube status`
2. Get correct IP: `minikube ip`
3. Use NodePort: `http://<minikube-ip>:30080`
4. API Gateway pod is running: `kubectl get pods`

---

## 📊 Useful Commands

### **View Pod Logs:**
```bash
# View logs
kubectl logs <pod-name> -n product-services-dev

# Follow logs (real-time)
kubectl logs -f <pod-name> -n product-services-dev

# Previous logs (if pod restarted)
kubectl logs --previous <pod-name> -n product-services-dev
```

### **Execute Commands in Pod:**
```bash
# Shell into pod
kubectl exec -it <pod-name> -n product-services-dev -- /bin/sh

# Run single command
kubectl exec <pod-name> -n product-services-dev -- env
```

### **Port Forward (for local testing):**
```bash
# Forward local port to pod
kubectl port-forward pod/<pod-name> 8080:8080 -n product-services-dev

# Forward to service
kubectl port-forward svc/api-gateway 8080:8080 -n product-services-dev

# Now access: http://localhost:8080
```

### **Scale Deployments:**
```bash
# Scale to 3 replicas
kubectl scale deployment inventory-service --replicas=3 -n product-services-dev

# Check
kubectl get pods -n product-services-dev
```

### **Delete Resources:**
```bash
# Delete a deployment
kubectl delete deployment inventory-service -n product-services-dev

# Delete a service
kubectl delete service inventory-service -n product-services-dev

# Delete everything in namespace
kubectl delete all --all -n product-services-dev

# Delete namespace (and everything in it)
kubectl delete namespace product-services-dev
```

---

## 🎯 Summary of What You've Learned

### **Phase 1: Local Build**
✅ Gradle builds JAR files  
✅ JAR files are executable applications  

### **Phase 2: Containerization**
✅ Dockerfiles package JARs into images  
✅ Docker images are portable  
✅ Minikube has its own Docker daemon  

### **Phase 3: PostgreSQL in Kubernetes**
✅ ConfigMaps store configuration  
✅ Secrets store sensitive data  
✅ PersistentVolumeClaims provide storage  
✅ Deployments manage pods  
✅ Services provide stable DNS names  

### **Phase 4: Microservices in Kubernetes**
✅ Services communicate via Kubernetes DNS  
✅ `imagePullPolicy: Never` uses local images  
✅ NodePort exposes services externally  
✅ ClusterIP for internal communication  

### **Phase 5: Testing**
✅ Complete request flow through all services  
✅ Data persists in PostgreSQL  
✅ Kubernetes networking works  

---

## 🚀 Next Steps (Future Phases)

### **Phase 6: Ingress Controller**
- Install Nginx Ingress
- Single external IP for all services
- Path-based routing

### **Phase 7: ConfigMaps for Properties**
- Externalize configuration
- Change config without rebuilding images

### **Phase 8: Health Checks**
- Liveness probes
- Readiness probes
- Kubernetes auto-healing

### **Phase 9: Resource Limits**
- CPU limits
- Memory limits
- Prevent resource starvation

### **Phase 10: ArgoCD (GitOps)**
- Automated deployments
- Git as source of truth
- Declarative infrastructure

---

## 📚 Key Kubernetes Concepts Learned

| Concept | Purpose | Example |
|---------|---------|---------|
| **Namespace** | Logical isolation | `product-services-dev` |
| **Pod** | Smallest unit, runs containers | `inventory-service-xxx` |
| **Deployment** | Manages pod replicas | `replicas: 1` |
| **Service** | Stable DNS name, load balancer | `postgres-service` |
| **ConfigMap** | Non-sensitive configuration | Database name |
| **Secret** | Sensitive data | Passwords |
| **PersistentVolumeClaim** | Storage for data | PostgreSQL data |
| **ClusterIP** | Internal-only service | Most services |
| **NodePort** | External access | API Gateway |

---

## 🎓 Congratulations!

You've successfully:
- ✅ Built a microservices application locally
- ✅ Containerized all services with Docker
- ✅ Deployed PostgreSQL to Kubernetes
- ✅ Deployed 4 microservices to Kubernetes
- ✅ Tested the complete application
- ✅ Learned core Kubernetes concepts

**You're now ready for advanced topics like ArgoCD, Istio, and production best practices!** 🚀
