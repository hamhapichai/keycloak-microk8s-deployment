# keycloak-microk8s-deployment
Keycloak with PostgreSQL database deployment on microk8s kubernetes with Auto-Scaling Setup Guide
# Keycloak with Database Deployment on Kubernetes Setup Guide
To create a scalable Keycloak deployment on a Kubernetes cluster managed by MicroK8s, you'll need to handle several aspects, including database setup, Keycloak configuration, auto-scaling, and potentially session clustering for high availability. This guide covers each step in detail:

## Prerequisites
- __MicroK8s Installation:__ Ensure MicroK8s is installed and running on your system. You can check the status and enable necessary add-ons with the following commands:
````
microk8s status
microk8s enable dns storage metrics-server ingress
````
- __Namespace Createion:__ Create a dedicated namespace for the Keycloak and PostgreSQL deployments with 01-namespace.yaml or use below command
````
microk8s kubectl create namespace keycloak
````

---
## Step 1: Create Namespace
Create namespace in kubernetes cluster
````
apiVersion: v1
kind: Namespace
metadata:
  name: keycloak
````
Apply to create namespace
````
microk8s kubectl apply -f 01-namespace.yaml
````
## Step 2: Deploy PostgreSQL
PostgreSQL serves as the relational database for Keycloak.

### 2.1 Create PostgreSQL Secrets
Store the database credentials using a Kubernetes secret for security.
````
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: keycloak
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: aab++112
  POSTGRES_DB: keycloak
````
Apply the secret by this command
````
microk8s kubectl apply -f 02-postgres-secret.yaml
````
### 2.2 PersistentVolumeClaim and PostgreSQL Deployment
Check if the `storage` addon is enabled:
````
microk8s enable storage
````
Create PostgreSQL with a PVC deployment:
````
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: keycloak
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: microk8s-hostpath

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: keycloak
spec:
  ports:
  - port: 5432
  selector:
    app: postgres

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: keycloak
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
        image: postgres:12
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_PASSWORD
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_DB
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
````
Apply the postgreSQL configuration:
````
microk8s kubectl apply -f 03-postgres.yaml
````
## Step 3: Deploy Keycloak
create keycloak deployment configuration
````
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
spec:
  ports:
  - port: 8080
  selector:
    app: keycloak

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:24.0.3
        args: ["start-dev"]
        resources:
          requests:
            cpu: "500m"
            memory: "1024Mi"
          limits:
            cpu: "1000m"
            memory: "2048Mi"
        env:
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: "admin"
        - name: DB_VENDOR
          value: "POSTGRES"
        - name: DB_ADDR
          value: "postgres"
        - name: DB_PORT
          value: "5432"
        - name: DB_DATABASE
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_DB
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_PASSWORD
        ports:
        - containerPort: 8080
````
Apply keycloak deployment configuration:
````
microk8s kubectl apply -f 04-keycloak.yaml
````
## Step 4: Configure Auto-Scaling with HPA
Enable auto-scaling for Keycloak based on CPU usage
````
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: keycloak-hpa
  namespace: keycloak
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: keycloak
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
````
Apply the HPA configuration:
````
microk8s kubectl apply -f 05-keycloak.yaml
````

## Step 5: Access Keycloak
Expose nodeport
````
microk8s kubectl expose deployment keycloak --type=NodePort --name=keycloak-nodeport --port=8080 -n keycloak
````
Get mapped nodeport to access keycloal
````
microk8s kubectl get svc keycloak-nodeport -n keycloak
````
Access keycloak admin menu via http://hostip:port like http://192.168.2.60:31560 with admin user
- username: admin
- password: admin
