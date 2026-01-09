# Kubernetes-Production-Cluster-on-AWS-EC2



https://github.com/user-attachments/assets/7e4037e1-c333-4500-9e5f-ae82092df688



## Overview
This project shows how to build a real Kubernetes cluster from zero on AWS EC2.
The goal is to understand how Kubernetes works internally by setting up everything manually, the same way platform and DevOps teams do in real companies.
This is not a demo cluster. This is a real, working Kubernetes setup.

---

## What This Project Builds

* One Kubernetes control plane node
* Two Kubernetes worker nodes
* Container runtime installation
* Kubernetes components installation
* Pod networking using Calico
* A running application exposed using a service

---

## Step 1: Create EC2 Instances

Create three EC2 instances from the AWS Console.

Use these settings:

* Ubuntu Server 22.04 LTS
* Instance type: t2.medium
* Security group:

  * Allow SSH (port 22) from your IP
  * Allow all traffic inside the same security group

Name the instances clearly:

* k8s-master
* k8s-worker-1
* k8s-worker-2

---

## Step 2: Connect to Instances

SSH into each instance.

```bash
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP
```

Run all setup steps on all three nodes unless mentioned otherwise.

---

## Step 3: System Preparation

Update the system.

```bash
sudo apt update && sudo apt upgrade -y
```

Disable swap.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Load required kernel modules.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Apply system settings.

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

---

## Step 4: Install Container Runtime (containerd)

```bash
sudo apt install -y containerd
```

Configure containerd.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart containerd.

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Step 5: Install Kubernetes Tools

```bash
sudo apt install -y apt-transport-https ca-certificates curl
```

Add Kubernetes repository.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install Kubernetes components.

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Step 6: Initialize Control Plane (Master Only)

Run only on the master node.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubectl access.

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check nodes.

```bash
kubectl get nodes
```

---

## Step 7: Install Pod Network (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait until system pods are running.

```bash
kubectl get pods -n kube-system
```

---

## Step 8: Join Worker Nodes

Run the join command shown after kubeadm init on each worker node.

```bash
sudo kubeadm join MASTER_IP:6443 --token TOKEN \
--discovery-token-ca-cert-hash sha256:HASH
```

Verify cluster status from the master.

```bash
kubectl get nodes
```

---

## Step 9: Create Namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
```

---

## Step 10: Deploy Application

Create a deployment.

```bash
kubectl create deployment nginx --image=nginx -n dev
```

Check pods.

```bash
kubectl get pods -n dev
```

Expose the deployment.

```bash
kubectl expose deployment nginx --type=NodePort --port=80 -n dev
```

Get service details.

```bash
kubectl get svc -n dev
```

Access the application using any worker node public IP and the NodePort.

---
