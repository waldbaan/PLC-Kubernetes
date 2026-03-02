#!/bin/bash
# source: https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99
# this shell script installs a worker node. 
# after running this script the command "sudo kubeadm join ...." as as shown on the master at the end of the install
# must be executed.


# Update the package index
sudo apt-get update
sudo apt upgrade

# load required kernel modules on all nodes
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# configure critical  kernel parameters for kubernetes
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# reload changes
sudo sysctl --system

# Install required packages
sudo apt-get install -y apt-transport-https ca-certificates curl

# Install dependencies for containerd
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# enable docker repository
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# update package list en install containerd
sudo apt update
sudo apt install -y containerd.io

# configure containerd to start using systemd as cgroup
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd


# Download the Google Cloud public signing key
sudo curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Add the Kubernetes APT repository  New repository k8s from 2024!!
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt-get update

# Update the package index again
sudo apt-get update

# Install kubelet, kubeadm, and kubectl
sudo apt-get install -y kubelet kubeadm kubectl

# Mark the packages to hold them at their current version
sudo apt-mark hold kubelet kubeadm kubectl

# Disable swap memory (Kubernetes requires swap to be disabled)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable and start kubelet service
sudo systemctl enable kubelet
sudo systemctl start kubelet

# Initialize the Kubernetes cluster using kubeadm (Master Node)
# sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Set up local kubeconfig (master)
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Apply a Pod network add-on (using Calico in this example)
# kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

# Print instructions for joining worker nodes
# kubeadm token create --print-join-command