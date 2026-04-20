# Deployment Guide for Login Web Application on Kubernetes

This guide provides step-by-step instructions for deploying the login web application with MySQL database on Kubernetes. All configuration files are already included in the repository.

## 1. Clean Up Previous Deployments

First, clean up any previous deployments related to this project:

```bash
# Delete deployments
kubectl delete deployment login-app mysql

# Delete services
kubectl delete service login-app mysql

# Delete PVCs and PVs
kubectl delete pvc mysql-pvc
kubectl delete pv mysql-pv

# Delete secrets
kubectl delete secret mysql-secret
```
![image](https://hackmd.io/_uploads/Sk8YsX1pbl.png)

## 2. Build and Load Docker Image

```bash
 git clone https://github.com/Widhi-yahya/kubernetes_installation_docker.git
```
    
![image](https://hackmd.io/_uploads/rJ-12X16-l.png)
    
    
Navigate to the app directory and build the Docker image:

```bash
# Navigate to app directory
cd k8s-login-app/app

# Build Docker image
docker build -t login-app:latest .

# Save Docker image as TAR file for distribution to worker nodes
docker save login-app:latest > login-app.tar
```
    
![image](https://hackmd.io/_uploads/HkBn2Xk6Zl.png)


If your worker nodes don't share the Docker registry with your control plane, transfer and load the image on all nodes:

```bash
# Transfer the image to worker nodes (replace with actual node IPs)
scp login-app.tar user@worker-node:/home/user/

# On each worker node, load the image
docker load < login-app.tar
```
                
![Screenshot 2026-04-17 104716](https://hackmd.io/_uploads/S1kKaXypbx.png)
                           
![image](https://hackmd.io/_uploads/S1Kn6m1p-g.png)


    
## 3. Prepare Storage for MySQL

Create a directory on your worker node to store MySQL data:

```bash
# Create a directory on your worker node for MySQL data (execute on worker node)
sudo mkdir -p /mnt/data
sudo chmod 777 /mnt/data
```
![image](https://hackmd.io/_uploads/HynbR716Zl.png)


## 4. Deploy MySQL Database

Apply the MySQL configurations:

```bash
# Apply MySQL configurations
kubectl apply -f k8s/mysql-secret.yaml
kubectl apply -f k8s/mysql-pv.yaml
kubectl apply -f k8s/mysql-pvc.yaml
kubectl apply -f k8s/mysql-service.yaml
kubectl apply -f k8s/mysql-deployment.yaml

# Check if MySQL pod is running
kubectl get pods -l app=mysql

# Wait for MySQL pod to be ready
kubectl wait --for=condition=ready pod -l app=mysql --timeout=180s
```

![image](https://hackmd.io/_uploads/H1Ss0mkTWe.png)
![image](https://hackmd.io/_uploads/ByT6R7kT-g.png)

                           
## 5. Deploy Web Application

Deploy the web application after MySQL is running:

```bash
# Apply web application configurations
kubectl apply -f k8s/web-deployment.yaml
kubectl apply -f k8s/web-service.yaml

# Check if web application pods are running
kubectl get pods -l app=login-app
```
![image](https://hackmd.io/_uploads/By8gkVJaZg.png)
![image](https://hackmd.io/_uploads/SyYjB416-x.png)

## 6. Access the Application

The application is exposed through a NodePort service on port 30080. You can access it from either node:

```
http://10.34.7.115:30080  (Master node)
http://10.34.7.5:30080    (Worker node)
```

Both URLs work from any machine on your local network (10.34.7.0/24).

## 7. Testing the Application

1. Open your web browser and navigate to `http://10.34.7.115:30080`
2. Register a new user or use the default credentials:
   - Username: `admin`
   - Password: `admin123`
3. After login, you'll be redirected to the dashboard where you can upload images

![image](https://hackmd.io/_uploads/B1i6rVkaZx.png)

    
## 8. Important: Calico Networking Configuration

The cluster uses Calico CNI with VXLAN. If you experience networking issues between nodes, ensure Calico is configured to detect the correct network interface:

```bash
# Fix Calico IP detection to use the correct interface
kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=can-reach=10.34.7.115

# Restart calico-node pods to apply changes
kubectl delete pod -n calico-system -l k8s-app=calico-node

# Wait for calico-node pods to be ready
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n calico-system --timeout=180s

# Verify correct IPs are detected on all nodes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.projectcalico\.org/IPv4Address}{"\n"}{end}'
```
![image](https://hackmd.io/_uploads/r17K8EJTbg.png)

## 9. Troubleshooting

If you encounter issues:

```bash
# Check pod status
kubectl get pods

# Check MySQL logs
kubectl logs -l app=mysql

# Check web application logs
kubectl logs -l app=login-app

# Check MySQL connectivity from web app
kubectl exec -it $(kubectl get pod -l app=login-app -o jsonpath='{.items[0].metadata.name}') -- sh -c 'nc -zv mysql 3306'

# Check service configuration
kubectl get svc

# If database connection fails, restart login-app deployment
kubectl rollout restart deployment login-app

# Test DNS resolution from login-app pod
kubectl exec -it $(kubectl get pods -l app=login-app -o name | head -1) -- nslookup mysql

# Check Calico networking status
kubectl get pods -n calico-system
```

## 10. Database Management

To manually manage the database:

```bash
# Connect to MySQL
kubectl exec -it $(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') -- mysql -u root -p

# Enter password: Otomasi-13

# Then run MySQL commands
USE loginapp;
SHOW TABLES;
SELECT * FROM users;
```
    
![image](https://hackmd.io/_uploads/BJDTI4kT-x.png)


## 11. Accessing the Application

### Login Application
- **URL**: http://10.34.7.115:30080 or http://10.34.7.5:30080
- **Default Credentials**: 
  - Username: `admin`
  - Password: `admin123`
![image](https://hackmd.io/_uploads/ry_GPV1p-e.png)

### Kubernetes Dashboard
- **URL**: https://10.34.7.115:30119 or https://10.34.7.5:30119
- **Access Token**: Generate with:
  ```bash
  kubectl create token widhi -n kube-system --duration=24h
  ```
![image](https://hackmd.io/_uploads/B1uJnNJpZe.png)

![image](https://hackmd.io/_uploads/Hy1Rs4kpWl.png)

This completes the deployment of the login web application with MySQL on Kubernetes.
```
```    
### **REPLICA**
![image](https://hackmd.io/_uploads/B1GunE1pWg.png)
![image](https://hackmd.io/_uploads/HyHk64Jp-l.png)
![image](https://hackmd.io/_uploads/H19jTEk6Wg.png)