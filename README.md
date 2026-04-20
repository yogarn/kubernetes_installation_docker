# Kubernetes Installation on Ubuntu 22.04/24.04
Group Members:
1. Yoga Raditya Nala (235150201111020)
2. Vincentia Melody Vivianne (235150201111047)
3. Izzah Faiq Putri Madani (235150201111039)


Get the detailed information about the installation from the below-mentioned websites of **Docker** and **Kubernetes**.

[Docker](https://docs.docker.com/)

[Kubernetes](https://kubernetes.io/)

### Set up the Docker and Kubernetes repositories:

### Requirements
1. Ubuntu machines as Master and Worker ( build on VM using Bridge Adapter) minimal 2 machines: 1 Master and 1 Worker
2. Networking uses local network (FILKOM)



> Download the GPG key for docker

```bash
wget -O - https://download.docker.com/linux/ubuntu/gpg > ./docker.key

gpg --no-default-keyring --keyring ./docker.gpg --import ./docker.key

gpg --no-default-keyring --keyring ./docker.gpg --export > ./docker-archive-keyring.gpg

sudo mv ./docker-archive-keyring.gpg /etc/apt/trusted.gpg.d/
```

![image](https://hackmd.io/_uploads/r14F47Chbe.png)
![image](https://hackmd.io/_uploads/ry_o4mC3be.png)

> Add the docker repository and install docker

```bash
# we can get the latest release versions from https://docs.docker.com

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" -y
sudo apt update -y
sudo apt install git wget curl socat -y
sudo apt install -y docker-ce

```

![image](https://hackmd.io/_uploads/Sk8eBQChWx.png)
![image](https://hackmd.io/_uploads/r1ieSQ0nWx.png)

**To install cri-dockerd for Docker support**

**Docker Engine does not implement the CRI which is a requirement for a container runtime to work with Kubernetes. For that reason, an additional service cri-dockerd has to be installed. cri-dockerd is a project based on the legacy built-in Docker Engine support that was removed from the kubelet in version 1.24.**

> Get the version details

```bash
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
```

> Run below commands

```bash

wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz

tar xzvf cri-dockerd-${VER}.amd64.tgz

sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket

sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/

sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket

```

![image](https://hackmd.io/_uploads/Skp-LmC3Wg.png)
![image](https://hackmd.io/_uploads/BkV9LXAn-g.png)

> Add the GPG key for kubernetes

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

> Add the kubernetes repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

> Update the repository

```bash
# Update the repositiries
sudo apt-get update
```

![image](https://hackmd.io/_uploads/BJFhL7AnZg.png)
![image](https://hackmd.io/_uploads/BkzALm03Zg.png)

> Install  Kubernetes packages.

```bash
# Use the same versions to avoid issues with the installation.
sudo apt-get install -y kubelet kubeadm kubectl
```

![image](https://hackmd.io/_uploads/rkfgwQR3bl.png)
![image](https://hackmd.io/_uploads/Hy8DPQCnbl.png)

> To hold the versions so that the versions will not get accidently upgraded.

```bash
sudo apt-mark hold docker-ce kubelet kubeadm kubectl
```

> Enable the iptables bridge

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

![image](https://hackmd.io/_uploads/rJxcP702bl.png)
![image](https://hackmd.io/_uploads/rJxcP702bl.png)

### Disable SWAP
> Disable swap on controlplane and dataplane nodes

```bash
sudo swapoff -a
```

```bash
sudo vim /etc/fstab
# comment the line which starts with **swap.img**.
```

![image](https://hackmd.io/_uploads/S1Zu_QC3Ze.png)
![image](https://hackmd.io/_uploads/HkZEOX0nWg.png)

### On the Control Plane server (Master node)

> Initialize the cluster by passing the cidr value and the value will depend on the type of network CLI you choose.

**Calico**

```bash
# Calico network
# Make sure to copy the join command
sudo kubeadm init \
  --apiserver-advertise-address=192.168.122.213 \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --pod-network-cidr=10.244.0.0/16

# Copy your join command and keep it safe.
# Below is a sample format
# Add --cri-socket /var/run/cri-dockerd.sock to the command
kubeadm join 192.168.122.213:6443 --token ssn143.ie9xv91h4i6hqc9k --cri-socket unix:///var/run/cri-dockerd.sock --discovery-token-ca-cert-hash sha256:46d035479a2c8e558ba0ba04fa1c8415470da4141cf3e129b45b2177e8d0cd6b 
```

![image](https://hackmd.io/_uploads/H1jP2gJ6Ze.png)
![image](https://hackmd.io/_uploads/rJprax1aZg.png)

> To start using the cluster with current user.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

![image](https://hackmd.io/_uploads/SJ0u6eJa-e.png)

> To set up the Calico network

```bash
# Use this if you have initialised the cluster with Calico network add on.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml -O

# Change the ip to 10.244.0.0/16 if the node network is 192.168.x.x
kubectl create -f custom-resources.yaml
```

![image](https://hackmd.io/_uploads/SJEjTgyabl.png)
![image](https://hackmd.io/_uploads/Sy6A6x1pbx.png)
![image](https://hackmd.io/_uploads/SyVlRxJpbl.png)


> Check the nodes

```bash
# Check the status on the master node.
kubectl get nodes
```

![image](https://hackmd.io/_uploads/rkn07ZyTZl.png)

### On each of Data plane node (Worker node)

> Joining the node to the cluster:

> Don't forget to include *--cri-socket unix:///var/run/cri-dockerd.sock* with the join command

```bash
sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash
#Ex:
# kubeadm join <control_plane_ip>:6443 --cri-socket unix:///var/run/cri-dockerd.sock --token 31rvbl.znk703hbelja7qbx --discovery-token-ca-cert-hash sha256:3dd5f401d1c86be4axxxxxxxxxx61ce965f5xxxxxxxxxxf16cb29a89b96c97dd
# sudo kubeadm join 10.34.7.115:6443 --cri-socket unix:///var/run/cri-dockerd.sock --token kwdszg.aze47y44h7j74x6t --discovery-token-ca-cert-hash sha256:3bd51b39b3a166a4ba5914fc3a19b61cfe81789965da6ac23435edb6aeed9e0d
```

![image](https://hackmd.io/_uploads/rJprax1aZg.png)

**TIP**

> If the joining code is lost, it can retrieve using below command

```bash
kubeadm token create --print-join-command
```

### To install metrics server (Master node)

```bash
git clone https://github.com/mialeevs/kubernetes_installation_docker.git
cd kubernetes_installation_docker/
kubectl apply -f metrics-server.yaml
cd
rm -rf kubernetes_installation_docker/
```

![image](https://hackmd.io/_uploads/B15BS7Ja-l.png)

### Installing Dashboard (Master node)

1. *Installing Helm:*
Download and install Helm with the following commands:
```bash
     curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
     chmod +x get_helm.sh
     ./get_helm.sh
     helm   
```

![image](https://hackmd.io/_uploads/BJ9KHQy6Ze.png)

3. *Adding the Kubernetes Dashboard Helm Repository:*
Add the repository and verify it:
```bash   
     helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
     helm repo list    
```

![image](https://hackmd.io/_uploads/Hyw3wQk6We.png)

5. *Installing Kubernetes Dashboard Using Helm:*
Install it in the `kubernetes-dashboard` namespace:
```bash     
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
kubectl get pods,svc -n kubernetes-dashboard  
```

![image](https://hackmd.io/_uploads/B1eKbu71TWx.png)

7. *Accessing the Dashboard:*
Expose the dashboard using a NodePort:
```bash      
kubectl expose deployment kubernetes-dashboard-kong --name k8s-dash-svc --type NodePort --port 443 --target-port 8443 -n kubernetes-dashboard
```
run: 
```
kubectl get pods,svc -n kubernetes-dashboard
```
use this port to access the dashboard from phy node IP: 
....
service/k8s-dash-svc                           NodePort    10.110.85.135   <none>        443:30346/TCP   23s

![image](https://hackmd.io/_uploads/SJpadQypZl.png)

9. *Generating a Token for Login:*
Create a service account and generate a token:
```bash
nano k8s-dash.yaml
```

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: widhi
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: widhi-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: widhi
  namespace: kube-system
```
then run:
```bash
kubectl apply -f k8s-dash.yaml
```

![image](https://hackmd.io/_uploads/ryxyqQJ6Wx.png)
![image](https://hackmd.io/_uploads/SkOWc7kpWl.png)

10. Generate the token:    
     kubectl create token widhi -n kube-system

![image](https://hackmd.io/_uploads/rkmEcmyaWe.png)