+++
title = "Installing bare-metal kubernetes on homelab"
date = "2024-06-22"
+++

Promox + Kubernetes + Ubuntu = <3

<!--more-->

## Motiviation

This week, I started studying for the CKA exam. Therefore, I needed to install Kubernetes.
I chose to set up a kubernetes cluster on my Proxmox server. If you only need kubernetes, there are 
simpler solutions such as k3s, kind, minikube, etc.

## Initial setup

I created two Ubuntui server VMs on Promox. The architecture is very simple, one master and one node both
with 2GB of memory and 2vCPU.

## Enabling Ipv4 forwarding (run on master / node)

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

## Loading kernel modules (run on master / node)

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

## Disabling swap (run on master / node)

```
swapoff -a
```

## Installing the containerD (run on master / node)

```
apt install containerd -y
```

## Creating config default (run on master / node)

```
mkdir /etc/containerd
```

```
containerd config default > /etc/containerd/config.toml
```

```
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
```

```
systemctl restart containerd
```

## Installing CNI plugins (run on master / node)

```
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
```

```
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz
```

## Adding apt repository (run on master / node)
```
apt update
apt install -y apt-transport-https ca-certificates curl gpg
```

```
mkdir -p -m 755 /etc/apt/keyrings
```

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

## Installing kubeadm, kubectl and kubelet (run on master / node)

```
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

## Enabling kubelet service (run on master / node)

```
systemctl enable kubelet
systemctl start kubelet
```

# Creating kubernetes cluster (run only on master)

```
kubeadm init --kubernetes-version=1.29.0
```

## Intalling network plugin

I will install the WeaveNet plugin, but, there are many plugins, like:

* Calico
* Cilium
* Flannel

See the complete list:
https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy

## Installing WeveNet (run only on master)

```
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

## Installing the node

This is the last step to finish our kubernetes cluster.

## Getting the join commmand (run only on master)

```
kubeadm token create --print-join-command
```

## Creating the node (run only on node)

Run the command we got from 'kubeadm token create' that we ran on the master node

### Example

```
kubeadm join 192.168.88.230:6443 --token <your token> --discovery-token-ca-cert-hash <your hash>
```

## Final considerations

* Copy the /etc/kubernetes/admin.conf to your computer and save in ~/.kube/config, else, your need to connect
on master to execute commands.

### Getting the pods running on cluster

```
kubectl get pods -A
```

### Getting all nodes

```
kubectl get nodes
```

