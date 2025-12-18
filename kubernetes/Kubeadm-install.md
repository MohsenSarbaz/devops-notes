# Building a Kubernetes v1.31 Cluster from Scratch on Ubuntu 24.04

This guide documents how I installed a **3-node Kubernetes cluster** using **kubeadm** and **containerd** on **Ubuntu 24.04**.
## Environment
- OS: Ubuntu 24.04
- Kubernetes: v1.31
- Runtime: containerd
- Cluster type: kubeadm
- Nodes:
  - `k8s1-m` – Control Plane
  - `k8s2-w` – Worker
  - `k8s3-w` – Worker
## Prerequisites
- Static IP addresses
- Internet access
- Root or sudo access
- Swap disabled
- Time synchronization enabled

## Step 0: Set Hostnames and `/etc/hosts`
### Set hostname (run on each node)
```
hostnamectl set-hostname k8s1-m   # control-plane (master)
hostnamectl set-hostname k8s2-w   # worker
hostnamectl set-hostname k8s3-w   # worker
```

 ### Update `/etc/hosts` (on all nodes)
 ```
sudo tee -a /etc/hosts <<EOF
192.168.0.111 k8s1-m
192.168.0.112 k8s2-w
192.168.0.113 k8s3-w
EOF
 ```

## Step 1: Disable Swap 
```
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
free -h #shows swap is off
```

## Step 2: Install containerd Runtime
### Add Docker repository
```
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo tee /etc/apt/keyrings/docker.asc > /dev/null

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list
```
### Install containerd
```
sudo apt update
sudo apt install -y containerd.io
containerd --version #verify
```

## Step 3: Configure containerd (Systemd Cgroup)
Create the default config:
`sudo containerd config default | sudo tee /etc/containerd/config.toml
`
Enable systemd cgroup:
Locate the line that reads `SystemdCgroup = false` under the `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]` section. Change `false` to `true`. It should look like this:
`sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml`
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Step 4: Load Required Kernel Modules
`br_netfilter` and `overlay` modules are needed to manage network traffic and to support network overlays respectively.
```
sudo modprobe overlay br_netfilter
```
make them persist:
```
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

## Step 6: Install Kubernetes Components (v1.31)
### Add Kubernetes repository
```
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```
### Install packages
```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
kubeadm version
kubectl version --client
```
> kubelet not running yet is normal.


## Step 7: Initialize the Control Plane
### Using Flannel

```
kubeadm init \
--apiserver-advertise-address=192.168.0.111 \
--pod-network-cidr=10.244.0.0/16
```
### Using Calico (alternative)
```
kubeadm init \
--apiserver-advertise-address=192.168.0.111 \
--pod-network-cidr=192.168.0.0/16
```

## Step 8: Configure kubectl Access
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 9: Install CNI Plugin

### Flannel
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Calico
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
kubectl get pods -A
```

## Step 10: Join Worker Nodes

Run on each worker node:
```
kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

---

## References

- [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
    
- [https://kubernetes.io/docs/concepts/cluster-administration/addons/](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
