# Kubernetes Setup Guide

## 1. Common for all Nodes

### Step 1 - Disable swap

Disable swap immediately and persistently by removing it from `fstab`:
```bash
# Disable swap for the current session
sudo swapoff -a

# Remove swap configuration from fstab to persist across reboots
sudo sed -i '/swap/d' /etc/fstab
```

Verify that swap is successfully disabled:
```bash
# Look for Swap row should show 0B for total, used, and free
free -h
```

### Step 2 - Enable iptables Bridged Traffic on all the Nodes

Configure necessary kernel modules for Kubernetes networking:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Set sysctl parameters required by setup, ensuring they persist across reboots:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Apply the sysctl parameters without a reboot
sudo sysctl --system
```

Verify the configuration is correctly applied:
```bash
# Verify modules are loaded
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify sysctl parameters are applied
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

Update packages and reboot the system to apply all changes:
```bash
sudo apt update
sudo apt upgrade -y

# Reboot system and re-login
sudo reboot
```

### Step 3 - Install Container Runtime
*Reference: [Docker official documentation](https://docs.docker.com/engine/install/ubuntu/)*

Install Docker's official GPG key and add the repository:
```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Install the container runtime (containerd):
```bash
sudo apt update
sudo apt install containerd.io
```

Configure containerd to use the `systemd` cgroup driver:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Enable and restart the containerd service:
```bash
sudo systemctl enable containerd
sudo systemctl restart containerd

# Verify the service is running
sudo systemctl status containerd
```

### Step 4 - Installing kubeadm, kubelet and kubectl
*Reference: [Kubernetes official documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)*

Download the Kubernetes package repository key and add the repository:
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install Kubernetes components and prevent them from being automatically updated:
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable and start the kubelet service:
```bash
sudo systemctl enable --now kubelet
sudo systemctl restart kubelet

# Verify the service is running
systemctl status kubelet
```

Verify the installation and versions:
```bash
# Verify installation and version
kubeadm version
kubelet --version
kubectl version --client
```

## 2. Only For Master Node
*Reference: [Kubernetes official documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)*

### Step 5 - Initialize the Control Plane

Initialize the cluster on the master node. Ensure you replace `192.168.1.x` with your master node's actual IP:
```bash
sudo kubeadm init --apiserver-advertise-address=192.168.1.x --pod-network-cidr=192.168.0.0/16
```

Set up local kubeconfig for your administrative user:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
*Note: Save the `kubeadm join` command printed in the terminal to join worker nodes.*

Verify the cluster status:
```bash
# Check if the node registered
kubectl get nodes

# Check if the core Kubernetes pods are running
kubectl get pods -n kube-system
```

### Step 6 - Install Calico CNI
*Reference: [Calico official documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)*

Install the Tigera Calico operator and custom resource definitions:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
```

Download and apply the custom resources manifest to configure Calico:
```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml
kubectl create -f custom-resources.yaml
```

Monitor the installation and verify the pods are running:
```bash
watch kubectl get tigerastatus

# Verify Calico pods are running
kubectl get pods -n calico-system
```

## 3. Join Worker Nodes

Run the join command provided by `kubeadm init` on each worker node:
```bash
sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

## 4. Trouble shooting

If you missed or lost the join token, recreate it on the master node:
```bash
# Create join token if you missed
kubeadm token create --print-join-command
```

Verify your cluster status on the master node:
```bash
# Verify your cluster on master node
kubectl get pods -A
kubectl get nodes -o wide
```

If a worker fails to join the cluster, reset it and clean up CNI configurations before retrying:
```bash
# If worker fail to join cluster follow k8s docs
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo systemctl restart containerd
```

## Setup Metrics Server
*Reference: [Metrics Server official documentation](https://artifacthub.io/packages/helm/metrics-server/metrics-server)*

Install Helm:
```bash
sudo apt update
sudo snap install helm --classic
helm version
```

Add the metrics-server repository and install it:
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

# Install metrics-server allowing insecure TLS for internal communication
helm upgrade --install metrics-server metrics-server/metrics-server \
 --namespace kube-system \
 --set args={--cert-dir=/tmp,--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP}
```

Alternatively, install using a custom values file:
```bash
# Or using custom values
helm upgrade --install metrics-server metrics-server/metrics-server \
 --namespace kube-system \
 -f custom-values.yaml
```

Verify the installation:
```bash
# Verify
kubectl get pods -n kube-system | grep metrics-server
```
