# Deploying MySQL and WordPress Using Kubernetes

Here's a step-by-step guide to installing Minikube using Docker Compose, creating a persistent volume and claim, setting up MySQL with secrets, and deploying WordPress. 

---

## Step 1. Install Minikube

- Go to https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download.
- Choose your Operating System, and you will see the commands to install the binary on your machine.

![Minikube Binary Install](images/minikube-binary-install.png)

- Create your project directory:
```bash
mkdir mysql-wordpress-kubernetes
cd mysql-wordpress-kubernetes/
```
- Then, add directories and files to make your project structure look like this:
```txt
.
├── README.md
├── images
│   └── minikube-binary-install.png
├── mysql
│   ├── mysql-deployment.yaml
│   ├── mysql-secret.yaml
│   └── mysql-service.yaml
├── storage
│   ├── pv.yaml
│   └── pvc.yaml
└── wordpress
    ├── wordpress-deployment.yaml
    └── wordpress-service.yaml
```

## Step 2. Create a Persistent Volume and Persistent Volume Claim

- Access the Minikube VM to create the /mnt/data/mysql
```bash
minikube ssh
sudo mkdir -p /mnt/data/mysql
sudo chmod 777 /mnt/data/mysql
ls -ld /mnt/data/mysql #To verify. 
exit  #: Log out of the Minikube VM and return to the CLI (local machine).
```
- The -p flag ensures that the parent directories are created if they don’t already exist.
- The chmod 777 command sets full read, write, and execute permissions for all users, which is helpful for testing but can be restricted later.

Persistent Volumes (PV) provide storage for your Kubernetes applications, while Persistent Volume Claims (PVC) allow applications to request specific storage resources dynamically.

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
    path: "/mnt/data/mysql"
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
kubectl apply -f storage/pv.yaml
kubectl apply -f storage/pvc.yaml
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
  username: bXlzcWw=   # Base64 encoded 'mysql'
  password: cGFzc3dvcmQ= # Base64 encoded 'password'
```

**Apply the secret**:
```bash
kubectl apply -f mysql/mysql-secret.yaml
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
        ports:
        - containerPort: 3306
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
kubectl apply -f mysql/mysql-deployment.yaml
kubectl apply -f mysql/mysql-service.yaml
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
        image: wordpress:php8.0-apache
        ports:
        - containerPort: 80
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
kubectl apply -f wordpress/wordpress-deployment.yaml
kubectl apply -f wordpress/wordpress-service.yaml
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

**Stop and remove Minikube as desired**:
```bash
minikuke stop  #To stop Minikube
minikube delete # To delete Minikube
```
**To clean up Minikube files**
```bash
rm -rf ~/.minikube
rm -rf ~/.kube
```

**To remove minikube and its binary**
```bash
sudo rm -f /usr/local/bin/minikube
brew uninstall minikube  # For MacOS. 
minikube version   # To verify removal.
```

**Remove data directories**:
```bash
rm -rf /data/mysql
rm -rf /var/lib/minikube
```

**Remove the project directory**:
```bash
rm -rf /mysql-wordpress-kubernetes
```

---

This setup provides a fully functional WordPress application backed by MySQL with persistent storage and secret management.
