This Git repository contains supporting files for my "Certified Kubernetes Administrator (CKA)" video course. See https://sandervanvugt.com for more details. It is also used in the "CKA Crash Course" that I'm teaching at https://learning.oreilly.com. 

In this course you need to have your own lab environment. This lab environment should consist of 3 virtual machines, using Ubuntu LTS server 20.4 or later (22.4 is recommended)
Make sure the virtual machines meet the following requirements
*	2GB RAM
*	2 vCPUs
*	20 GB disk space
*	No swap
For instructions on how to set up Ubuntu Server 22.04, see the document "Installing Ubuntu 22-04" in this Git repository.
For information on getting started with VirtualBox, see this video: https://www.youtube.com/watch?v=4qwUHSaIJdY
Alternatively, check out my video course "Virtualization for Everyone" for an introduction to different virtualization solution. 

To set up the required tools on the cluster nodes, the following scripts are provided:
*	setup-container.sh installs containerd. Run this script first
*	setup-kubetools.sh install the latest version of kubelet, kubeadm and kubectl
*	setup-kubetool-previousversion.sh installs the previous major version of the kubelet, kubeadm and kubectl. Use this if you want to practice cluster upgrades

## My personal notes on install on Ubuntu 24.04 with cilium kube-proxy replacement

i did my install on an Ubuntu 24.04 LTR

### 1. System Update and Upgrade

```bash
  apt update && sudo apt upgrade -y
  reboot
```
  
### 2. Disable Swap
Kubernetes officially requires swap to be disabled for optimal performance and stability.

```bash
  swapoff -a
  sed -i '/ swap / s/^/#/' /etc/fstab

### 3. Install and configure  a Container Runtime : containerd
Kubernetes uses a container runtime to run containers. Containerd is the recommended runtime.

```bash  
  apt install -y containerd
```

##### 3.b) Generate a default containerd configuration file:

```bash
  mkdir -p /etc/containerd
  containerd config default | sudo tee /etc/containerd/config.toml
```
##### 3.c) Enable systemd as the cgroup driver for containerd. This is crucial for compatibility with kubelet.
```bash
  sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```
##### 3.d) Restart and Enable Containerd:
```bash
  systemctl restart containerd
  systemctl status containerd.service 
  systemctl enable containerd
```
### 4. Configure Kernel Modules and Sysctl Parameters  
Kubernetes needs specific kernel modules and sysctl parameters to be enabled for networking.

##### 4.a) Load Kernel Modules:
```bash  
  modprobe overlay
  modprobe br_netfilter
  tee /etc/modules-load.d/k8s.conf <<EOF
  overlay
  br_netfilter
  EOF
```  
##### 4.b) Configure Sysctl Parameters:
Create a Kubernetes configuration file for sysctl:
 ```bash 
  tee /etc/sysctl.d/k8s.conf <<EOF
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward   = 1
  EOF
```
Apply the changes:
```bash
  sysctl --system
```
### 5. Install kubelet, kubeadm, and kubectl
These are the core Kubernetes components:

 + kubeadm: The tool to bootstrap your cluster.￼

 + kubelet: The agent that runs on each node in the cluster and ensures containers are running in pods.

 + kubectl: The command-line utility for interacting with your Kubernetes cluster.

Add Kubernetes APT Repository:
```bash
  apt install -y apt-transport-https ca-certificates curl gpg
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/  /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   apt update 
   apt install -y kubelet kubeadm kubectl
   apt-mark hold kubelet kubeadm kubectl
```
### 6. kubeadm init 
If you want to use Cilium’s kube-proxy replacement, kubeadm needs to skip the kube-proxy deployment phase, so it has to be executed with the --skip-phases=addon/kube-proxy option:
https://docs.cilium.io/en/latest/installation/k8s-install-kubeadm/#installation-using-kubeadm
```bash
  kubeadm init --skip-phases=addon/kube-proxy
  systemctl status kubelet
  journalctl -xeu kubelet

  #in case of succes
  mkdir -p $HOME/.kube
  cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config
  kubectl get nodes --output=wide

```

### in case you got an error with kubeadm init reset all before trying again
```bash
kubeadm reset -f
systemctl stop kubelet
systemctl stop containerd
rm -rf /etc/cni/net.d
rm -rf /var/lib/kubelet/*
rm -rf /var/lib/containerd/*
systemctl daemon-reload
sudo systemctl start containerd
```

### test your kubernetes node
```bash
  mkdir -p $HOME/.kube
  cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config
  kubectl get nodes --output=wide
```

### install helm and cilium

https://docs.cilium.io/en/latest/installation/k8s-install-kubeadm/#installation-using-kubeadm

```bash
  wget https://get.helm.sh/helm-v3.18.2-linux-amd64.tar.gz
  tar xvfz helm-v3.18.2-linux-amd64.tar.gz 
  rsync -av linux-amd64/helm  /usr/local/bin/
  curl -LO https://github.com/cilium/cilium/archive/main.tar.gz
  tar xzf main.tar.gz
  cd cilium-main/install/kubernetes/
  helm install cilium ./cilium --namespace kube-system
  CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
  CLI_ARCH=amd64
  if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
  curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
  sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
  sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
  rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
  cd
  git clone https://github.com/cilium/cilium.git
  cd cilium
  cilium status --wait
  kubectl get nodes --output=wide
```

##### you should get something like this : 
```bash
  NAME                      STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
  ovh-k8s-master-2025       Ready    control-plane   21m   v1.33.1   51.89.98.176   <none>        Ubuntu 22.04.5 LTS   5.15.0-141-generic   containerd://1.7.27
```

