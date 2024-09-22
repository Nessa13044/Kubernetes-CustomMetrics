---
title : "Kubernetes Components"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1.1 </b> "
---
A Kubernetes cluster consists of a set of worker machines, called nodes, that run containerized applications. Every cluster has at least one worker node. The worker node(s) host the Pods that are the components of the application workload. The control plane manages the worker nodes and the Pods in the cluster. In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability.

In this lab, I only developed a model that only includes 3 nodes, which does not meet high availability. If possible, I will deploy it in the next lab or you can read it in the article below.
  - [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

Next we will go through the components of kubernetes to understand the functions of each component that we will deploy in this lab.

![architecture](/images/1.k8s/Capture.png)

#### 1.Control Plane Components
- The control plane is responsible for making decisions related to resource allocation, job scheduling, and network traffic routing. For example, in Kubernetes, the control plane decides where to deploy containers based on available resources.
- Control plane components can be run on any machine in the cluster. However, for simplicity, setup scripts typically start all control plane components on the same machine, and do not run user containers on this machine.

**kube-apiserver**
- It provides a RESTful HTTP API that users, management tools, and other Kubernetes components (such as kube-controller-manager, kube-scheduler, and kubelet) use to communicate with the cluster. Any requests related to the cluster's state or configuration go through kube-apiserver.

**etcd**
- etcd stores all Kubernetes state data, including information about Pods, Nodes, ConfigMaps, Secrets, and many other resources. Every change in the cluster, such as creating, updating, or deleting resources, is recorded in etcd.

- etcd is designed with High Consistency and Availability in mind to ensure data consistency across the entire cluster, even if failures occur. It uses the Raft algorithm to maintain data consistency between replicas in an etcd cluster. This ensures that data is always up to date and available even if one or more nodes fail.

**kube-scheduler**
- it is responsible for scheduling Pods so that they can run on Nodes in the cluster.

- When a new Pod is created that is not assigned to any Node, the kube-scheduler decides which Node in the cluster is best suited to run that Pod. This decision is based on many factors such as available resources (CPU, RAM), the resource requirements of the Pod, and other constraints.

**kube-controller-manager**
- Controllers continuously monitor the current state of resources in the cluster and adjust them to match the desired state. For example, if the number of running Pods is lower than the desired number, kube-controller-manager will create more Pods to meet the demand.

- kube-controller-manager is responsible for troubleshooting problems in the cluster. If a Pod, Node, or other resource fails, controllers will respond by restarting that resource or creating a new one.

#### 2.Node Components

**Kubelet**
- An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod.

- The kubelet takes a set of PodSpecs that are provided through various mechanisms and ensures that the containers described in those PodSpecs are running and healthy. The kubelet doesn't manage containers which were not created by Kubernetes.

**Kube-proxy**
- kube-proxy sets up rules to forward network traffic to Pods. When a Service is created, kube-proxy creates iptables or ipvs rules on each Node to forward traffic to the Pods behind that Service. This distributes traffic evenly across Pods, providing simple load balancing.

- kube-proxy continuously monitors changes in the Kubernetes API to update its network rules accordingly. When there are changes to the Service's configuration, such as adding or removing Pods, kube-proxy adjusts the network rules to ensure that traffic is forwarded correctly.

**Container runtime**
- The container runtime is responsible for creating and launching containers from images. It provides an isolated environment for containers, ensuring that applications inside the container can run independently and do not interfere with the system or other containers. It also manages the stopping and deletion of containers when they are no longer needed or when requested by Kubernetes.
- Kubernetes interacts with the container runtime through the Container Runtime Interface (CRI). There are different types of container runtimes and in this lab I'm using Containerd.