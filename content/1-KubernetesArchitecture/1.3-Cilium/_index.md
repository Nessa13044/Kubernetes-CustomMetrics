---
title : "Cilium"
date :  "`r Sys.Date()`" 
weight : 3 
chapter : false
pre : " <b> 1.3 </b> "
---
#### Overview
There are dozens of CNIs available for Kubernetes but, their features, scale, and performance vary greatly. Many of them rely on a legacy technology **(iptables)** that cannot handle the scale and churn of Kubernetes environments leading to increased latency and reduced throughput.

Cilium is an open source project that provides a networking and security solution based on **eBPF (extended Berkeley Packet Filter)** technology for containers and Kubernetes environments.

**Feature that impressed me the most was the Networking, Security and Observability**
{{% notice note %}}
Below is Hubble, a network monitoring and observation tool, part of the Cilium project.
{{% /notice %}}
![Hubble](/images/1.k8s/hubble.png)

#### More on CILium Network Performance

![ebpf](/images/1.k8s/ebpf.png)

- The above image shows how traditional packet filtering works using iptables and using the new technology ebpf in kubernetes.
Read more [here](https://blog.palark.com/why-cilium-for-kubernetes-networking/)