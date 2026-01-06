## Install and Create  cluster using kubeadm



https://v1-31.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

After vms for master, worker nodes have been created : 

# Check network adapters
If you have more than one network adapter, and your Kubernetes components are not reachable on the default route, we recommend you add IP route(s) so Kubernetes cluster addresses go via the appropriate adapter.
Check required ports
These required ports need to be open in order for Kubernetes components to communicate with each other. You can use tools like netcat to check if a port is open. For example:
nc 127.0.0.1 6443 -v
To disable swap, sudo swapoff -a can be used to disable swapping temporarily. To make this change persistent across reboots, make sure swap is disabled in config files like /etc/fstab, systemd.swap, depending how it was configured on your system. Comment out on vim /etc/fstab/ 

## Container Runtimes preq

You need to install a container runtime into each node in the cluster so that Pods can run there. This page outlines what is involved and describes related tasks for setting up nodes.
Kubernetes 1.31 requires that you use a runtime that conforms with the Container Runtime Interface (CRI).

# For this scenario we will be configuring CRI-O : 

# Network configuration
By default, the Linux kernel does not allow IPv4 packets to be routed between interfaces. Most Kubernetes cluster networking implementations will change this setting (if needed), but some might expect the administrator to do it for them. (Some might also expect other sysctl parameters to be set, kernel modules to be loaded, etc; consult the documentation for your specific network implementation.)
Enable IPv4 packet forwarding
To manually enable IPv4 packet forwarding:
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
# Apply sysctl params without reboot
sudo sysctl --system
Verify that net.ipv4.ip_forward is set to 1 with:

sysctl net.ipv4.ip_forward

https://github.com/cri-o/packaging/blob/main/README.md#usage

# Define the Kubernetes version and used CRI-O stream
KUBERNETES_VERSION=v1.34
CRIO_VERSION=v1.34

# We will be installing v1.31 as part of this demo
Create a script called packages.sh to install cri-o, and kubelet from https://github.com/olisabosun/kubernetes/edit/master/scripts/packages-kubernetes-1.32

Paste contents and run script 
Vim packages.sh
Sudo sh packages.sh
Paste below:

#!/bin/bash

KUBERNETES_VERSION=v1.32
CRIO_VERSION=v1.32

echo ""
echo  "\033[4mDisabling Swap Memory.\033[0m"
echo ""
sudo swapoff -a
sed -e '/swap/s/^/#/g' -i /etc/fstab
echo
echo
echo "##################################################"
echo "############# INSTALLING PACKAGES ################"
echo "##################################################"
echo
echo
apt-get update -y
apt-get install -y software-properties-common curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
apt-get update -y
apt-get install -y cri-o kubelet kubeadm kubectl cri-tools
apt-mark hold kubelet kubeadm kubectl
systemctl start crio.service
modprobe br_netfilter
sysctl -w net.ipv4.ip_forward=1
echo
echo
echo "##################################################"
echo "############# PACKAGES INSTALLED  ################"
echo "##################################################"
echo
echo
echo ""
echo "===="
echo "Generate token on manager using"
echo "==="
echo "kubeadm token create --print-join-command"
echo ""
echo ""
echo "### next steps ONLY on manager ###"
echo ""
echo "sudo kubeadm config images pull"
echo "sudo kubeadm init --apiserver-advertise-address 10.0.0.100 --pod-network-cidr 172.17.0.0/16"
echo "## calico install ONLY on manager ##"
echo "kubectl apply -f https://docs.tigera.io/calico/latest/manifests/calico.yaml"


sudo crictl images

sudo kubeadm config images pull

sudo crictl images

To configure the API server advertise address for control plane nodes created with both init and join, the flag --apiserver-advertise-address can be used. Preferably, this option can be set in the kubeadm API as InitConfiguration.localAPIEndpoint and JoinConfiguration.controlPlane.localAPIEndpoint.

Note : --pod-network-cidr is a network add on that can be added but is not mandatory same as --apiserver-advertise-address

# Initialize control plane node 
sudo kubeadm init --apiserver-advertise-address 192.168.49.133 

# Initialize control plane with pod cidr network add on for IP range:
sudo kubeadm init --apiserver-advertise-address 192.168.49.133 --pod-network-cidr 172.17.00/16

sudo kubeadm init --apiserver-advertise-address 10.0.0.100 --pod-network-cidr 172.17.0.0/16


If successful you will get this output:

# To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf



Look out for this on output:

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/


Container runtime network, Underlay and Overlay(pod) network run in Kubernetes cluster.

For this demo we will be using Calico pod network.
https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises 

Go to https://github.com/networknuts/kubernetes/blob/master/scripts/calico.yaml and copy contents to Master vm; create a file called calico.yaml 

kubectl apply -f calico.yaml on manager node

kubectl get nodes


kubectl -n kube-system get pods




Calico is installed and control plane is ready.

This concluded manager configuration.

# Worker setup : login to worker node * no need install calico on workers

On master node run this command to copy package script to worker :
scp packages.sh worker01@192.168.49.134:

Go to worker node, run sudo sh packages.sh to run script

Go to master node and run this command to generate token:

kubeadm token create --print-join-command

Copy token to worker nodes and add sudo and run the command

E.g - 

# If encounter conntrack errors, run this commands for kubernetes depend install :

sudo apt update
sudo apt install -y \
    conntrack \
    socat \
    ebtables \
    ethtool \
    ipvsadm \
    iptables

# Go to worker nodes and run this command:
kubeadm join 192.168.49.133:6443 --token 5d32i5.9k4o0casa0ownrj8 --discovery-token-ca-cert-hash sha256:86c9722e1e60ccb428e65cc95f9c076042959cb07ee06a14dce4cc364d74f639

# Validate on master nodes the workers joined the cluster:

kubectl -n kube-system get pods
Kubectl get nodes


