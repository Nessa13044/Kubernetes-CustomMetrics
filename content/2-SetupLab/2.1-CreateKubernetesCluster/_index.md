---
title : "Create Kubernetes Cluster"
date : "`r Sys.Date()`"
weight : 2
chapter : false
pre : " <b> 2.1 </b> "
---


#### Prerequisites: 
 - Minimum 2GB RAM or more
 - Minimum 2 CPU cores / or 2 vCPU
 - 20 GB free disk space or more

{{% notice note %}}
Connect to all EC2s and follow the steps below which detail the steps to setup a 2 node cluster. 
{{% /notice %}}

#### 1. Prepare EC2 Instances.
- To optimize costs for this lab, I will use two **EC2** instances type **t2.medium**.
- Each instances have **2 vCPU** and **4 GB Memory**
![ec2](/images/2.prerequisite/ec2.png)

- Check two EC2 are running.
![ec2](/images/2.prerequisite/check.png)

- Now connect to the instance and follow the steps below.

#### 2. Set hostname on Each Node
- Login to to master node and set hostname via hostnamectl command. 
```
$ sudo hostnamectl set-hostname K8s_master
$ exec bash
```
{{% notice note %}}
Do the same on the client nodes with name "K8s_client".
{{% /notice %}}

- Add the following lines in /etc/hosts file on each node
```
ip k8s-master
ip k8s-client
```

#### 3. Disable Swap & Add kernel Parameters
- Execute swapoff and sed command to disable swap. Make sure to run the following commands on all the nodes.
```
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
- Load the following kernel modules on all the nodes,
```
$ sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
$ sudo modprobe overlay
$ sudo modprobe br_netfilter
```
- Set the following Kernel parameters for Kubernetes, run beneath tee command
```
$ sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT
```

Reload the above changes, run
```
$ sudo sysctl --system
```
#### 4. Install Containerd Runtime
- In this guide, we are using containerd runtime for our Kubernetes cluster. So, to install containerd, first install its dependencies.
```
$ sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```
- Enable docker repository
```
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
- Now, run following apt command to install containerd
```
$ sudo apt update
$ sudo apt install -y containerd.io
```
- Configure containerd so that it starts using systemd as cgroup.
```
$ containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
$ sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
- Restart and enable containerd service
```
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd
```

#### 5. Install Kubectl, Kubeadm and Kubelet
- Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
- Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
{{% notice note %}}
If the directory `/etc/apt/keyrings` does not exist, you can create using command: `sudo mkdir -p -m 755 /etc/apt/keyrings`
{{% /notice %}}
- next
```
#This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
- Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
``` 
- (Optional) Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```
{{% notice note %}}
It will take a lot of time to run the above commands for all 3 nodes. If HA is deployed, the node will be up to 6 or more nodes and it will take 2-3 times more time @@. I am very lazy, so I have written the `Script` below, just click 1 time to complete. hehe
{{% /notice %}}

```
#!/bin/bash

read -p "Enter name: " name
#-----set hostname
hostnamectl set-hostname $name
#exec bash
 
echo "Enter your cluster: Ex: 
10.1.1.1 master
10.1.1.2 client
more
press enter + enter if done"

while true; do
    read -p "> " line
    if [ -z "$line" ]; then
        break
    fi
    echo "$line" | sudo tee -a /etc/hosts > /dev/null
done

echo "update /etc/hosts."

#------Disable Swap & Add kernel Parameters
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
tee -a /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

tee -a /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

#--Install Containerd Runtime
apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
apt install -y containerd.io
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
systemctl restart containerd

status="$(systemctl show -p SubState containerd | cut -d'=' -f2)"
if [[ "${status}" == "running" ]]; then
        echo "Service containerd is $status"
else
        echo "Service not running - try install containerd again"
fi
#-------Install Kubectl, Kubeadm and Kubelet
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet

exec bash
echo "-----setup done-----"
```

# Initialize
- Only master node

{{% notice note %}}
As i mentioned in part 1, i will use new model in which will use the power of new technology that is `ebpf`. to make it easy to understand i will initialize kubernetes cluster without using kube-proxy. use `ebpf` instead of `iptables`. see image below
{{% /notice %}}

![ebpf](/images/1.k8s/ebpf.png)

- Run this command on master node
```
kubeadm init --control-plane-endpoint=k8s-master --upload-certs --skip-phases=addon/kube-proxy 
```
- cluster initialization successful
![ebpf](/images/2.prerequisite/kubeadm.png)

- Start interacting with cluster, run following commands on the master node.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

-  On each worker node, run command **kubeadm join....** 
![ebpf](/images/2.prerequisite/worker.png)

#### 6.Set up Cilium
![ebpf](/images/2.prerequisite/k8s.png)
- So k8s cluster is almost complete. next we will setup networking for the cluster so they can communicate with ech other.
- First download and install cilium-cli
```
curl -LO https://github.com/cilium/cilium-cli/releases/download/v0.16.2/cilium-linux-amd64.tar.gz
curl -LO https://github.com/cilium/cilium-cli/releases/download/v0.16.2/cilium-linux-amd64.tar.gz.sha256sum
```
- [See other releases of cilium here](https://github.com/cilium/cilium-cli/releases/)

- Check file integrity
```
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
cilium-linux-amd64.tar.gz: OK
```
{{% notice note %}}
If the result shows "FAILED", it means that the integrity of the file is not guaranteed. You should consider using another release.
{{% /notice %}}
- Install cilium 
{{% notice note %}}
Use this command first to check available cilium versions `cilium install --list-versions`
{{% /notice %}}
```
tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin/
cilium install --version 1.15.0
```
- So we have a fully installed kubernetes cluster and ready to use.
![done](/images/2.prerequisite/done.png)

