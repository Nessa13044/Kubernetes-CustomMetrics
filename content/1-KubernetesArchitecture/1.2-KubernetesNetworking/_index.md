---
title : "Kubernetes Networking"
date :  "`r Sys.Date()`" 
weight : 2 
chapter : false
pre : " <b> 1.2 </b> "
---
A Pod is the smallest deployable unit in Kubernetes and represents a single instance of an application. Each Pod has its unique IP address and can communicate with other Pods in the same cluster without the need for network address translation (NAT). This means that each Pod can listen on the same port without conflict.Because every component in the cluster is connected to one flat network. In a flat network, all components can communicate with each other without the need for any hardware, such as routers or switches. This is achieved by the Kubernetes network model.

#### 1. Kubernetes Networking Model

![Kubernetes Networking Model](/images/1.k8s/Kubernetes-Networking-Model.png)

- The pods are able to interact with each other because of their unique IP addresses. This is the reason why they can communicate with each other over the network.

- However, Kubernetes operates across multiple nodes (machines), and pods can be deployed on any of these nodes. This means that pods might be running on different nodes and they need a method to communicate regardless of their location.

- To facilitate this communication, Kubernetes employs a networking model that ensures pods can talk to each other no matter where they're running. This involves the use of a Container Network Interface (CNI) within Kubernetes, which handles routing traffic between pods, load balancing, and ensuring seamless communication across the cluster.

- On each node, the Kubernetes network model is implemented using a combination of the container runtime and the CNI plugin. The container runtime sets up the network namespace for each container, while the CNI plugin configures networking rules and policies to enable communication between pods in the cluster.

#### 2.Container Network Interface (CNI)
- Kubernetes relies on Container Network Interface (CNI) plugins to assign IP addresses to pods and manage routes between all pods. Kubernetes doesn't ship with a default CNI plugin, however, managed Kubernetes offerings come with a pre-configured CNI.
{{% notice note %}}
In this lab I will be using cilium. Cilium is based on a technology called eBPF,sounds very interesting 😆. I will explain why I chose it in the next section.
{{% /notice %}}

#### 3.Communication In Kubernetes Network
#### 3.1 Container-to-container communication
- The containers within the pod (the network namespace) can communicate through localhost and share the same IP address and ports. The "network namespaces" in Linux allow us to have separate network interfaces and routing tables from the rest of the system.

![container-to-container](/images/1.k8s/container-to-container.png)

- But deploying multiple containers in the same pod is quite rare when deploying applications according to microservices architecture.

#### 3.2 Pod-to-pod communication
- The nodes in the cluster have their IP address and a CIDR range from where they can assign IP addresses to the pods. A unique CIDR range per node guarantees a unique IP address for every pod in the cluster. The pod-to-pod communication happens through these IP addresses.

- Virtual ethernet devices **(veth)** connect the two veths across network namespaces. Each pod that runs on the node has a veth pair connecting the pod namespace to the nodes' network namespace. In this case, one virtual interface connects to the network namespace in the pod **(eth0)**, and the other virtual interface connects to the root network namespace on the node.

{{% notice note %}}
The communication between pods in the same node looks like this.
{{% /notice %}}

![pod-to-pod-samenode](/images/1.k8s/pod-to-pod-samenode.png)

- The traffic goes from the **eth0 interface** in the pod to the **veth** interface on the node side, and then through the **virtual bridge** to another **virtual interface** that's connected to **eth0** of the destination pod.
- This is where the Container Network Interface (CNI) comes into play. Amongst other things, the CNI knows how to set up this cross-node communication for pods.

{{% notice note %}}
And what if the pods are on different nodes. how do they route, Below explains Pod to Pod Communication on different nodes.
{{% /notice %}}

![pod-to-pod-differentnode](/images/1.k8s/pod-to-pod-differentnode.png)

- Here is routing table on one of the nodes, we'll see that it specifies where to send the packets.
```
default via 172.18.0.1 dev eth0
10.244.0.0/24 via 172.18.0.4 dev eth0
10.244.2.0/24 via 172.18.0.3 dev eth0
10.244.1.1 dev veth2609878b scope host
10.244.1.1 dev veth23974a9d scope host
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.2

```
- The line **10.244.2.0/24 via 172.18.0.3 dev eth0** says that any packets destined to IP's in that CIDR range, which are the pods running on node2, should be sent to 172.18.0.3 via the eth0 interface.

#### 3.3 Pod-to-service communication
- Pod operations like delete or create can change the ip, it is difficult to assign design if using ip of each pod, we need a durable IP address. Kubernetes solves this problem using a concept of a Kubernetes service.

- When created, a service in Kubernetes gives us a virtual IP (cluster IP) backed by one or more pods. When we send traffic to the service IP, it gets routed to one of the backing pod IPs.

- Kube-proxy controller connects to the Kubernetes API server and watches for changes to services and pods. As pods get created and destroyed, kube-proxy updates the iptables rules to reflect the changes. Whenever a service/pod gets updated, kube-proxy updates the iptables rules so traffic sent to the service IP gets correctly routed to one of the backing pods.

![kube-proxy-apiserver](/images/1.k8s/kube-proxy-apiserver.png)

- The image above shows services with 3 backend pods, when we send packets from a pod to the service IP, they get filtered through the iptables rules, where the destination IP (service IP) gets changed to one of the backing pod IPs.

![route-services](/images/1.k8s/route-services.png)

- On the way back (from the destination pod to the source pod), the destination pod IP gets changed back to the service IP, so the source pod thinks it's receiving the response from the service IP.

#### 3.4 Ingress and egress communication
#### Egress communication (traffic existing the cluster)
- On the way out of the cluster, the iptables ensure the source IP of the pod gets modified to the internal IP address of the node (VM). Typically, when running a cloud-hosted cluster, the nodes have private IPs and run inside a virtual private cloud network (VPC).
- We need to attach an internet gateway to the VPC network to allow the nodes access to the internet. The gateway performs network address translation (NAT) and changes the internal node IP to the public node IP. NAT allows the response from the internet to be routed back to the node and eventually to the original caller. 

#### Ingress communication (traffic entering the cluster)
- We need a public IP address to get the outside traffic inside the cluster. A Kubernetes LoadBalancer service allows us to get an external IP address. Behind the scenes, a cloud provider specific controller creates an actual load balancer instance in the cloud. The LB has an external IP address to send traffic to or hook up your custom domain.
- When the traffic arrives at the LB, it gets routed to one of the nodes in the cluster. Then, the iptables rules on the chosen node kick in, do the necessary NAT, and direct the packets to one of the pods that's part of the service.
- However, we don't want to create a LoadBalancer instance of every service we want to expose. Ideally, we'd have a single external IP address and the ability to serve multiple Kubernetes services behind that.