
# Kubernetes Cluster Setup on Linode

## Step 1: Configure Networking

### 1.1 Update Hostnames and Hosts File

#### 1.1.1 Set Hostnames

On each node, SSH into the instance:

```bash
ssh root@<public_ip>
```

Set the hostname:

```bash
hostnamectl set-hostname k8s-master  # For master node
hostnamectl set-hostname k8s-worker1  # For worker 1
hostnamectl set-hostname k8s-worker2  # For worker 2
```

#### 1.1.2 Update /etc/hosts

Add the private IPs and hostnames to `/etc/hosts` on each node:

```bash
nano /etc/hosts
```

Add the following lines (replace with your actual private IPs):

```plaintext
10.0.0.1 k8s-master
10.0.0.2 k8s-worker1
10.0.0.3 k8s-worker2
```

### 1.2 Disable Swap

Kubernetes requires swap to be disabled:

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#/g' /etc/fstab
```

## Step 2: Install Docker and Kubernetes Components

### 2.1 Install Container Runtime (containerd)

Perform these steps on all nodes.

#### 2.1.1 Install Dependencies

```bash
apt update
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

#### 2.1.2 Add Docker’s GPG Key

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

#### 2.1.3 Add Docker Repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### 2.1.4 Install containerd

```bash
apt update
apt install -y containerd.io
```

#### 2.1.5 Configure containerd

```bash
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

#### 2.1.6 Restart containerd

```bash
systemctl restart containerd
systemctl enable containerd
```

### 2.2 Install Kubernetes Components

#### 2.2.1 Add Kubernetes GPG Key

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

#### 2.2.2 Add Kubernetes Repository

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

#### 2.2.3 Install kubeadm, kubelet, and kubectl

```bash
apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
#### 2.2.4 Install socat and enable ipv4 ip forwarding

```bash
sudo apt install socat
sudo sh -c "echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf"
sudo sysctl -p
```

## Step 3: Initialize the Kubernetes Master Node

### 3.1 Initialize Kubernetes

On the master node (k8s-master), run:

```bash
kubeadm init --apiserver-advertise-address=10.0.0.1 --pod-network-cidr=192.168.0.0/16
```

Replace `10.0.0.1` with the master node's private IP.

### 3.2 Configure kubectl for the Root User

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

Or for a non-root user:

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3.3 Save the Join Command

Copy the `kubeadm join` command displayed after initialization. It will look like:

```bash
kubeadm join 10.0.0.1:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## Step 4: Install a Pod Network Add-on

### 4.1 Install Calico

On the master node:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 4.2 Verify the Installation

Check that all pods are running:

```bash
kubectl get pods --all-namespaces
```

## Step 5: Join Worker Nodes to the Cluster

### 5.1 On Each Worker Node

Run the `kubeadm join` command you saved earlier:

```bash
kubeadm join 10.0.0.1:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 5.2 Verify Nodes in the Cluster

On the master node:

```bash
kubectl get nodes
```

You should see all three nodes with the status `Ready`.

## Step 6: Deploy Your Application

### 6.1 Create a Deployment

Create a file named `my-app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: your-docker-image  # Replace with your image
        ports:
        - containerPort: 80
        env:
        - name: DATABASE_HOST
          value: "10.0.0.4"  # HAProxy server private IP
        - name: DATABASE_PORT
          value: "3306"
```

### 6.2 Create a Service

Create a file named `my-app-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007  # Access via this port on any node
```

### 6.3 Deploy the Application

Apply the configurations:

```bash
kubectl apply -f my-app-deployment.yaml
kubectl apply -f my-app-service.yaml
```

### 6.4 Verify the Deployment

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

### 6.5 Test Connectivity to HAProxy

On any pod:

```bash
kubectl exec -it <pod-name> -- /bin/bash
ping 10.0.0.4  # HAProxy server private IP
```

Ensure that your application can connect to the database via HAProxy.

## Step 7: Scale Your Cluster

### 7.1 Scaling Pods

To scale your application up or down:

```bash
kubectl scale deployment my-app --replicas=5
```

### 7.2 Adding Worker Nodes

#### 7.2.1 Create a New Linode Instance

Follow the same steps as before to create a new worker node.
Ensure it is attached to the `k8s-vpc`.

#### 7.2.2 Install Dependencies

Repeat Step 3 on the new node.

#### 7.2.3 Join the Cluster

Use the `kubeadm join` command to add the node.

### 7.3 Removing Worker Nodes

#### 7.3.1 Drain the Node

On the master node:

```bash
kubectl drain k8s-worker2 --delete-local-data --force --ignore-daemonsets
```

#### 7.3.2 Delete the Node

```bash
kubectl delete node k8s-worker2
```

#### 7.3.3 Reset `kubeadm` on the Worker

On the worker node:

```bash
kubeadm reset -f
```

#### 7.3.4 Delete the Linode Instance

Remove the instance from the Linode Cloud Manager.

---
