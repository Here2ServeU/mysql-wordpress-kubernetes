# Deploying MySQL and WordPress Using Kubernetes

Here's a step-by-step guide to installing Minikube using Docker Compose, creating a persistent volume and claim, setting up MySQL with secrets, and deploying WordPress. 

---

## Step 1. Install Minikube

- Go to https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download.
- Choose 

```

## Step 2. Create a Persistent Volume and Persistent Volume Claim

Persistent Volumes (PV) provide storage for your Kubernetes applications, while Persistent Volume Claims (PVC) allow applications to request specific storage resources dynamically.

**Persistent Volume**:
- Defines a directory on the host system (/data/mysql) for MySQL data.
```bash
mkdir -p /data/mysql
```

**Create a Persistent Volume** (pv.yaml):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/mysql"
```

**Create a Persistent Volume Claim** (pvc.yaml):
- Requests 1Gi of storage from the defined Persistent Volume.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### Apply the files:
```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
```
## Step 3. Create a Secret for MySQL Credentials

**Create a Secret for MySQL** (mysql-secret.yaml):
- Secrets securely store sensitive data, such as MySQL login credentials. 
- The data is Base64 encoded.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  username: bXlzcWw=   # Base64 encoded value of 'mysql'
  password: cGFzc3dvcmQ= # Base64 encoded value of 'password'
```

**Apply the secret**:
```bash
kubectl apply -f mysql-secret.yaml
```
## Step 4. Deploy MySQL Using the Volume Claim and Secrets

This deployment sets up MySQL with the persistent volume for data storage and secrets for secure credential management.

**Create a MySQL Deployment** (mysql-deployment.yaml):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - mountPath: "/var/lib/mysql"
          name: mysql-storage
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

**Create a MySQL Service** (mysql-service.yaml):
- Exposes MySQL within the Kubernetes cluster on port 3306.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  clusterIP: None
```

**Apply the files**:
```bash
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
```

## Step 5. Deploy WordPress

- WordPress connects to MySQL using the credentials stored in the secret.

**Create a WordPress Deployment** (wordpress-deployment.yaml):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: username
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 80
```

**Create a WordPress Service** (wordpress-service.yaml):
- Exposes WordPress externally using a NodePort.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
spec:
  selector:
    app: wordpress
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```
**Apply the files**:
```bash
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml
```
## Step 6. Test the WordPress Application
**Get the NodePort for WordPress:**
```bash
kubectl get svc wordpress
```

**Open the application in your browser**: 
```text
http://<minikube_ip>:<node_port>.
```
**You can get the Minikube IP using**:
```bash
minikube ip
```

---

## Cleanup Steps
To clean up all resources and stop Minikube:

**Delete all Kubernetes resources**:
```bash
kubectl delete svc,deploy,pvc,pv,secret --all
```

**Stop and remove Minikube**:
```bash
docker-compose down
```

**Remove data directories**:
```bash
rm -rf /data/mysql
rm -rf /var/lib/minikube
```

---

This setup provides a fully functional WordPress application backed by MySQL with persistent storage and secret management.
